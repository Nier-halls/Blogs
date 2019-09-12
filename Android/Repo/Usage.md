 
 
## Repo命令行

### 初始化 repo init -u
repo init -u https://github.com/LineageOS/android.git -b lineage-15.1

用于获取对应的manifest配置文件
默认获取https://github.com/LineageOS/android.git目录下的default文件作为manifest目录，参数-b则是切换到对应branch也就是
“lineage-15.1”获取manifest文件。

### 删除不用的本地分支
repo abandon 分支名 [<project>…]
 
### 切换分支
repo checkout <branchname>  [<project>…]

### 显示manifest文件内容
repo manifest

### 查看本地repo管理的所有projects
repo list

 ## Repo标签说明
```
<remote name="github"
        fetch=".."
        review="review.lineageos.org" />

<remote name="aosp"
        fetch="https://android.googlesource.com"
        review="android-review.googlesource.com"
        revision="refs/tags/android-8.1.0_r52" />

<default revision="refs/heads/lineage-15.1"
         remote="github"
         sync-c="true"
         sync-j="4" />

 <project 
    path="build/blueprint" 
    name="platform/build/blueprint" 
    groups="pdk,tradefed" 
    remote="aosp" />

```
#### remote：代表github项目，可以有多个供project选择
* name：一个别名
* fetch：github项目的地址，
 fetch=".." 代表如repo init -u https://android.googlesource.com/platform/manifest时，url="https://android.googlesource.com/platform/../"也就是"https://android.googlesource.com/"
* revision：代表git clone下来以后需要切换到的分支

#### project：代表一个git仓库repository
* remote：代表这个仓库属于哪个github项目
* name：github项目下的子项目仓库地址，完整的目录就是remote.fetch + project.name
* path：git clone下来以后放在本地的目录，不设置默认和那么一样。估计就是git clone命令后面的参数

#### default：代表project的默认配置
属性和project的一致


## todo
Repo的分支是什么概念