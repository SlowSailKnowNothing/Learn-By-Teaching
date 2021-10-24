# Learn-By-Teaching
通过互相分享维护彼此的进度，并且相互讲解知识点的方式进行学习。

#### 困境

在进行java后端学习的时候，许多面试要考的知识点（也即常说的八股文部分）并不会在自己单薄的实践项目中用到，且涉及的范围广，因此一个人在学习的时候常常失去动力；其次，准备秋招也是一个漫长的过程，常常会出现懈怠的情况，所以，创建该项目，希望和一群朋友们一起分享自己的学习心得，相互激励。这也要求，每次的“学习”一定要有具体的“产出”，这就确保了每次的学习是有效的，且“产出”的结果还可以为以后所用；而且，对于一些重点难点，也可以在未来通过视频会议的方式，“以教代学”，相互帮对方解决难以理解的八股文部分（比如你教我hashmap的源码，我教你threadlocal的源码），为了满足自己的社交需求（虚荣心），或许有更好的效果也未知。

#### 方案

该项目暂时还只有一个简单的想法，并没有具体的规划，大家可以参与进来一起优化。

暂时的想法是，每一个想参与的人，可以在仓库建立一个属于自己id的文件夹，然后在该文件夹下建立自己学习内容的分类。每周向仓库推送自己本周学习的内容。另外，在仓库的总文件夹下，大家可以维护一个大家都可以直接看见的进度表和计划表，可以共享看到大家的每周计划的完成情况，可以相互推动对方学习。

此外，当遇到难以理解内容的时候，可以通过结伴学习的方式，相互认领任务，然后可以用视频会议的方式让自己来分享自己想要学习的内容，同时对方也可以理清难点。

#### git参与步骤

###### 获取项目

首先点击本仓库的fork按钮，fork me ：）

这个时候，你就拥有一份本仓库的副本了。

假设你fork之后的仓库地址为：https://github.com/user1/repository （下面的命令请自行替换自己的地址）

那么接下来将该仓库克隆到本地。

```java
git clone https://github.com/user1/repository
```

然后你会在本地得到一个Learn-By-Teaching的文件夹，进入该文件夹。

```java
cd Learn-By-Teaching
```

然后添加该项目的远程地址。

```java
git remote add upstream https://github.com/SlowSailKnowNothing/Learn-By-Teaching.git
```

然后拉取该项目的最新内容：

```java
git pull upstream main
```

###### 建立分支

注意，我们上述的操作都是在自己代码仓库的main分支上做的，而在提交自己的代码的时候，应该切换一个分支，比如：

```
git checkout -b my_change
```

###### 添加内容

在该分支做如下事情：

新建一个属于自己的文件夹：

```java
mkdir SlowSailKnowNothing
```

这里建议就用github用户名，这样可以保证不冲突。

建立属于自己的主题目录：

```java
mkdir leetcode juc jvm sourcecode springboot mysql redis
```

在对应的文件夹下添加自己本周的学习成果。

当然，如果有自己博客的话，可以附上自己的博客地址。

###### 提交到自己的仓库

```
git add .
git commit -m “message need to be added here”
```

我们在自己的代码更新之后，可能远程仓库即该项目的主仓库已经有了新的更新，如果这个时候提交PR会导致conflict。因此，在我们提交自己更新代码前，应该切换到我们自己的主分支，然后拉取最新的远程仓库内容：

```java
git checkout main
git pull upstream main
```

切换回自己的my_change分支，然后与main分支合并：

```java
git checkout my_change
git rebase main
```

然后将自己的my_change分支更新到自己的远程仓库的my_change分支中去：

```
git push origin my_change
```

###### 提交pr

到自己的仓库，切换到自己的my_change分支，然后点击new pull request。添加自己的描述，等待合并。

在自己的pr被合并之后，可以删除自己的分支。



#### 其他说明

每一个积极参与项目的成员都可以成为该项目的contributer。如果仓库内的高质量内容足够多，或许也可以获得一些star。并且大家可以在参与的过程中学习利用github进行协作。

可能没有多少人参与，甚至可能只有一个人参与~如果只有一个人的话，那就一个人吧：）
