# git笔记

## 1、git简介及安装

Git是目前世界上最先进的**分布式**版本控制系统。

### 1.1 git的安装

* linux下：ubuntu系统直接使用命令`sudo apt-get install git`完成安装。

* Windows下：从git的[官方网站](https://git-scm.com/downloads)下载安装程序，然后按照默认选项进行安装。

安装完成后可以输入git --version检查是否安装成功。

*  **安装完成后，还需要最后一步设置，在命令行输入: **
```
$ git config --global user.name "Your Name"
$ git config --global user.email "email@example.com"
```
因为git是分布式版本控制系统，所以，每个机器都必须自报家门：你的名字和Email地址。


## 2、git的使用

### 2.1 创建版本库

step1 ：选择一个合适的地方，创建一个空目录。（在windows下，为了防止出错，尽量不要出现中文目录）

step2：通过 `git init`命令，将这个目录变成Git可以管理的仓库，
```
D:\git>git init
Initialized empty Git repository in D:/git/.git
```
### 2.2 将文件添加到git仓库
添加文件到Git仓库，分两步：
1. 使用命令`git add <file>`，注意，可反复多次使用，添加多个文件；
2. 使用命令`git commit -m <message>`，完成。

简单解释一下`git commit`命令，`-m`后面输入的是本次提交的说明，可以输入任意内容，当然最好是有意义的，这样能从历史记录里方便地找到改动记录。

例如：我创建一个lwh.txt文件，并上传到git仓库中。
```
D:\git>git add lwh.txt
D:\git>git commit -m "first commit lwh.txt"
[master (root-commit) 63b6497] first commit lwh.txt
 1 file changed, 1 insertion(+)
 create mode 100644 lwh.txt
```

`git commit`命令执行成功后会提示，`1 file changed`：1个文件被改动（新添加的lwh.txt文件）；`1 insertions`：插入了一行内容（lwh.txt有两行内容）

### 2.3 版本更新与回退
#### 2.3.1 版本更新
之前已经将lwh.txt文件加入到git仓库中，若继续修改lwh.txt。可以通过命令`git status`以及命令`git diff`来查看改动的内容。

```
D:\git>git status
On branch master
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

        modified:   lwh.txt

no changes added to commit (use "git add" and/or "git commit -a")
```
`git status`命令可以让我们时刻掌握仓库当前的状态，上面的命令输出告诉我们，lwh.txt被修改过了，但还没有准备提交的修改。但是若想要查看修改的内容，则需要使用`git diff`命令。
```
D:\git>git diff
diff --git a/lwh.txt b/lwh.txt
index b91567c..717f67c 100644
--- a/lwh.txt
+++ b/lwh.txt
@@ -1 +1,2 @@
-fine i like git
\ No newline at end of file
+fine i like git
+i modify it
\ No newline at end of file
```
此时提交修改至git仓库与提交新文件步骤一样，也是先`git add <file>`，再`git commit -m`

#### 2.3.2 版本回退

每次修改文件并提交至仓库，当把文件改乱了或者误删文件的时候，就需要版本回退，以继续开展工作。

在Git中，用`git log`命令查看历史纪录。

```
D:\git>git log
commit a177d4f597b4e45f2feb8cfcacd77bedd46e829e (HEAD -> master)
Author: lvwenhua <928399277@qq.com>
Date:   Wed Oct 23 15:43:25 2019 +0800

    learn diff and status

commit 63b6497a1f98a361bbf14cb7e9699764681529ed
Author: lvwenhua <928399277@qq.com>
Date:   Wed Oct 23 15:30:31 2019 +0800

    first commit lwh.txt
```

`git log`命令显示从最近到最远的提交日志，我们可以看到2次提交。可用参数`--pretty=oneline`使得只有一行输出。

commit后跟的字符串代表的是commit id(版本号)，回退版本需要使用。

在Git中，用`HEAD`表示当前版本，也就是最新的提交`a177d4f...`，上一个版本就是`HEAD^`，上上一个版本就是`HEAD^^`，当然往上100个版本写100个`^`比较容易数不过来，所以写成`HEAD~100`

此时若想要回到第一次提交的版本，使用命令`git reset --hard HEAD^`

```
D:\git>git reset --hard HEAD"^"
HEAD is now at 63b6497 first commit lwh.txt
```
此时再次用命令`git log`查看commit日志记录。

```
D:\git>git log
commit 63b6497a1f98a361bbf14cb7e9699764681529ed (HEAD -> master)
Author: lvwenhua <928399277@qq.com>
Date:   Wed Oct 23 15:30:31 2019 +0800

    first commit lwh.txt
```
查看文件，可以发现已经被还原了。

但是，此时又后悔了，想再次回到刚才回退前的状态，也可以。通过查看屏幕上的历史记录找到上一次commit的id，reset到该id即可返回到该版本。也可以通过`git reflog`命令查看每一次命令，找到commit id。

**总结：**

- `HEAD`指向的版本就是当前版本，因此，Git允许我们在版本的历史之间穿梭，使用命令`git reset --hard commit_id`。
- 穿梭前，用`git log`可以查看提交历史，以便确定要回退到哪个版本。
- 要重返未来，用`git reflog`查看命令历史，以便确定要回到未来的哪个版本。

### 2.4 git工作区、暂存区及版本库

* #### 工作区（Working Directory）
  工作区就是在电脑中能看到的目录，如我电脑的git文件夹就是一个工作区。

  <img src="C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1571818124909.png" alt="1571818124909" style="zoom:80%;" />

* #### 版本库（Repository）
  

工作区有一个隐藏目录.git，这个不算工作区，而是Git的版本库。版本库git commit以后的最终版本存入地方，git最重要的一个地方，因为只有版本库的修改才可以跟踪。

Git的版本库里存了很多东西，其中最重要的就是称为stage（或者叫index）的**暂存区**，还有Git为我们自动创建的第一个分支`master`，以及指向`master`的一个指针叫`HEAD`

<img src="C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1571818896637.png" alt="1571818896637" style="zoom:80%;" />

当添加文件到git仓库中时，分为`add`'和`commit`两步。

第一步是用`git add`把文件添加进去，实际上就是把文件修改添加到暂存区；

第二步是用`git commit`提交更改，实际上就是把暂存区的所有内容提交到当前分支。

创建Git版本库时，Git自动为我们创建了唯一一个`master`分支，所以，现在，`git commit`就是往`master`分支上提交更改。

为了更好的理解暂存区，做一个小实验。新建一个test.txt文件夹，内容随便写；修改lwh.txt，追加一行内容。

  此时执行`git status`命令，显示如下内容：

<img src="C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1571819939644.png" alt="1571819939644" style="zoom:80%;" />

  此时，执行命令`git add test.txt`，**将test.txt加入暂存区**，再次执行`git status`，结果如下：

<img src="C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1571819955804.png" alt="1571819955804" style="zoom:80%;" />

此时使用`git diff`命令，结果如下：

<img src="C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1571820017011.png" alt="1571820017011" style="zoom:80%;" />

使用`git diff --cached`命令，结果如下：

<img src="C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1571820055783.png" alt="1571820055783" style="zoom:80%;" />

执行`git commit -m "diff and diff --cached"`命令后，使用`git status`命令，结果如下：

<img src="C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1571820580979.png" alt="1571820580979" style="zoom:80%;" />

可以发现test.txt已经没有了，分别执行`git diff`和`git diff --cached`命令，结果如下：

<img src="C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1571820667403.png" alt="1571820667403" style="zoom:80%;" />

**结果的解释如下：**

*git diff比较的是工作目录中当前文件和暂存区域快照之间的差异， 也就是修改之后还没有暂存起来的变化内容。若要查看已暂存的将要添加到下次提交里的内容，可以用 git diff --cached 命令。*

  *请注意，git diff 本身只显示尚未暂存的改动，而不是自上次提交以来所做的所有改动。 所以有时候你一下子暂存了所有更新过的文件后，运行 git diff 后却什么也没有，就是这个原因。*

### 2.5 git文件修改、删除

#### 2.5.1 Git管理修改的特性

Git跟踪并管理的是修改，而非文件。

`git diff filename`:比较工作区和暂存区

`git diff HEAD -- filename`:比较工作区和版本库的最新版本

`git diff --cached `:显示暂存区的中的修改

**总结**：每次修改，如果不用`git add`命令提交到暂存区，那就不会加入到`commit`中

#### 2.5.2 撤销修改

场景1：当你改乱了工作区某个文件的内容，想直接丢弃工作区的修改时，用命令`git checkout -- file`。

场景2：当你不但改乱了工作区某个文件的内容，还添加到了暂存区时，想丢弃修改，分两步，第一步用命令`git reset HEAD <file>`，就回到了场景1，第二步按场景1操作。

场景3：已经提交了不合适的修改到版本库时，想要撤销本次提交，参考**版本回退**一节，不过前提是没有推送到远程库。

#### 2.5.3 删除文件

* 如果用的`rm <file>`删除文件，那就相当于只删除了工作区的文件，如果想要恢复，直接用`git checkout -- <file>`就可以 

* 如果用的是`git rm <file>`删除文件，那就相当于不仅删除了文件，而且还添加到了暂存区，如果想要恢复，需要先`git reset HEAD <file>`，然后再`git checkout -- <file> `

* 如果想彻底把版本库的删除掉，先`git rm <file>`，再`git commit <file>` 就ok了，此时如果想要恢复，需要参考版本回退一节的内容。

**注意：从来没有被添加到版本库就被删除的文件，是无法恢复的！**

### 2.6 远程仓库

Git是分布式版本控制系统，同一个Git仓库，可以分布到不同的机器上。最早，肯定只有一台机器有一个原始版本库，此后，别的机器可以“克隆”这个原始版本库，而且每台机器的版本库其实都是一样的，并没有主次之分。

一种方案是找一台电脑充当服务器的角色，每天24小时开机，其他每个人都从这个“服务器”仓库克隆一份到自己的电脑上，并且各自把各自的提交推送到服务器仓库里，也从服务器仓库中拉取别人的提交。

github这个网站就是提供Git仓库托管服务的，通过它可以免费获得Git远程仓库。

#### 2.6.1 添加远程库

可以通过github网站，免费注册一个账号，在github上创建git仓库，这样，GitHub上的仓库既可以作为备份，又可以让其他人通过该仓库来协作。

第一次连接github需要上传本地的rsa公钥。

要关联一个远程库，使用命令`git remote add origin git@server-name:path/repo-name.git`；

关联后，使用命令`git push -u origin master`第一次推送master分支的所有内容；

此后，每次本地提交后，只要有必要，就可以使用命令`git push origin master`推送最新修改；

#### 2.6.2 从远程库克隆

要克隆一个仓库，首先必须知道仓库的地址，然后使用`git clone`命令克隆。

Git支持多种传输协议，包括`https`，但通过`ssh`支持的原生`git`协议速度最快。

### 2.7 分支管理

#### 2.7.1 创建、合并分支

* **(1)创建、合并分支的原理**

在版本回退一节中，可以了解到，对于每次提交操作，git都把它们串成一条时间线，这条时间线就是一个分支。在git中，默认的分支是`master`分支。`HEAD`严格来说不是指向提交，而是指向`master`，`master`才是指向提交的，所以，`HEAD`指向的就是当前分支。

一开始的时候，`master`分支是一条线，Git用`master`指向最新的提交，再用`HEAD`指向`master`，就能确定当前分支，以及当前分支的提交点：

<img src="C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1571836979314.png" alt="1571836979314" style="zoom:80%;" />

每次提交，`master`分支都会向前移动一步，这样，随着不断提交，`master`分支的线也越来越长。

当我们创建新的分支，例如`dev`时，Git新建了一个指针叫`dev`，指向`master`相同的提交，再把`HEAD`指向`dev`，就表示当前分支在`dev`上：

<img src="C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1571837006461.png" alt="1571837006461" style="zoom:80%;" />

git创建一个分支很快，因为所作的操作仅仅是增加了dev指针，并改变HEAD，使其指向dev。

不过，从现在开始，对工作区的修改和提交就是针对`dev`分支了，比如新提交一次后，`dev`指针往前移动一步，而`master`指针不变：

<img src="C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1571837395032.png" alt="1571837395032" style="zoom:80%;" />

假如我们在`dev`上的工作完成了，就可以把`dev`合并到`master`上。

合并的最简单的思路就是将`master`直接指向`dev`，这样就实现了合并。

<img src="C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1571837464347.png" alt="1571837464347" style="zoom:80%;" />

合并完分支后，甚至可以删除`dev`分支。删除`dev`分支就是把`dev`指针给删掉，删掉后，我们就剩下了一条`master`分支：

<img src="C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1571837484325.png" alt="1571837484325" style="zoom: 80%;" />

* **(2)创建、合并分支实践**

首先，创建`dev`分支，然后切换到`dev`分支

```
D:\git>git checkout -b dev
Switched to a new branch 'dev'
```

`git checkout`命令加上`-b`参数表示创建并切换，相当于以下两条命令：

```
D:\git>git branch dev
D:\git>git checkout dev
Switched to a new branch 'dev'
```

然后，用`git branch`命令查看当前分支：

```
D:\git>git branch
* dev
  master
```

`git branch`命令会列出所有分支，当前分支前面会标一个`*`号。

然后，我们就可以在`dev`分支上正常提交，比如对`lwh.txt`做个修改，然后提交。

```
D:\git>git commit -m "test dev fenzhi"
[dev 2ae6633] test dev fenzhi
 1 file changed, 2 insertions(+), 1 deletion(-)
```

现在，`dev`分支的工作完成，切换回`master`分支：

```
D:\git>git checkout master
Switched to branch 'master'
```

切换回`master`分支后，再查看`lwh.txt`文件，刚才添加的内容不见了！因为那个提交是在`dev`分支上，而`master`分支此刻的提交点并没有变：

<img src="C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1571838155006.png" alt="1571838155006" style="zoom:80%;" />

然后，将`dev`分支合并到`master`上：

```
D:\git>git merge dev
Updating 2ae6633..f1c5c56
Fast-forward
 lwh.txt | 3 ++-
 1 file changed, 2 insertions(+), 1 deletion(-)
```

`git merge`命令用于合并指定分支到当前分支。合并后，再查看`lwh.txt`的内容，就可以看到，和`dev`分支的最新提交是完全一样的。

注意到上面的`Fast-forward`信息，Git告诉我们，这次合并是“快进模式”，也就是直接把`master`指向`dev`的当前提交，所以合并速度非常快。

合并完成后，就可以放心地删除`dev`分支了：

```
D:\git>git branch -d dev
Deleted branch dev (was f1c5c56).
```

删除后，查看`branch`，就只剩下`master`分支了：

```
D:\git>git branch
* master
```

* **(3)合并过程中的冲突**

当master分支和其他分支(如feature1分支)对同一文件作出更改并提交之后，此时分支图如下：

<img src="C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1571839458020.png" alt="1571839458020" style="zoom:80%;" />

这种情况下，Git无法执行“快速合并”，只能试图把各自的修改合并起来，但这种合并就可能会有冲突。

```
D:\git>git merge feature1
Auto-merging lwh.txt
CONFLICT (content): Merge conflict in lwh.txt
Automatic merge failed; fix conflicts and then commit the result.
```

此时，`lwh.txt`文件的内容为：

<img src="C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1571839830162.png" alt="1571839830162" style="zoom:80%;" />

Git用`<<<<<<<`，`=======`，`>>>>>>>`标记出不同分支的内容，修改文件内容后保存。然后提交`lwh.txt`修改到git仓库。

此时，`master`分支和`feature1`分支变成了下图所示：

<img src="C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1571840023688.png" alt="1571840023688" style="zoom:80%;" />

用带参数的`git log`可以看到分支的合并情况：

![1571840092775](C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1571840092775.png)

此时，删除`feature1`分支，冲突解决完成。


* **（4）强制禁用`Fast forward`合并模式**

默认情况下，合并分支时，Git会用`Fast forward`模式，但这种模式下，删除分支后，会丢掉分支信息。

如果要强制禁用`Fast forward`模式，Git就会在merge时生成一个新的commit，这样，从分支历史上就可以看出分支信息。

命令：`git merge --no-ff -m "merge with no-ff" dev`

其中`--no-ff`参数，表示禁用`Fast forward`，因为这种合并方式要创建一个新的commit，所以加上`-m`参数，把commit描述写进去。

* **(5)总结：**

  查看分支：`git branch`

  创建分支：`git branch <name>`

  切换分支：`git checkout <name>`或者`git switch <name>`

  创建+切换分支：`git checkout -b <name>`或者`git switch -c <name>`（最新版本git才支持）

  删除分支：`git branch -d <name>`

  合并某分支到当前分支：`git merge <name>`
  
  用`git log --graph`命令可以看到分支合并图。
  
  注：当Git无法自动合并分支时，就必须首先解决冲突。解决冲突后，再提交即合并完成。解决冲突就是把Git合并失败的文件手动编辑为我们希望的内容，再提交。

#### 2.7.2 分支管理策略

在实际开发中，应该按照几个基本原则进行分支管理：

首先，`master`分支应该是非常稳定的，也就是仅用来发布新版本，平时不能在上面干活；

那在哪干活呢？干活都在`dev`分支上，也就是说，`dev`分支是不稳定的，到某个时候，比如1.0版本发布时，再把`dev`分支合并到`master`上，在`master`分支发布1.0版本；

参与开发的每个人都在`dev`分支上干活，每个人都有自己的分支，时不时地往`dev`分支上合并就可以了。

所以，团队合作的分支看起来就像这样：

<img src="C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1571842012196.png" alt="1571842012196" style="zoom:80%;" />

* bug分支

  修复bug时，通过创建新的bug分支进行修复，然后合并，最后删除；

  当手头工作没有完成时，先把工作现场`git stash`一下，然后去修复bug，修复后，再`git stash pop`，回到工作现场；

  在master分支上修复的bug，想要合并到当前dev分支，可以用`git cherry-pick <commit>`命令，把bug提交的修改“复制”到当前分支，避免重复劳动。

* feature分支

  软件开发中，总有无穷无尽的新的功能要不断添加进来。添加一个新功能时，为了防止将主分支搞乱，每添加一个新功能，最好新建一个feature分支，在上面开发，完成后，合并，最后，删除该feature分支。

  如果要丢弃一个没有被合并过的分支，可以通过`git branch -D <name>`强行删除。

* 多人协作

  多人协作的工作模式通常是这样：

  1. 首先，可以试图用`git push origin <branch-name>`推送自己的修改；
  2. 如果推送失败，则因为远程分支比你的本地更新，需要先用`git pull`试图合并；
  3. 如果合并有冲突，则解决冲突，并在本地提交；
  4. 没有冲突或者解决掉冲突后，再用`git push origin <branch-name>`推送就能成功！

  如果`git pull`提示`no tracking information`，则说明本地分支和远程分支的链接关系没有创建，用命令`git branch --set-upstream-to origin/<branch-name> <branch-name> `。

  这就是多人协作的工作模式，一旦熟悉了，就非常简单。

  **分支总结**

  - 强制删除分支，使用`git branch -D <name>`;
  - 临时保存工作现场，使用`git stash`，恢复时，可以使用`git stash pop`或者先`git stash apply`然后`git stash drop`;
  - Git专门提供了一个`cherry-pick`命令，能复制一个特定的提交到当前分支;
  - 查看远程库信息，使用`git remote -v`；
  - 本地新建的分支如果不推送到远程，对其他人就是不可见的；
  - 从本地推送分支，使用`git push origin branch-name`，如果推送失败，先用`git pull`抓取远程的新提交；
  - 在本地创建和远程分支对应的分支，使用`git checkout -b branch-name origin/branch-name`，本地和远程分支的名称最好一致；
  - 建立本地分支和远程分支的关联，使用`git branch --set-upstream branch-name origin/branch-name`；
  - 从远程抓取分支，使用`git pull`，如果有冲突，要先处理冲突。

### 2.8 标签管理

发布一个版本时，我们通常先在版本库中打一个标签（tag），这样，就唯一确定了打标签时刻的版本。将来无论什么时候，取某个标签的版本，就是把那个打标签的时刻的历史版本取出来。所以，标签也是版本库的一个快照。tag就是一个让人容易记住的有意义的名字，它跟某个commit绑在一起。

#### 2.8.1 创建标签

首先，切换到需要打标签的分支上：`git checkout master`；然后，敲命令`git tag <name>`就可以打一个新标签。

默认标签是打在最新提交的commit上的，如果需要对之前的commit打标签，只需要找到commit id，然后打上就可以了。

比方说要对`add merge`这次提交打标签，它对应的commit id是`f52c633`，敲入命令：`git tag v0.9 f52c633`

小结：

- 命令`git tag <tagname>`用于新建一个标签，默认为`HEAD`，也可以指定一个commit id；
- 命令`git tag -a <tagname> -m "blablabla..."`可以指定标签信息；
- 命令`git tag`可以查看所有标签。

#### 2.8.2 操作标签

- 命令`git push origin <tagname>`可以推送一个本地标签；
- 命令`git push origin --tags`可以推送全部未推送过的本地标签；
- 命令`git tag -d <tagname>`可以删除一个本地标签；
- 命令`git push origin :refs/tags/<tagname>`可以删除一个远程标签。

