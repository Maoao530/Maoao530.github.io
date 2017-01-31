---
title: Git笔记
date: 2016-07-01 22:22:24
categories:
- 其他
tags: 
- Git
---

好记性不如烂笔头，把常用的GIT命令在这里记录一下。

<!-- more -->

# 一、Git流程图

![Git流程](/img/archives/git.png)

- workspace: 本地的工作目录。（记作A）
- index：缓存区域，临时保存本地改动。（记作B）
- local repository: 本地仓库，只想最后一次提交HEAD。（记作C）
- remote repository：远程仓库。（记作D）

# 二、新建

```
git init                 //当前目录初始化为GIT代码库
git init [project-name]  //新建一个目录，将其初始化为Git代码库
git clone [url]          //检出
git config --global user.email "you@example.com" //配置email
git config --global user.name "Name" //配置用户名
```

# 三、操作

```
git add <file> // 文件添加，A → B
git add . // 所有文件添加，A → B
git commit -m "代码提交信息" //文件提交，B → C
git commit --amend //与上次commit合并, *B → C
git push origin master //推送至master分支, C → D
git pull //更新本地仓库至最新改动， D → A
git fetch //抓取远程仓库更新， D → C
git log //查看提交记录
git status //查看修改状态
git diff//查看详细修改内容
git show//显示某次提交的内容
```

# 四、撤销操作

```
git reset <file>//某个文件索引会回滚到最后一次提交， C → B
git reset//索引会回滚到最后一次提交， C → B
git reset --hard // 索引会回滚到最后一次提交， C → B → A
git checkout // 从index复制到workspace， B → A
git checkout -- files // 文件从index复制到workspace， B → A
git checkout HEAD -- files // 文件从local repository复制到workspace， C → A
```

# 五、分支相关

```
git checkout -b branch_name //创建名叫“branch_name”的分支，并切换过去
git checkout master //切换回主分支
git branch -d branch_name // 删除名叫“branch_name”的分支
git push origin branch_name //推送分支到远端仓库
git merge branch_name // 合并分支branch_name到当前分支(如master)
git rebase //衍合，线性化的自动， D → A
```

# 六、冲突处理

```
git diff //对比workspace与index
git diff HEAD //对于workspace与最后一次commit
git diff <source_branch> <target_branch> //对比差异
git add <filename> //修改完冲突，需要add以标记合并成功
```

# 七、其他

```
git diff //对比workspace与index
git diff HEAD //对于workspace与最后一次commit
git diff <source_branch> <target_branch> //对比差异
git add <filename> //修改完冲突，需要add以标记合并成功
```

关于Git更详细可以参考：

[Git完整命令地址](https://git-scm.com/book/zh/v2)