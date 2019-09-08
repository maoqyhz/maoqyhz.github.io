---
title: Hexo Next 集成 utterances 评论系统
tags:
  - Hexo
  - Hexo Next 
categories:
  - 技术
copyright: true
date: 2019-08-20 22:43:16
---

可能是目前静态博客最好的评论系统。

<!--more-->

说好了再也不折腾主题和样式了，那不是可能的！之前一直在用一个国产的评论系统 valine，这是一个基于 Leancloud 的一个评论系统，展示的方式还算不错啦，但是也存在问题，一是没有评论推送，二是后续无法更好的沟通。

于是就诞生了另外一系列基于 GitHub issue 的评论系统，例如 gitment、gitalk。这类评论系统会为每一篇博客生成一个 issue 作为评论存储，自然发布评论后，GitHub会自动邮件推送，后续也可继续在上面交流，一举两得。原本也想用 gitalk 的，因为 hexo next 主题默认是集成的，但是无意中在 v2 看了一篇文章（[建议大家弃用 Gitalk 和 Gitment 等权限过高的 Github OAuth App](https://www.v2ex.com/t/535608)），立马就带打消了念头，因为不安全！

上面两种评论系统获取的 OAuth 权限过高，直接可以读写评论人所有的 repo，存在巨大的风险。而 [utterances](https://github.com/utterance) 是通过 GitHub app 实现的，仅需要获取用于评论的那个 repo 即可。

> A lightweight comments widget built on GitHub issues. Use GitHub issues for blog comments, wiki pages and more!
>
> - Open source. 🙌
> - No tracking, no ads, always free. 📡🚫
> - No lock-in. All data stored in GitHub issues. 🔓
> - Styled with Primer, the css toolkit that powers GitHub. 💅
> - Dark theme. 🌘
> - Lightweight. Vanilla TypeScript. No font downloads, JavaScript frameworks or > polyfills for evergreen browsers. 🐦🌲

## 集成流程

### GitHub 配置与脚本获取
1. 创建存放 comments 的代码仓库，必须为 public，且可创建 issue。
2. [install utterances app](https://github.com/apps/utterances) 点击这个链接安装utterances app到刚刚创建的那个仓库。
3. 打开 https://utteranc.es/ ，根据提示的步骤生成所需的js script。也可在下面代码的基础上更新自己的信息。
```javascript
<script src="https://utteranc.es/client.js"
        repo="maoqyhz/comments" # onwer/repo
        issue-term="pathname"   # 命名issue的格式，默认pathname
        label="Comment"         # 创建issue的tag，默认Comment
        theme="github-light"    # 评论系统theme，github-light或github-dark
        crossorigin="anonymous" # 跨域，默认
        async>
</script>
```
### Hexo Next 主题配置

首先，进入主题目录，在 `layout/_third-party/comments/` 中创建 `utterances.swig`，并添加以下代码：
```
{% if theme.utterances.enable %}
<script src="https://utteranc.es/client.js"
        repo="{{ theme.utterances.repo }}"
        issue-term="{{ theme.utterances.issue_term }}"
        label="Comment"
        theme="{{ theme.utterances.theme }}"
        crossorigin="anonymous"
        async>
</script>
{% endif %}
```
再在 `layout/_partials/comments.swig` 中添加以下代码：
```
{% elseif theme.utterances.enable %}
 <div class="comments" id="comments">
    {% include '../_third-party/comments/utterances.swig' %}
 </div>
{% endif %}
```
最后，在主题配置文件 `theme/_config.yml` 中添加如下配置：
```yaml
# utterances 
utterances:
  enable: true
  repo:                  # owner/repo
  issue_term: pathname   # pathname, url, title
  theme: github-light    # github-light or github-dark
```

重新生成网页，就能看到评论系统了。