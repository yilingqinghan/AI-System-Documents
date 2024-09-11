# Git 

## Git基础

| 命令                                                   | 含义                                                         | 效果                                                         |
| ------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| git clone                                              | 克隆到本地                                                   |                                                              |
| git branch                                             | 显示所有本地分支                                             |                                                              |
| git branch -a                                          | 显示所有分支，包括远程分支                                   |                                                              |
| git checkout                                           | 将工作区切换到本地分支                                       |                                                              |
| git checkout -b demo                                   | 创建并切换到demo分支                                         |                                                              |
| git checkout -B demo                                   | 将demo分支指向当前并切换到demo分支                           | <img src="/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240716104909699.png" alt="image-20240716104909699" style="zoom:40%;" /> |
| git checkout e0eacdd81d5b                              | 将工作区切换到commit e0eacdd81d5b                            |                                                              |
| git checkout v4.19.180-sw1.1.0                         | 将工作区切换到tag v4.19.180-sw1.1.0对应的commit              |                                                              |
| git checkout upstream/linux-rolling-stable             | 将工作区切换到远程库upstream中的linux-rolling-stable分支     | 备注：'detached HEAD' state 不位于任何本地分支上，<br />属于游离的状态：只要checkout的不是分支，就会自动detach |
| git status                                             | 查看当前工作区的状态                                         | <img src="/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240716111229027.png" alt="image-20240716111229027" style="zoom:33%;" /> |
| git add -- <path>                                      | 将工作区的文件<path>提交到暂存区(也可以是目录)               | <img src="/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240716111319525.png" alt="image-20240716111319525" style="zoom:50%;" /> |
| git add -u                                             | 将已追踪文件的更新添加到暂存区                               | <img src="/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240716111342394.png" alt="image-20240716111342394" style="zoom:50%;" /> |
| git add -A                                             | 将工作区的所有文件添加到暂存区                               | <img src="/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240716111614722.png" alt="image-20240716111614722" style="zoom:50%;" /> |
| git diff                                               | 查看位于工作区的修改                                         | <img src="/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240716111659885.png" alt="image-20240716111659885" style="zoom:50%;" /> |
| git diff --staged                                      | 查看位于暂存区的修改                                         | <img src="/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240716111717906.png" alt="image-20240716111717906" style="zoom:50%;" /> |
| git restore                                            | 将工作区的文件恢复成原始状态                                 | <img src="/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240716111758478.png" alt="image-20240716111758478" style="zoom:50%;" /> |
| git restore --staged                                   | 将暂存区的文件恢复到工作区                                   | <img src="/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240716111806942.png" alt="image-20240716111806942" style="zoom:50%;" /> |
| git commit                                             | 提交暂存区的代码到新的commit                                 |                                                              |
| git commit -s                                          | 自动添加signed-off<br />表明你同意遵守开发者的贡献协议<br />Signed-off-by: Your Name <your.email@example.com> |                                                              |
| ==git commit --amend==                                 | ==修改当前的commit，用于修改上一个提交，不会产生新的commit== |                                                              |
| git show                                               | 显示commit的详细内容                                         |                                                              |
| git show --diff-filter=<br />                          | A：仅显示新增<br />D：仅显示删除<br />M：仅显示修改<br />R：仅显示重命名<br />a/d/m/r：不显示新增/删除/修改/重命名 | ![image-20240716204831710](/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240716204831710.png) |
| git show --name-only                                   | 仅显示文件名                                                 | ![image-20240716205242272](/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240716205242272.png) |
| git show --name-status                                 | 仅显示文件名以及操作类型                                     | ![image-20240716205233716](/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240716205233716.png) |
| git fetch                                              | 从远程仓库获取更新到本地仓库，但不会合并到当前分支           |                                                              |
| git fetch -t                                           | 仅拉取远程仓库的标签tag                                      |                                                              |
| git fetch -p                                           | 在获取远程更新时，自动删除本地已经删除的远程分支的引用       |                                                              |
| git fetch -pP                                          | 清除本地多余的标签tag                                        |                                                              |
| git rebase <branch>                                    | 将当前分支的更改重新应用到指定的 `<branch>` 上               | <img src="/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240716210237699.png" alt="image-20240716210237699" style="zoom:67%;" /> |
| git rebase -i <base>                                   | 进入交互式模式，在指定的 `<base>` 之上重新应用提交           |                                                              |
| git push                                               | 如果本地分支跟远程分支不同名不能直接push                     |                                                              |
| git push <远程仓库> <本地分支>:<远程分支>              | git push origin feature-branch:main                          |                                                              |
|                                                        | 如果处于detach状态，需要在远程分支名前加上refs/head/前缀     | ![image-20240716210602075](/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240716210602075.png) |
| git pull                                               | 等同fetch + merge                                            |                                                              |
| git log                                                | 显示log                                                      |                                                              |
| git log -p                                             | 显示log并显示详细信息                                        |                                                              |
| git log -p --diff-filter                               | --diff-filter参数与show一致                                  |                                                              |
| git log --name-only                                    |                                                              |                                                              |
| git log --name-status                                  |                                                              |                                                              |
| git log origin/linux-5.10.y-sw                         | 显示远程linux-5.10.y-sw分支的log                             |                                                              |
| git log -n5                                            | 显示5条log                                                   |                                                              |
| ==git log --oneline==                                  | ==用一行显示commit号与标题==                                 | ![image-20240716210757340](/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240716210757340.png) |
| git log v4.19.180-sw.1.0.1...v4.19.180-sw1.1.0         | 显示v4.19.180-sw.1.0.1到v4.19.180-sw1.1.0之间的log           |                                                              |
| git log --graph                                        | 显示merge的分支详情                                          | ![image-20240716210832625](/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240716210832625.png) |
| ==git stash==                                          | ==临时保存当前工作目录中的修改（即使这些修改还没有提交），并将工作目录恢复到最近一次提交的干净状态== | ![image-20240716225804434](/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240716225804434.png) |
| git stash push -m ‘message’                            | 给stash条目<br/> 添加message信息                             | ![image-20240716225830305](/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240716225830305.png) |
| git stash push -- <path>                               | 指定要存储的文件                                             |                                                              |
| ==git stash list==                                     | ==列出存储区的条目==                                         |                                                              |
| git stash show [-p]                                    | 显示stash对文件条目的修改类型<br />-p：显示详细修改内容<br />默认恢复stash@{0}，<br/> 可以手动指定 | ![image-20240716225944214](/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240716225944214.png) |
| ==git stash apply==                                    | ==恢复存储区中的修改==                                       |                                                              |
| git add<br />git stash apply<br />git restore --staged | 如果apply被拒                                                | ![image-20240716230149239](/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240716230149239.png) |
| git stash drop                                         | 删除stash条目（存储区是个栈结构，index会变<br />最早push的，index最大）drop前面的，后面的index自动-1，如果要drop最前面的3个stash，  每次都是drop stash@{0} |                                                              |
| ==git stash pop==                                      | ==效果等于apply + drop==                                     |                                                              |
| ==git reset <file/dir>==                               | ==停止跟踪某些文件【处理比如想要分别提交两次两个文件但是全部已经git add了的情况】== |                                                              |
| git status -s                                          | 简览git状态                                                  |                                                              |
| git commit -a                                          | 提交暂存的文件(等同add+commit)                               |                                                              |
| git rm <file>                                          | 从工作目录和索引中删除                                       |                                                              |
| ==git rm --cached <file>==                             | ==仅从索引中删除，但保留工作目录中的文件==                   |                                                              |
| ==git rm -r directory/==                               | ==删除目录==                                                 |                                                              |
| ==git mv a b==                                         | ==重命名操作==                                               |                                                              |
| git log –stat                                          | 提交的简略信息                                               |                                                              |
| git log  –pretty=[oneline\|short\|full\|fuller]        | 以不同格式显示提交历史                                       |                                                              |
| git log --pretty=format:"%h - %an, %ar : %s"           | 以想要的方式打印                                             | <img src="/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240718111624909.png" alt="image-20240718111624909" style="zoom:50%;" /> |
| git log --pretty=format:"%h %s" --graph                |                                                              |                                                              |
| git log --relative-date                                | 使用较短的相对时间而不是完整格式显示日期(比如“2 weeks ago”)  |                                                              |
| git log --since=2.weeks                                | 显示最近两周的所有提交                                       |                                                              |
| ==git log -S <func_name>==                             | ==只显示了包含对func操作的提交==                             | ![image-20240718122712025](/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240718122712025.png) |
| ~~git checkout — <file>~~                              | 禁止使用！！                                                 |                                                              |
| git remote -v                                          | 显示需要读写远程仓库使用的 Git 保存的简写与其对应的 URL      |                                                              |
| git fetch <branch>                                     | 可以用字符串代替URL                                          |                                                              |
| git push origin master                                 | 只有当你有所克隆服务器的写入权限，并且之前没有人推送过时，这条命令才能生效。 <br />当你和其他人在同一时 间克隆，他们先推送到上游然后你再推送到上游，你的推送就会毫无疑问地被拒绝。 <br />你必须先抓取他们的工作 并将其合并进你的工作后才能推送。 |                                                              |
| git remote show <remote>                               | 查看某远程仓库的信息                                         |                                                              |
| git remote rename <remote> <newname>                   | 修改一个远程仓库的简写名                                     |                                                              |
| git remote remove paul                                 | 移除一个远程仓库                                             |                                                              |
| git tag                                                | 列出所有已有的标签                                           |                                                              |
| git tag -l "v1.8.5*"                                   | 只列出1.8.5系列                                              |                                                              |
| git tag -a v1.4 -m "my version 1.4"                    | 创建附注标签                                                 | 根据git show v1.4查看打上的标签                              |
| git tag v1.4-lw                                        | 打轻量标签(只需要提供名字)                                   | 同上，只会显示出提交信息<img src="/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240718200309844.png" alt="image-20240718200309844" style="zoom:50%;" /> |
| git tag -a v1.2 9fceb02                                | 给已经提交过的内容打标签(需要加上校验和(部分也行))           |                                                              |
| git push origin v1.5                                   | git push 命令并不会传送标签到远程仓库服务器上。<br /> 在创建完标签后你必须显式地推送标签到 共享服务器上 |                                                              |
| git push origin --tags                                 | 一次push很多标签                                             |                                                              |
| git tag -d v1.4-lw                                     | 删除标签                                                     |                                                              |
| git push origin :refs/tags/v1.4-lw                     | 将冒号前面的空值推送到远程标签名，从而高效地删除它           |                                                              |
| git push origin --delete <tagname>                     | 删除远程标签                                                 |                                                              |
| ==$ git config --global alias.co checkout==            | ==起别名：git commit就是git co==                             |                                                              |
| ==$ git config --global alias.br branch==              | ==起别名：git branch就是git br==                             |                                                              |
| ==$ git config --global alias.ci commit==              | ==起别名：git commit就是git ci==                             |                                                              |
| ==$ git config --global alias.st status==              | ==起别名：git status就是git st==                             |                                                              |
| git config --global alias.unstage 'reset HEAD --'      | 添加取消暂存别名                                             |                                                              |
| git config --global alias.last 'log -1 HEAD'           | 查看最后一次提交                                             |                                                              |

###### 配置忽略文件.gitignore

 所有空行或者以 # 开头的行都会被 Git 忽略。

• 可以使用标准的 glob 模式【简化了的正则表达式，\*任意多字符，[abc]，?任意一个，[0-9]，\*\*匹配任意中间目录】匹配，它会递归地应用在整个工作区中。

• 匹配模式可以以(/)开头防止递归。

• 匹配模式可以以(/)结尾指定目录。

• 要忽略指定模式以外的文件或目录，可以在模式前加上叹号(!)取反。

```gitignore
# 忽略所有的 .a 文件 
*.a
# 但跟踪所有的 lib.a，即便你在前面忽略了 .a 文件 
!lib.a
# 只忽略当前目录下的 TODO 文件，而不忽略 subdir/TODO 
/TODO
# 忽略任何目录下名为 build 的文件夹 
build/
# 忽略 doc/notes.txt，但不忽略 
doc/server/arch.txt doc/*.txt
# 忽略 doc/ 目录及其所有子目录下的 .pdf 文件 
doc/**/*.pdf
```

## git分支

| git branch <name>                              | 创建新分支(在当前所在的提交对象上创建一个指针)<br />git branch 命令仅仅 创建 一个新分支，并不会自动切换到新分支中去 | <img src="/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240719222209580.png" alt="image-20240719222209580" style="zoom:33%;" /> |
| ---------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| git log --oneline --decorate                   | 查看各个分支当前所指的对象                                   | <img src="/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240719222225216.png" alt="image-20240719222225216" style="zoom:33%;" /> |
| git checkout <name>                            | 切换分支                                                     | <img src="/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240719222250763.png" alt="image-20240719222250763" style="zoom:33%;" /> |
|                                                | 当commit后：git commit -a -m 'made a change'                 | **<img src="/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240719222422014.png" alt="image-20240719222422014" style="zoom:33%;" />** |
|                                                | 然后再git checkout master                                    | <img src="/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240719222459389.png" alt="image-20240719222459389" style="zoom:33%;" /> |
|                                                | 如果此时再修改并直接提交                                     | <img src="/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240719222557804.png" alt="image-20240719222557804" style="zoom:33%;" /> |
| ==git log --oneline --decorate --graph --all== | ==输出你的提交历史、各个分支的指向以及项目的分支分叉情况==   | 【 Git 的分支实质上仅是包含所指对象校验和<br />(长度为 40 的 SHA-1 值字符串)的文件，所以<br />它的创建和销毁 都异常高效。 创建一个新分支<br />就相当于往一个文件中写入 41 个字节(40 个字<br />符和 1 个换行符)】 |
| git checkout -b <name>                         | 等同于新建分支+切换过去                                      |                                                              |
| git branch -d <name>                           | 删除分支                                                     |                                                              |
| git checkout master<br />git merge iss53       | 将iss53合并到master分支                                      | <img src="/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240720195814962.png" alt="image-20240720195814962" style="zoom:33%;" /><img src="/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240720195915026.png" alt="image-20240720195915026" style="zoom:33%;" /><img src="/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240720200008474.png" alt="image-20240720200008474" style="zoom:33%;" /> |
|                                                | 在两个不同的分支中，对同一个文件的同一个部分进行<br />了不同的修改，Git 就没法干净的合并它们，此时会停止 |                                                              |
| git mergetool                                  | 使用图形化工具解决冲突                                       |                                                              |
| git branch                                     | 显示分支列表                                                 |                                                              |
| ==git branch -v==                              | ==查看每个分支的最后一个提交==                               |                                                              |
| git branch --merged                            | 过滤列表中已经合并到当前分支的分支<br />(反之用–no-merged)   |                                                              |
| git branch -D <name>                           | 强制删除分支                                                 |                                                              |

> 因为 Git 使用简单的三方合并，所以就算在一段较长的时间内，反复把一个分支合并入另一个分支，也不是什 么难事。 也就是说，在整个项目开发周期的不同阶段，你可以同时拥有多个开放的分支;你可以定期地把某些 主题分支合并入其他分支中。
>
> 许多使用 Git 的开发者都喜欢使用这种方式来工作，比如只在 master 分支上保留完全稳定的代码——有可能仅仅是已经发布或即将发布的代码。 他们还有一些名为 develop 或者 next 的平行分支，被用来做后续开发或者 测试稳定性——==这些分支不必保持绝对稳定，但是一旦达到稳定状态，它们就可以被合并入 master 分支了==。 这 样，在确保这些已完成的主题分支(短期分支，比如之前的 iss53 分支)能够通过所有测试，并且不会引入更 多 bug 之后，就可以合并入主干分支中，等待下一次的发布。

<img src="/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240720203628625.png" alt="image-20240720203628625" style="zoom:33%;" />

<img src="/Users/yilingqinghan/Documents/BaiduPan/④笔记/④使用指南/assets/image-20240720203729406.png" alt="image-20240720203729406" style="zoom:50%;" />

- 一些大型项目还有一个 proposed(建议) 或 pu: proposed updates(建议更新)分支，它可能因包含一些不成熟的内容而不能进入 next 或者 master 分支
- 当它们具有一定程度的稳定性后，再把它们合并入具有更高级别稳定 性的分支中
