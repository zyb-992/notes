# Git

## 三个地区

- 工作区
  - 在编辑器中编辑代码的地方，即还没有执行`git add`时。
- 暂存区
  - 指执行了`git add`后，对应了修改了的代码存储的地区
- 提交区
  - 指执行了`git commit`后，对应的`commit`存储的地区

## 远程仓库

```bash
# 通过git clone执行代码仓库克隆到本地
git clone url

# 查看远程仓库
git ls-remote

# 使用git pull origin获取远端分支
git pull origin
```

​	

## 分支管理

```bash
# 新建分支br
git branch br

# 将本地分支与远端分支进行关联
git branch -u origin/remote_br br
git branch --set-upstream-to br:origin/remote_br

git branch -m 

git branch -d
```



## 文件管理

```bash
# 修改一半的文件 不想要了 
git checkout -- file
git restore .

```



## 显示差异

```bash
# 显示工作区与暂存区的差异
git diff

# 显示暂存区与提交区的差异
# 即git add后的项目代码与提交区中最后一个commit版本的项目代码对比
git diff -staged
```



## 代码重置

```bash
# git reset的三种重置方式

# 将提交区的代码还原至commit_id对应的commit
git reset --soft commit_id

# 将提交区/暂存区的代码还原至commit_id对应的commit
git reset --mixed commit_id
  
# 将提交区/暂存区/工作区的代码还原至commit_id对应的commit
git reset --hard commit_id
```



## 变基

```bash
# 交互式操作，对commit_id之前到commit执行变基
git rebase -i commit_id 

# 
git rebase master
```

