# 1 标识

> [繁华似锦Fighting - 简书 (jianshu.com)](https://www.jianshu.com/u/fa63cd14aa2c)
>
> Git是分布式版本控制系统，所以需要填写用户名和邮箱作为一个标识。
>
> ```shell
> $ git config --global user.name "liqs"
> $ git config --global user.email "1992256738@qq.com"
> ```
>
> 注意：git config --global 参数，有了这个参数，表示你这台机器上所有的Git仓库都会使用这个配置，当然你也可以对某个仓库指定的不同的用户名和邮箱。

> 需要注意的是.git是隐藏目录，需要显示隐藏文件才能看到。
> 工作区（working directory）：简单来说保存在本地的文件或目录都属于工作区，平常修改代码都是在工作区进行的。一般本地的文件，除了隐藏的.git目录，其它都属于工作区。上图中看到的mydir、file1.txt、file2.txt都是属于工作区。
> 暂存区（Stage）：又称索引区（index），顾名思义，它是工作区与本地仓库之间的一个过渡，**它记录了工作区的代码状态（有无改动，或者作了哪些改动）。位于.git目录中。**
> 本地仓库（repository）**:记录了本地代码的状态，位于.git目录中。**
> 关于状态的理解：
> 想一下，我们平时在修改一份重要的文件时，肯定要先做一份备份，目的是防止修改出错，最终还能恢复。假如这份文件本身很大，如果如果备份就会占用额外较多的空间，这样就显得不太划算。还有另外一种方式，就是记录文件的修改状态，而不是对文件本身进行备份。文件状态类似如下描述：
> 文件第一行：将xxx换成了####
> 文件第三行和第四行之间，增加了一行，内容是……
> 说白了文件状态就是文件的修改记录，根据修改记录我们就可以知道我们对文件作了哪些修改，根据修改记录也可以恢复到文件修改之前的状态。

> git status 查看文件在工作区、暂存区、本地仓库之间的变化
> git diff 查看工作区与暂存区的差异
> git diff --cached 查看暂存区与本地仓库的区别
> git diff HEAD 查看工作区与本地仓库之间的差异

[【学了就忘】Git后悔药 — 31.reset版本回退说明 - 简书 (jianshu.com)](https://www.jianshu.com/p/859161daf265)

# 2 基础操作

## 2.1 创建版本库

> 什么是版本库？版本库又名仓库，英文名repository,你可以简单的理解一个目录，这个目录里面的所有文件都可以被Git管理起来，每个文件的修改，删除，Git都能跟踪，以便任何时刻都可以追踪历史，或者在将来某个时刻还可以将文件”还原”。
>
> 首先要明确下，所有的版本控制系统，只能跟踪文本文件的改动，比如txt文件，网页，所有程序的代码等，Git也不列外，版本控制系统可以告诉你每次的改动，但是图片，视频这些二进制文件，虽能也能由版本控制系统管理，但没法跟踪文件的变化，只能把二进制文件每次改动串起来，也就是知道图片从1kb变成2kb，但是到底改了啥，版本控制也不知道。
>
> 1. 通过命令git init把当前目录变成git可以管理的仓库，这时候目录下会多一个.git的目录，这个目录是Git来跟踪管理版本的。
>
>    ```shell
>    $ git init
>    $ git add readme.txt	# 添加到暂存区
>    # 或
>    $ git rm --cached readme.txt	# 在暂存区将某文件删除
>
>    $ git commit -m "reamme.txt提交"	# 把文件提交到仓库，-m后接提交的注释
>
>    $ git status	# 查看是否还有文件未提交
>
>    $ git diff readme.txt	# 查看工作区和暂存区的差异
>
>    $ git diff --cached readme.txt	# 查看暂存区和上一次提交的差异
>
>    # 等同于
>    $ git diff --staged readme.txt
>
>    function dup()
>    {
>    	-开头的行，是只出现在源文件中的行
>
>    	+开头的行，是只出现在目标文件中的行
>
>    	空格开头的行，是源文件和目标文件中都出现的行
>
>    	差异按照差异小结进行组织，每个差异小结的第一行都是定位语句，由@@开头，@@结尾。
>    }
>    ```
>
>    [用 diff 检查改动 | Learn Version Control with Git (git-tower.com)](https://www.git-tower.com/learn/git/ebook/cn/command-line/advanced-topics/diffs)
>
> 2. 查看历史记录
>
>    > ```shell
>    > $ git log
>    > ```
>    >
>    > ![](https://img-blog.csdnimg.cn/img_convert/c953336614e9d66002ae88feda9f148f.png)
>    >
>    > ​	git log命令显示从最近到最远的修改日志。如果嫌上面显示的信息太多的话，我们可以使用命令 git log -pretty=oneline演示如下：
>    >
>    > ```shell
>    > $ git log --pretty=oneline
>    > 
>    > $ git log --pretty=oneline --abbrev-commit	# 显示版本号的前几位
>    > 
>    > $ git log --graph	# 更直观地展示分支之间的关系
>    > 
>    > $ git log --graph --abbrev-commit
>    > ```
>    >
>    > ![](https://img-blog.csdnimg.cn/img_convert/6c9aee41b9aa8dd58b8eaf05dfa3ebd4.png)
>
> 3. 版本回退
>
>    > 如果我们想使用版本回退操作，我想把当前的版本回退到上一个版本，可以使用如下两种命令，第一种是：git reset –hard HEAD^那么如果要回退到上上个版本只需把HEAD^改成HEAD^^以此类推。如果要回退到前100个版本的话，使用上面的方法肯定不方便，我们可以使用下面的简便命令操作：git reset –hard HEAD~100即可
>    >
>    > 这时候使用git log将没有后续的修改日志了，需要通过git reflog命令获取后续的修改日志。如果想返回至回退前的版本，可以先通过git reflog命令查找到对应的版本号，然后通过git reset –hard 版本号进行返回。
>    >
>    > ```shell
>    > $ git reset --hard HEAD^	# 回退到上一个版本
>    > $ git reset --hard HEAD~5	# 回退到前5个版本
>    > 
>    > $ git log	# 回退后，修改日志将从当前版本开始显示，这意味着后续修改日志信息无法显示
>    > 
>    > $ git reflog	# 查找所有日志，查找到后续的版本的版本号
>    > 
>    > $ git reset --hard 版本号	# 返回到回退前的版本
>    > 
>    > $ git reset --hard <版本号>
>    > $ git reset --soft <版本号>	# --hard表示抛弃当前工作区，--soft表示回退到之前的版本，但保留当前工作区的修改，可以重新提交
>    > ```
>
> 4) git status详情
>
>    > * "Changes to be committed："表示已经从工作区add到暂存区的file（文件或文件夹），可以通过git restore –staged filename命令将该file从暂存区移出，只有工作区有该文件，该文件就为Untracked files。
>    >
>    > * "Changes not staged for commit："表示工作区，暂存区都存在的file（文件或文件夹），在工作区进行修改或删除，但是没有add到暂存区，可以通过git add file命令将变更（修改，删除）的file add到暂存区，此时Changes not staged for commit将没有file的记录了。可以通过**git restore file**命令取消file在工作区的变更，那么暂存区的file内容还是以前的，并且file在Changes not staged for commit状态下没有记录。
>    > * "Untracked file："表示只在工作区有的file（文件或文件夹），也就是在暂存区没有该file
>    >
>    > 

## 2.2 Git撤销修改和删除文件操作

> ```shell
> $ git checkout --readme.txt	# 把readme.txt文件在工作区做的修改全部撤销，即使工作区状态与暂存区该文件状态同步
> 
> # 注意：命令git checkout -- readme.txt 中的 -- 很重要，如果没有 -- 的话，那么命令变成创建分支了。
> 
> $ touch b.txt
> $ git add b.txt
> $ git commit -m "添加b.txt文件"
> 
> $ rm b.txt
> $ git add b.txt
> $ git commit -m "b.txt 删除"
> ```
>

# 3 远程仓库

> 本地Git仓库和github仓库之间的传输是通过SSH加密的，所以本地需要生成一对密钥，私钥和公钥，私钥用于解密，公钥用于对方加密。
>
> 然后在github上创建一个新仓库，并使之与本地仓库关联。
>
> ```shell
> git remote add <shortname> <url>
> # 添加远程仓库之后，便可以使用shortname来代替整个URL了
> ```
>
> 
>
> ```shell
> $ git remote add origin https://github.com/moshang1314/testgit.git
> 
> $ git push -u origin master	# 把本地仓库分支master内容推送到远程仓库
> ```
>
> ​	由于远程库是空的，我们第一次推送master分支时，加上了 –u参数，Git不但会把本地的master分支内容推送的远程新的master分支，**还会把本地的master分支和远程的master分支关联起来**，在以后的推送或者拉取时就可以简化命令。
>
> ​	从现在起，只要本地作了提交，就可以通过如下命令：
>
> ```shell
> git push origin master
> ```
>
> ​	**克隆远程仓库：**
>
> > 当你使用 `git clone` 指令克隆仓库时，请记住这会克隆分支和远程 `HEAD` 。默认情况下通常是 `master` 分支，并包括仓库中的所有其他分支。
>
> ```shell
> $ git clone https://github.com/moshang1314/testgit2	# 这种方式没有指定分支，当代码有多个分支时，拉取的是全部分支。
> $ git clone -b https://github.com/moshang1314/testgit2	# 指定拉master分支代码
> ```

# 4 创建与合并分支

> 每次提交，Git都把它们串成一条时间线，这条时间线就是一个分支。截止到目前，只有一条时间线，在Git里，这个分支叫主分支。HEAD严格来说不是指向提交，而是指向master，master才是指向提交的，所以，HEAD指向的就是当前分支。
>
> 创建dev分支，然后切换到dev分支上。如下操作：
>
> ```shell
> $ git checkout -b dev	# 创建并切换分支
> Switched to a new branch 'dev'
> 
> $ git branch	# 查看当前分支
> ```
>
> ​	git checkout命令加上-b表示创建并切换，相当于如下2条命令：
>
> ```shell
> $ git branch dev
> $ git checkout dev
> ```
>
> ​	git branch查看分支，会列出所有的分支，当前分支前面会添加一个星号。
>
> ​	合并分支：
>
> ```shell
> $ git checkout master	# 切换到分支master
> 
> $ git merge dev	# 合并dev分支到当前分支
> $ git branch -d dev	# 删除dev分支，其实就是删除dev指针
> ```
>
> > 使用“ git checkout 分支名” 进行分支切换需要尽量保证当前分支下工作区内容、暂存区内容、版本库中最新提交内容保持一致，**因为分支切换时原分支中如果工作区内容、暂存区内容、版本库中最新提交内容保持一致的话切换后对工作区和暂存区会进行清空，并将切换后版本库的最新文件同步到工作区和暂存区；**
> >
> >  
> >
> > 如果切换前原分支的工作区文件内容与暂存区中不同，或暂存区中文件与版本库中最新提交文件不同，而这些不同的文件名如果满足下列情况则被认为不能安全切换分支而需要使用“ git checkout -f 分支名 ”命令来强制切换：
> >
> > 1. 这些内容不一致的文件在原分支的版本库（最新）中已存在；
> >
> > 2.这些内容不一致的文件在要被切换到的分支的版本库（最新）中已存在；
> >
> > 之所以会认为这两种情况为不安全切换是因为在分支切换时要进行的两个操作：
> >
> > 1. 原分支切换时会将工作区、暂存区中与版本库中同名文件（路径+名称）进行删除；
> >
> > 2.被切换分支在切换时会将版本库中文件同步到工作区、暂存区中；
> >
> > 因此满足上面的两个不安全切换条件时不能安全的进行分支切换，如果使用“ git checkout -f 分支名 ”进行强制切换那么原分支下的工作区和暂存区中不满足安全切换的那些文件将丢失（被删除或被覆盖）。
> >
> >  
> >
> > 根据上面介绍的分支切换的详细过程我们可以知道即使分支可以安全切换也会存在一种情况，即：
> >
> > 切换前原分支的工作区文件内容与暂存区中不同且这些文件在版本库中没有同名文件并且在要切换到的分支版本库中有没有同名文件，或者暂存区中有文件而在版本库中没有对应的同名文件并且在要切换到的分支版本库中有没有同名文件，这些文件在分支切换时会被继续保留在工作区和暂存区中。
>
> **处理冲突：**
>
> > 在master分支下的readme.txt文件第五行添加内容"555555"，而在dev分支下的readme.txt第五行添加内容"666666"，然后将dev分支合并到master分支将产生冲突。具体操作如下：
> >
> > ```shell
> > $ git merge -m "merge dev" dev	# -m为提交合并的注释
> > Auto-merging readme.txt
> > CONFLICT (content):Merge conflict in readme.txt
> > Automatic merge failed; fix conflicts and then commit the result.
> > ```
> >
> > ​	对于有冲突的文件状态点的合并将停止合并，然后在文件中标明冲突，由程序员手动处理。
> >
> > ```shell
> > $ git status	# 查看合并冲突所在文件
> > On branch master
> > Your branch is ahead of 'origin/master' by 1 commit.
> >   (use "git push" to publish your local commits)
> > 
> > You have unmerged paths.
> >   (fix conflicts and run "git commit")
> >   (use "git merge --abort" to abort the merge)
> > 
> > Unmerged paths:
> >   (use "git add <file>..." to mark resolution)
> >         both modified:   readme.txt
> > 
> > no changes added to commit (use "git add" and/or "git commit -a")
> > 
> > $ cat readme.txt	# 查看具体冲突
> > 111111
> > 222222
> > 333333
> > <<<<<<< HEAD
> > 555555
> > =======
> > 444444
> > >>>>>>> dev
> > ```
> >
> > ​	fast-forward（快进合并）不会生成一次提交节点，而是直接将master分支的位置移动到新分支上，这样做的话对于master分支来说就没有生成新的版本（节点），一旦删除新的分支，则将没有新分支的任何信息了。
> >
> > ​	通常合并分支时，git一般使用”Fast forward”模式，在这种模式下，删除分支后，会丢掉分支信息，现在我们来使用带参数 –no-ff来禁用”Fast forward”模式。
> >
> > ```shell
> > $ git merge --no-ff name -m "merge name"
> > ```
> >
> > 
> >
> > ​	分支策略：首先master主分支应该是非常稳定的，也就是用来发布新版本，一般情况下不允许在上面干活，干活一般情况下在新建的dev分支上干活，干完后，比如上要发布，或者说dev分支代码稳定后可以合并到主分支master上来。
>
> ![](https://backlog.com/git-tutorial/cn/img/post/stepup/capture_stepup1_4_2.png)
>
> ![](https://backlog.com/git-tutorial/cn/img/post/stepup/capture_stepup1_4_4.png)
>

# 5 bug分支

> git stash命令会把当前所有未提交的修改（包括暂存区和工作区）都保存起来，用于恢复当前工作目录。此时，工作区、暂存区和仓库便达成了同步，可以进行安全的分支切换等工作。这样便可以避免很多不必要的提交记录。
>
> 可以通过git stash list命令来查看保存的工作现场。
>
> 可以使用以下两种方法进行恢复现场：
>
> 1) git stash apply恢复，恢复后，stash内容并不删除，你需要使用命令git stash drop来删除。默认是缓存堆栈中的第一个stash（即stash@{0}），也可以通过名字指定使用或删除哪个stash。如：git stash drop stash@{0}。或可使用git stash clear命令，删除所有缓存的stash。
> 2) 使用git stash pop，在恢复的同时把stash内容也删除了。
>
> **查看指定stash的diff：**
>
> ```shell
> $ git stash show
>  index.html | 1 +
>  style.css | 3 +++
>  2 files changed, 4 insertions(+)
> 
> # 在该命令后面添加-p或--patch可以查看特定stash的全部diff，如下：
> $ git stash show -p
> diff --git a/style.css b/style.css
> new file mode 100644
> index 0000000..d92368b
> --- /dev/null
> +++ b/style.css
> @@ -0,0 +1,3 @@
> +* {
> +  text-decoration: blink;
> +}
> diff --git a/index.html b/index.html
> index 9daeafb..ebdcbd2 100644
> --- a/index.html
> +++ b/index.html
> @@ -1 +1,2 @@
> +<link rel="stylesheet" href="style.css"/>
> ```
>
> **从stash创建分支：**
>
> > 如果你储藏了一些工作，暂时不去理会，然后继续在你储藏工作的分支上工作，你在重新应用工作时可能会碰到一些问题。如果尝试应用的变更是针对一个你那之后修改过的文件，你会碰到一个归并冲突并且必须去化解它。如果你想用更方便的方法来重新检验你储藏的变更，你可以运行 git stash branch，这会创建一个新的分支，检出你储藏工作时的所处的提交，重新应用你的工作，如果成功，将会丢弃储藏。
> >
> > ```shell
> > $ git stash branch testchanges
> > Switched to a new branch "testchanges"
> > # On branch testchanges
> > # Changes to be committed:
> > #   (use "git reset HEAD <file>..." to unstage)
> > #
> > #      modified:   index.html
> > #
> > # Changes not staged for commit:
> > #   (use "git add <file>..." to update what will be committed)
> > #
> > #      modified:   lib/simplegit.rb
> > #
> > Dropped refs/stash@{0} (f0dfc4d5dc332d1cee34a634182e168c4efc3359)
> > ```
>
> **暂存未跟踪或忽略的文件：**
>
> > 默认情况下，git stash会缓存下列文件：
> >
> > * 添加到暂存区的修改（staged changes）
> > * Git跟踪的但未添加到暂存区的修改（unstaged changes）
> >
> > 但不会缓存以下文件：
> >
> > * 在工作目录中的新的文件（unstracked files）
> > * 被忽略的文件（ignored file）
> >
> > git stash目录提供了参数用于缓存上面两种类型的文件。使用-u或者–include-unstracked可以stash untracked文件。使用-a或者-all命令可以stash当前目录下的所有修改。
>
> [git-stash用法小结 - Tocy - 博客园 (cnblogs.com)](https://www.cnblogs.com/tocy/p/git-stash-reference.html)

# 6 多人协作

> 当你从远程库克隆时候，实际上Git自动把本地的master分支和远程的master分支对应起来了，并且远程库的默认名称是origin。
>
> 要查看远程库的信息 使用 git remote
> 要查看远程库的详细信息 使用 git remote –v
>
> **推送分支：**
>
> > 1. 推送分支就是把该分支上所有本地提交到远程库中，推送时，要指定本地分支，这样，Git就会把该分支推送到远程库对应的远程分支上。
> > 2. 使用命令 git push origin master
> >
> > 那么一般情况下，那些分支要推送呢？
> >
> > master分支是主分支，因此要时刻与远程同步。
> > 一些修复bug分支不需要推送到远程去，可以先合并到主分支上，然后把主分支master推送到远程去。
>
> **抓取分支：**
>
> > 因此：多人协作工作模式一般是这样的：
> >
> > 首先，可以试图用git push origin branch-name推送自己的修改.
> > 如果推送失败，则因为远程分支比你的本地更新早，需要先用git pull试图合并。
> > 如果合并有冲突，则需要解决冲突，并在本地提交。再用git push origin branch-name推送。

# 7 远程仓库的使用

## 7.1 查看远程仓库

> 如果想查看你已经配置的远程仓库服务器，可以运行 `git remote` 命令。 它会列出你指定的每一个远程服务器的简写。 如果你已经克隆了自己的仓库，那么至少应该能看到 origin ——这是 Git 给你克隆的仓库服务器的默认名字：
>
> ```shell
> $ git clone https://github.com/schacon/ticgit
> Cloning into 'ticgit'...
> remote: Reusing existing pack: 1857, done.
> remote: Total 1857 (delta 0), reused 0 (delta 0)
> Receiving objects: 100% (1857/1857), 374.35 KiB | 268.00 KiB/s, done.
> Resolving deltas: 100% (772/772), done.
> Checking connectivity... done.
> $ cd ticgit
> $ git remote
> origin
> ```
>
> 你也可以指定选项 `-v`，会显示需要读写远程仓库使用的 Git 保存的简写与其对应的 URL。
>
> ```shell
> $ git remote -v
> origin	https://github.com/schacon/ticgit (fetch)
> origin	https://github.com/schacon/ticgit (push)
> ```
>
> 如果你的远程仓库不止一个，该命令会将它们全部列出。 例如，与几个协作者合作的，拥有多个远程仓库的仓库看起来像下面这样：
>
> ```shell
> $ cd grit
> $ git remote -v
> bakkdoor  https://github.com/bakkdoor/grit (fetch)
> bakkdoor  https://github.com/bakkdoor/grit (push)
> cho45     https://github.com/cho45/grit (fetch)
> cho45     https://github.com/cho45/grit (push)
> defunkt   https://github.com/defunkt/grit (fetch)
> defunkt   https://github.com/defunkt/grit (push)
> koke      git://github.com/koke/grit.git (fetch)
> koke      git://github.com/koke/grit.git (push)
> origin    git@github.com:mojombo/grit.git (fetch)
> origin    git@github.com:mojombo/grit.git (push)
> ```

## 7.2 添加远程仓库

> 运行`git remote add <shortname> <url>添加一个新的远程Git仓库，同时指定一个方便使用的简称`：
>
> ```shell
> $ git remote
> origin
> $ git remote add pb https://github.com/paulboone/ticgit
> $ git remote -v
> origin	https://github.com/schacon/ticgit (fetch)
> origin	https://github.com/schacon/ticgit (push)
> pb	https://github.com/paulboone/ticgit (fetch)
> pb	https://github.com/paulboone/ticgit (push)
> 
> $ git remote remove pb	# 移除远程仓库
> ```
>
> 现在你可以在命令行中使用字符串 `pb` 来代替整个 URL。 例如，如果你想拉取 Paul 的仓库中有但你没有的信息，可以运行 `git fetch pb`：
>
> ```shell
> $ git fetch pb
> remote: Counting objects: 43, done.
> remote: Compressing objects: 100% (36/36), done.
> remote: Total 43 (delta 10), reused 31 (delta 5)
> Unpacking objects: 100% (43/43), done.
> From https://github.com/paulboone/ticgit
>  * [new branch]      master     -> pb/master
>  * [new branch]      ticgit     -> pb/ticgit
> ```

## 7.3 从远程仓库中抓取和拉取

> ```shell
> git fetch <远程主机名> <远程分支名>:<本地分支
> ```
>
> 这个命令会访问远程仓库，从中拉取所有你还没有的数据。 执行完成后，你将会拥有那个远程仓库中所有分支的引用，可以随时合并或查看。
>
> ```shell
> $ git fetch origin master # 从远程主机的master分支拉取最新内容
> $ git merge FETCH_HEAD	# 将拉取下来的最新内容合并到当前所在的分支中
> ```
>
> 如果你使用 `clone` 命令克隆了一个仓库，命令会自动将其添加为远程仓库并默认以 “origin” 为简写。 所以，`git fetch origin` 会抓取克隆（或上一次抓取）后新推送的所有工作。 必须注意 `git fetch` 命令只会将数据下载到你的本地仓库——它并不会自动合并或修改你当前的工作。 当准备好时你必须手动将其合并入你的工作。
>
> 如果你的当前分支设置了跟踪远程分支（阅读下一节和 [Git 分支](https://git-scm.com/book/zh/v2/ch00/ch03-git-branching) 了解更多信息）， 那么可以用 `git pull` 命令来自动抓取后合并该远程分支到当前分支。 这或许是个更加简单舒服的工作流程。默认情况下，`git clone` 命令会自动设置本地 master 分支跟踪克隆的远程仓库的 `master` 分支（或其它名字的默认分支）。 运行 `git pull` 通常会从最初克隆的服务器上抓取数据并自动尝试合并到当前所在的分支。
>
> 即将远程主机的某个分支的更新取回，并与本地指定的分支合并，完整格式可表示为：
>
> ```shell
> git pull <远程主机名> <远程分支名>:<本地主机名>
> ```
>
> > xxx:指文件名
> >
> > git update-index --assume-unchanged xxx //pull时候忽略xxx这个文件
> >
> > git update-index --no-assume-unchanged xxx //pull时候取消忽略xxx这个文件
> >
> > //忽略Output 这个文件夹，注意后面带着 / 斜杠
> >
> > git update-index --assume-unchanged Output/

## 7.4 推送到远程仓库

> 当你想分享你的项目时，必须将其推送到上游。 这个命令很简单：`git push <remote> <branch>`。 当你想要将 `master` 分支推送到 `origin` 服务器时（再次说明，克隆时通常会自动帮你设置好那两个名字）， 那么运行这个命令就可以将你所做的备份到服务器：
>
> ```shell
> git push <远程主机名> <本地分支名>:<远程分支名>
> # 如果本地分支名与远程分支名相同，则可以省略冒号：
> git push <远程主机名> <本地分支名>
> 
> git push origin master	# 将本地的master分支推送到origin主机的master分支
> 
> # 如果本地版本与远程版本有差异，但又要强制推送可以使用 --force 参数：
> git push --force origin master
> 
> # 删除主机的分支可以使用 --delete 参数，以下命令表示删除 origin 主机的 master 分支：
> git push origin --delete master
> ```
>
> 只有当你有所克隆服务器的写入权限，并且之前没有人推送过时，这条命令才能生效。 当你和其他人在同一时间克隆，他们先推送到上游然后你再推送到上游，你的推送就会毫无疑问地被拒绝。 你必须先抓取他们的工作并将其合并进你的工作后才能推送。

## 7.5 查看某个远程仓库

> 如果想要查看某一个远程仓库的更多信息，可以使用 `git remote show <remote>` 命令。 如果想以一个特定的缩写名运行这个命令，例如 `origin`，会得到像下面类似的信息：
>
> ```shell
> $ git remote show origin
> * remote origin
>   Fetch URL: https://github.com/schacon/ticgit
>   Push  URL: https://github.com/schacon/ticgit
>   HEAD branch: master
>   Remote branches:
>     master                               tracked
>     dev-branch                           tracked
>   Local branch configured for 'git pull':
>     master merges with remote master
>   Local ref configured for 'git push':
>     master pushes to master (up to date)
> ```
>
> 这个命令列出了当你在特定的分支上执行 `git push` 会自动地推送到哪一个远程分支。 它也同样地列出了哪些远程分支不在你的本地，哪些远程分支已经从服务器上移除了， 还有当你执行 `git pull` 时哪些本地分支可以与它跟踪的远程分支自动合并。

## 7.6 远程仓库的重命名与移除

> 你可以运行 `git remote rename` 来修改一个远程仓库的简写名。 例如，想要将 `pb` 重命名为 `paul`，可以用 `git remote rename` 这样做：
>
> ```shell
> $ git remote rename pb paul
> $ git remote
> origin
> paul
> ```
>
> 值得注意的是这同样也会修改你所有远程跟踪的分支名字。 那些过去引用 `pb/master` 的现在会引用 `paul/master`。
>
> 如果因为一些原因想要移除一个远程仓库——你已经从服务器上搬走了或不再想使用某一个特定的镜像了， 又或者某一个贡献者不再贡献了——可以使用 `git remote remove` 或 `git remote rm` ：
>
> ```shell
> $ git remote remove paul
> $ git remote
> origin
> ```
>
> 一旦你使用这种方式删除了一个远程仓库，那么所有和这个远程仓库相关的远程跟踪分支以及配置信息也会一起被删除。

[Git - 远程仓库的使用 (git-scm.com)](https://git-scm.com/book/zh/v2/Git-基础-远程仓库的使用)

## 7.6 远程分支与本地分支关联

> 如果想直接使用git pull或git push拉取提交数据就必须先关联本地分支与远程分支，在执行git push <远程主机名> <本地分支名>:<远程分支名>拉取时，添加**-u选项**则会自动在这两分支间建立关联。
>
> ```shell
> $ git branch -set-upstream dev origin/dev	# 把本地分支dev和远程dev分支相关联
> 
> # 查看本地分支与远程分支的映射关系
> $ git branch -vv
> 
> # 查看全部分支，包括远程分支
> $ git branch -a
> 
> # 如果本地有分支，远程没有，建立映射
> $ git push --set-upstream origin dev:dev
> 
> 
> # 如果远程分支已存在,如果本地分支名与远程分支名相同则可以省略本地分支名
> $ git branch --set-upstream-to=origin/remote_branch local_branch 
> # 等同于
> $ git branch -u origin/remote_branch local_branch
> 
> # 撤销本地分支与远程分支的关系
> $ git branch --unset-upstream <本地分支名>
> 
> # 拉取远程分支
> $ git fetch origin（不指定分支名就是远程全部分支)
> $ git fetch origin XXX(拉下远程指定的分支)
> 
> # 删除远程仓库的function分支.git push origin :XXX(用一个空的本地分支更新远程分支)
> $ git push origin :XXX
> # 或
> $ git push origin --delete XXX
> ```
>
> [(204条消息) git 关联远程分支_John_desheng的博客-CSDN博客_git 关联远程分支](https://blog.csdn.net/John_desheng/article/details/112663991)

# 8 github

> Repository：简称Repo，可以理解为“仓库”，我们的项目就存放在仓库之中。也就是说，如果我们想要建立项目，就得先建立仓库；有多个项目，就建立多个仓库。
>
> Issues：可以理解为“问题”，举一个简单的例子，如果我们开源一个项目，如果别人看了我们的项目，并且发现了bug，或者感觉那个地方有待改进，他就可以给我们提出Issue，等我们把Issues解决之后，就可以把这些Issues关闭；反之，我们也可以给他人提出Issue。
>
> Star：可以理解为“点赞”，当我们感觉某一个项目做的比较好之后，就可以为这个项目点赞，而且我们点赞过的项目，都会保存到我们的Star之中，方便我们随时查看。在 GitHub 之中，如果一个项目的点星数能够超百，那么说明这个项目已经很不错了。
>
> Fork：可以理解为“拉分支”，如果我们对某一个项目比较感兴趣，并且想在此基础之上开发新的功能，这时我们就可以Fork这个项目，这表示复制一个完成相同的项目到我们的 GitHub 账号之中，而且独立于原项目。之后，我们就可以在自己复制的项目中进行开发了。
>
> Pull Request：可以理解为“提交请求”，此功能是建立在Fork之上的，如果我们Fork了一个项目，对其进行了修改，而且感觉修改的还不错，我们就可以对原项目的拥有者提出一个Pull请求，等其对我们的请求审核，并且通过审核之后，就可以把我们修改过的内容合并到原项目之中，这时我们就成了该项目的贡献者。
>
> Merge：可以理解为“合并”，如果别人Fork了我们的项目，对其进行了修改，并且提出了Pull请求，这时我们就可以对这个Pull请求进行审核。如果这个Pull请求的内容满足我们的要求，并且跟我们原有的项目没有冲突的话，就可以将其合并到我们的项目之中。当然，是否进行合并，由我们决定。
>
> Watch：可以理解为“观察”，如果我们Watch了一个项目，之后，如果这个项目有了任何更新，我们都会在第一时候收到该项目的更新通知。
>
> Gist：如果我们没有项目可以开源或者只是单纯的想分享一些代码片段的话，我们就可以选择Gist。不过说心里话，如果不翻墙的话，Gist并不好用。
>
> [(204条消息) 详述 GitHub 如何将代码从原分支合并到 fork 分支_CG国斌的博客-CSDN博客](https://guobinhit.blog.csdn.net/article/details/98039346)
