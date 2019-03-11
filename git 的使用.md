## git 的使用

### 知识点

#### Git如何知道当前在哪个分支？

​	它有一个名为```HEAD```的特殊指针，它指向当前的本地分支。



### 起步

#### 1、初始化

```shell
git init
```

#### 2、查看状态

```shell
git status
```

#### 3、文件的状态

+ 文件的状态分为两种：已跟踪和未跟踪

```shell
## 使用git status 命令，状态解释：
Untracked files：  
			# 未纳入版本控制
Changes to be committed：
			# 已暂存状态，
Changes not staged for commit：
			# 已跟踪文件发生变化，但是还未纳入暂存区

```

+ 已暂存和未暂存的修改

```shell
git diff  ## 当前文件和暂存中文件的不同
```

* 提交

```shell
git commit -m ''     # 提交暂存区的内容
git commit -a	     # 跳过使用暂存区，自动把所用已经跟踪过的文件暂存起来，一并提交
```

#### 4、远程仓库使用

```shell
## 添加远程仓库
git remote add <shortname> <url>
## 查看远程仓库
git remote 
	-v :显示git远程仓库的简写和对应url
## 从远程仓库抓取和拉取
git fetch [remote-name]
			#这个命令会访问远程仓库，从中拉取所用你还没有的数据
			#该命令会拉取数据到你本地的仓库，但是不会自动合并或修改你当前的工作，必须手动合并
## 推送到远程仓库
git push [remote-name] [branch-name]
			#当你之前没有被人推送时，才生效。否则会被拒绝
             #你必须，将其他人的工作拉取下来并将其合并进你的工作后才能推送
## 查看远程仓库
git remote show [remote-name]
## 远程仓库的重命名
git remote rename [old_shortname] [new_shortname]
## 移除远程仓库
git remote rm [shortname]
```

### 分支

#### 分支创建

```shell
git branch testing
# 仅仅创建一个新分支，并不会自动切换的新分支
```

#### 分支切换

```shell
git checkout testing
# HEAD指针就 指向testing分支了
```



