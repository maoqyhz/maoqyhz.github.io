---
title: Git push 时如何避免出现 "Merge branch 'master' of ..."
tags:
  - Git
categories:
  - 解决方案
copyright: true
date: 2018-06-18 13:27:17
---

> 在使用 Git 的进行代码版本控制的时候，往往会发现在 log 中出现 "Merge branch 'master' of ..." 这句话，如下图所示。日志中记录的一般为开发过程中对代码的改动信息，如果出现过多例如上述描述的信息会造成日志的污染。

<!-- more -->
![Alt text](1.png)

阅读了一些外文的博客，下面就来一探究竟。

## 产生原因分析

当多人合作开发一个项目时，本地仓库落后于远程仓库是一个非常正常的事情，可参考下图。

```
 A-B-C(master)
    \
     D(origin/master)
```
具体情境如下：
1. 我当前拉取的远端版本为 `B`，此时修改了代码，并在本地仓库 commit 一次，但并未 push 到远端仓库。
2. 另一位开发者在 `B` 的基础上，同样 commit 了一次并 push 到远端仓库。那么这个时候，我再 push 自己的代码就会发生错误，如下。

```
To github.com:maoqyhz/usegit.git
! [rejected]        master -> master (fetch first)
error: failed to push some refs to 'git@github.com:maoqyhz/usegit.git'
hint: Updates were rejected because the remote contains work that you do
hint: not have locally. This is usually caused by another repository pushing
hint: to the same ref. You may want to first integrate the remote changes
hint: (e.g., 'git pull ...') before pushing again.
```

这个时候我们会选择，先 pull，再 push。Ok，push 成功，但是此时我们查看 log 就会发现除了我们自己提交的那条日志之外，会多出一条 "Merge branch 'master' of ..."。

那么，为什么会出现这种现象呢？其实是与 Git 的工作原理有关，对 Git 比较了解的人应该会知道，无论是 `pull`、`push` 亦或是 `merge` 操作，其实背后都是有很多的不同的模式的。

在进行 pull 操作的同时，其实就是 fetch+merge 的一个过程。我们从 remote 分支中拉取新的更新，然后再合并到本地分支中去。
1. 如果 remote 分支超前于本地分支，并且本地分支没有任何 commit 的，直接从 remote 进行 pull 操作，默认会采用 `fast-forward` 模式，这种模式下，并不会产生合并节点，也就是说不会产生多余的那条 log 信息
2. 如果想之前那样，本地先 commit 后再去 pull，那么此时，remote 分支和本地会分支会出现分叉，这个时候使用 pull 操作拉取更新时，就会进行分支合并，产生合并节点和 log 信息。这两种状态分别如下图所示：
```
# fast-forword 
A-B-D(origin/master)
     \
      C'(master)

# merge
A-B-C-E(master)
   \ /
    D(origin/master)
```

## 如何避免
为了去除自动生成的 log 信息，有以下几种解决方案：
1. 如果你使用的是 Git Bash，直接使用 `git pull --rebase`。如果拉取不产生冲突，会直接 rebase，不会产生分支合并操作，如果有冲突则需要手动 fix 后，自行合并。
2. 如果使用的是 GUI，例如 TortoiseGit，可以先 fetch，再手动 rebase 就可以了。

## 关于 rebase 和 merge

关于什么时候使用 rebase，什么时候使用 merge，开发者总结了几条规则：

> - 从 remote 分支拉取更新到本地时，使用 rebase。
> - 当完成 bug 修复或新功能时，使用 merge 将子分支合并到主分支。
> - 没有人应该 rebase 一根共享的分支。

有关这两者具体的操作，可以参考我在文章最后列出的博客。

## References
- [git-merge完全解析](https://www.jianshu.com/p/58a166f24c81)
- [git: Why “Merge branch 'master' of … ”? when pull and push](https://stackoverflow.com/questions/15439527/git-why-merge-branch-master-of-when-pull-and-push)
- [4 Ways to Avoid Merge Commits in Git (or How to Stop Being a Git Tit)](http://kernowsoul.com/blog/2012/06/20/4-ways-to-avoid-merge-commits-in-git/)
- [Git rebase and the golden rule explained.](https://medium.freecodecamp.org/git-rebase-and-the-golden-rule-explained-70715eccc372)
- [Git - When to Merge vs. When to Rebase](https://www.derekgourlay.com/blog/git-when-to-merge-vs-when-to-rebase/)

