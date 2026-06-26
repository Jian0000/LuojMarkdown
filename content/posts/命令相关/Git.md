+++

title = 'Git'
date = '2026-06-08T16:01:00+08:00'
draft = false

+++

## Pull

拉下来切换到远程分支

```
git clone xxx
git checkout -b localName origin/remoteName
```

基于远程分支新建开发分支

```
git branch 开发分支名
```

> ```
> git branch -d 分支名//删除分支
> ```



### 提交修改

```
git add
git commit/git cz
```

> ### 取消commit
>
> #### 留下commit的修改
>
> ```
> git reset --soft <commit_hash>^ //也就是git add 之后，改动留在暂存区
> git reset --soft HEAD~
> git reset --mixed <commit_hash>^//也就是git add之前，改动留在工作区
> git reset --mixed HEAD~
> ```
>
> #### 不保存commit的修改
>
> ```
> git reset --hard <commit_hash>^//丢去此次commit的修改
> //！！！！注意他会将工作区的内容直接恢复到没有此次commit之前，如果改动还在工作区那么就会被复位
> ```
>
> ### 抵消commit
>
> ```
> git revert//git revert本质上是产生一个新的commit来撤销某个commit的修改，故可以利用git revert来回退版本
> ```



## push

#### push前务必拉取最新的代码

```c++
git pull --no-rebase origin 远程分支名
```

#### push前将开发分支的改动cherry-pick过去

```
//1.首先切换到你想将 commit 迁移到的目标分支
git checkout <目标分支>
//2.找到要迁移的 commit
git log <开发分支>
//3.使用 cherry-pick 迁移 commit
git cherry-pick <commit哈希值>
//如果要一次性迁移多个 commit，可以使用 .. 范围：
git cherry-pick <开始哈希值>^..<结束哈希值>
```

#### 将改动push到远程

```c++
git push origin localName:remoteName
```

```
git push origin --delete xxx//删除远程分支
```

> ### rebase与no-rebase
>
> ```c++
> remote：A-B-C-D
> local ：A-B-C-E
> git pull
> rebase：//相当于把代码拉下来，重新提交一次E
> A-B-C-D-‘E’//这个‘E’内容是一样的但是和本地最开始的commit id不同 
> 
> no rebase：//会进行一次合成，在本地多出一个新的为merge commit M.
>    D
>    ↓
> A-B-C-M
>    ↑
>    E
> ```





