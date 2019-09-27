# Binder Native层通讯总结

## 提纲
> * Binder整体流程概括
> * 数据结构概括：`binder_transaction_data`、`binder_transaction`、`binder_proc`
> * Binder流程步骤说明
>   1. 组装请求用`cmd` + `binder_transaction_data`
>   2. 调用Binder驱动层通讯函数ioctl，陷入内核层
>   3. 驱动层解析`binder_transaction_data`找到BinderServer进程`binder_proc`
>   4. 组装一个任务结构体`binder_transaction`加入到BinderServer进程任务队列中
>   5. 请求方进入阻塞状态等待BinderServer进程返回结果
>   6. BinderServer进程的其中一个线程唤醒并解析`binder_transaction`任务，组装`cmd` + `binder_transaction_data`返回给BinderServer的Native层
>   7. BinderServer从驱动层返回读取`binder_transaction_data`
>   8. 获取BBinder(JavaBBinder)调用Java层`Binder.onTransact`方法
>   9. 获取返回结果并且返回给驱动层
>   10. 找到阻塞等待唤醒的线程，组装`binder_transaction`分配到线程的任务队列中
>   11. BinderServer重新进入阻塞状态等待下次请求的到来
>   12. BinderClient被唤醒后解析`binder_transaction`任务，组装`cmd` + `binder_transaction_data`返回给BinderClient的Native层
>   13. 组装reply(Parcel)数据放回给上层
> * Binder流程图
> * Binder的线程管理
>   1. 默认线程数
>   2. 线程池扩充逻辑
>   3. 线程阻塞唤醒逻辑以及优化

### Binder整体流程概括

### 数据结构概括
Binder请求过程中重要的数据结构一共有三个，分别是`binder_transaction_data`、`binder_transaction`、`binder_proc`这三个数据贯穿整个Binder Native层的通讯流程；

#### binder_transaction_data
用来Native层Binder驱动层通讯的Binder请求数据结构
```
struct binder_transaction_data {
	union {
		__u32 handle; //发起请求时代表binder_server的handle引用号
		binder_uintptr_t ptr; //接收请求时由驱动赋值代表BBinder的弱引用
	} target;

	binder_uintptr_t cookie; //BBinder的强引用	
	__u32 code; //调用远端服务的方法code

	binder_size_t data_size; //数据总长度（不包括记录binder偏移量的数组长度）
	binder_size_t offsets_size; //记录Binder类型数据的数组总长度 
	union {
		struct {
			binder_uintptr_t buffer; //Parcel数据所在的地址
			binder_uintptr_t offsets; //Parcel数据中记录Binder数据的数组地址
		} ptr;
	} data;
}
```
binder_transaction_data结构体中数据块和Parcel结构是一致的主要分两个，一块是用来记录数据的，尾部一块是一个数组，用来记录前面数据部分中的binder数据的相对地址；具体字段的含义如下图；

![DecorView](./pic/pic_binder_transaction_data.png)


#### binder_proc`
描述Native层进程的结构体，保存着对应进程持有的Binder引用和Binder节点
```
struct binder_proc {
	struct hlist_node proc_node;  //进程节点
	struct rb_root threads;  //进程包含的所有线程红黑树
	struct rb_root nodes;  //binder_node红黑树节点
	struct rb_root refs_by_desc;  //binder_ref红黑树节点handle为key
	struct rb_root refs_by_node;  //binder_ref红黑树节点ptr为key
	int pid;  //进程pic
	void *buffer;

	struct list_head todo;  //binder_transaction任务队列
	wait_queue_head_t wait;  //等待队列头，用于阻塞线程的
	int max_threads;  //binder线程池的最大线程数
	int requested_threads;  //正在发起添加线程的请求数，不是0就是1
	int requested_threads_started; //当前总共存在的binder线程数目
	int ready_threads; //控限的binder线程数
}
```
#### binder_transaction`
Binder线程（进程）的任务描述结构体，是进程与进程之间通过Binder驱动进行通讯的载体
```
struct binder_transaction {
	struct binder_work work; //列表存储的节点，用来获取binder_transaction
	struct binder_thread *from; //任务发起的线程，用于后续的唤醒等操作
	struct binder_transaction *from_parent; //记录当前任务是由哪个任务引起的，当一个任务需要多次跨进程才能完成时会出现，形成一个链表
	struct binder_proc *to_proc;  //目标进程（目标进程和线程至少有一个不为NULL）
	struct binder_thread *to_thread; //目标线程（目标进程和线程至少有一个不为NULL）
	struct binder_transaction *to_parent;  //在binder线程处理服务时赋值，记录上一次发起请求且还在等待的任务，当服务处理结束后将该字段赋值给thread继续等待用
	struct binder_buffer *buffer;//下方说明
	unsigned int	code; //对应binder_transaction_data->code
}

struct binder_buffer {
	struct binder_transaction *transaction; //所属的binder_transaction任务
	struct binder_node *target_node; //标记请求的目标binder节点
	size_t data_size; //对应binder_transaction_data->data_size
	size_t offsets_size; //对应binder_transaction_data->offsets_size
	uint8_t data[0]; //对应binder_transaction_data->data.ptr.buffer
};
```

这三个数据结构的关系如下：

![DecorView](./pic/pic_binder_native_obj21.png)


### Binder流程步骤说明
从BinderClient端的一次跨进程请求开始BpBinder调用IPCThreadState.transact方法开始，主要分为以下几步逻辑；

#### 1 组装binder_transaction_data
首先BinderClient会组装一个用于Binder请求的数据结构binder_transaction_data;
```
status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
    int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)
{
    binder_transaction_data tr;

    tr.target.ptr = 0;
    tr.target.handle = handle;
    tr.code = code;
    tr.flags = binderFlags;
    tr.cookie = 0;
    tr.sender_pid = 0;
    tr.sender_euid = 0;

    tr.data_size = data.ipcDataSize();//Parcel数据长度
    tr.data.ptr.buffer = data.ipcData();//Parcel数据的其实地址
    tr.offsets_size = data.ipcObjectsCount()*sizeof(binder_size_t);//Binder数据偏移量数组总长度
    tr.data.ptr.offsets = data.ipcObjects();//Binder数据偏移量数组地址

    mOut.writeInt32(cmd);//BC_TRANSACTION
    mOut.write(&tr, sizeof(tr));

    return NO_ERROR;
}
```
需要注意的是请求将BpBinder传递过来的handle引用号赋值给了binder_transaction_data.target.handle变量上，用于告诉驱动本次请求的BinderServer；

#### 2 调用ioctl陷入内核层
组装完成binder_transaction_data后会统一根据协议写入到binder_write_read结构体中
```
status_t IPCThreadState::talkWithDriver(bool doReceive)
{
	binder_write_read bwr;
	bwr.write_size = outAvail;//cmd + binder_transaction_data的长度
    bwr.write_buffer = (uintptr_t)mOut.data();//cmd + binder_transaction_data
	bwr.read_size = mIn.dataCapacity();//默认256
    bwr.read_buffer = (uintptr_t)mIn.data();//空

	...
	ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0
	...
}
```
这里将cmd和binder_transaction_data根据协议传给binder_write_read结构体接着调用ioctl进入内核态；

#### 3 驱动层解析binder_transaction_data，找到Server进程
```
static void binder_transaction(struct binder_proc *proc,
			       struct binder_thread *thread,
			       struct binder_transaction_data *tr, int reply)
{
	if (tr->target.handle) {
		struct binder_ref *ref;
		//尝试在自己的进程描述体中找到BinderServer对应的binder引用
		ref = binder_get_ref(proc, tr->target.handle, true);
		if (ref == NULL) {
			return_error = BR_FAILED_REPLY;
			goto err_invalid_target_handle;
		}
		//通过binder引用找到binder节点
		target_node = ref->node;
	}	
	//找到节点中记录的BinderServer所在的进程描述体
	target_proc = target_node->proc;

	...
}
```
在向目标Binder发送请求前提必然是已经获取过这个BinderServer的引用，获取过内核就会以binder_ref的形式将BinderServer记录在请求进程的描述体binder_proc中，并且会生成一个对应的handle给上层用。

当上层发起请求是引用获取的顺序handle->binder_ref->binder_node最后获取到binder_node所在的进程后续要给进程提交任务；


#### 4 组装binder_transaction任务提交到Server进程任务队列中
在找到Server进程后就准备将本次请求封装成一个binder_transaction任务提交到Server进程任务队列中，来唤醒Server进程中的Binder线程来处理
```
static void binder_transaction(struct binder_proc *proc,
			       struct binder_thread *thread,

			       struct binder_transaction_data *tr, int reply)
{
	struct binder_transaction *t;
	...

	t->from = thread; //记录当前任务来自哪个BinderClient线程
	t->to_proc = target_proc; //BinderServer所在的进程
	t->to_thread = target_thread;//在请求过程中大部分时间为null，优化复用是才会有值
	t->code = tr->code;//调用BinderServer的方法code
	
	//在目标进程对应的mmap内存映射空间申请一段存放参数的内容
	t->buffer = binder_alloc_buf(target_proc, tr->data_size,
		tr->offsets_size, !reply && (t->flags & TF_ONE_WAY));
	t->buffer->target_node = target_node;//记录BinderServer对应的节点

	//从BinderClient进程拷贝数据到BinderServer的mmap映射内存
	if (copy_from_user(t->buffer->data, (const void __user *)(uintptr_t)
			   tr->data.ptr.buffer, tr->data_size)) {
		return_error = BR_FAILED_REPLY;
		goto err_copy_data_failed;
	}
	t->from_parent = thread->transaction_stack;//优化重用等待线程优化用
	thread->transaction_stack = t;//标记当前线程正在等待的任务
	t->work.type = BINDER_WORK_TRANSACTION;

		target_list = &target_proc->todo;
		target_wait = &target_proc->wait;

	list_add_tail(&t->work.entry, target_list);//将任务提交到目标进程的任务队列中
	wake_up_interruptible(target_wait);//唤醒目标进程中的某个线程
	...
}
```

#### 5 BinderClient进入阻塞状态等待BinderServer进程返回结果
BinderClient在提交完任务后会将提交任务的结果返回给Native层而后根据是否需要返回结果来判断受否进入阻塞状态，请求的是需要返回结果（reply不会null）则会进入阻塞状态
```
static int binder_thread_read(struct binder_proc *proc,
			      struct binder_thread *thread,
			      binder_uintptr_t binder_buffer, size_t size,
			      binder_size_t *consumed, int non_block)
{
	//当前线程没任务或者对应的任务队列是NULL则会阻塞等待请求任务，而当前线程是存在一个待完成的任务，所以是false
	int wait_for_proc_work = thread->transaction_stack == NULL &&
				list_empty(&thread->todo);

	if (wait_for_proc_work) {
		...
	} else {
		//阻塞进入等待状态
		ret = wait_event_freezable(thread->wait, binder_has_thread_work(thread));
}
```

#### 6 BinderServer进程的其中一个线程唤醒并解析`binder_transaction`任务
```
static int binder_thread_read(struct binder_proc *proc,
			      struct binder_thread *thread,
			      binder_uintptr_t binder_buffer, size_t size,
			      binder_size_t *consumed, int non_block)
{
	if (wait_for_proc_work) {
		//此时其中一个以进程等待队列头阻塞的线程被唤醒
		ret = wait_event_freezable_exclusive(proc->wait, binder_has_proc_work(proc, thread));
	}

	//从进程任务队列中获取一个任务
	if (!list_empty(&proc->todo) && wait_for_proc_work)
			w = list_first_entry(&proc->todo, struct binder_work, entry);

	switch (w->type) {
		case BINDER_WORK_TRANSACTION: {
			//从entry中重新获取到加入的binder_transaction
			t = container_of(w, struct binder_transaction, work);
		} break;
	}	

	//检查是否有目标BinderServer节点，有就说明是请求时BinderServer处理的任务
	//没有就说明是BinderClient线程被唤醒时用于返回数据的任务
	if (t->buffer->target_node) {
		//开始拼装给BinderServer进程用的binder_transaction_data数据体
		struct binder_node *target_node = t->buffer->target_node;
		tr.target.ptr = target_node->ptr;//记录目标Server的弱引用
		tr.cookie =  target_node->cookie; //记录目标BinderServer的引用
		cmd = BR_TRANSACTION;
	}else {
			tr.target.ptr = 0;
			tr.cookie = 0;
			cmd = BR_REPLY;
	}
	tr.code = t->code;//赋值请求方法code

	//下面是把之前的BinderClient端binder_transaction_data传给binder_transaction又重新转成
	//BinderServer所需要的binder_transaction_data中
	tr.data_size = t->buffer->data_size;
	tr.offsets_size = t->buffer->offsets_size;
	tr.data.ptr.buffer = (binder_uintptr_t)(
				(uintptr_t)t->buffer->data +
				proc->user_buffer_offset);
	tr.data.ptr.offsets = tr.data.ptr.buffer +
				ALIGN(t->buffer->data_size,
					sizeof(void *));

	//将cmd BR_TRANSACTION和binder_transaction_data写回到BinderServer所在的用户进程
	if (put_user(cmd, (uint32_t __user *)ptr))
		return -EFAULT;
	ptr += sizeof(uint32_t);
	if (copy_to_user(ptr, &tr, sizeof(tr)))
		return -EFAULT;
}
```
这一步主要是BinderServer收到任务后重新根据binder_transaction任务中的参数拼装成一个binder_transaction_data并复制到用户进程中去，让其去Java层处理这个任务，注意这里`tr.target.ptr`就是目标BinderServer对应的BBinder对象，里面保存着java层Binder的引用，是从Java层Binder->JavaBBinder->flat_binder_object->binder_node一步一步存到内核，然后又一步步的从内核返回给上层来处理任务；