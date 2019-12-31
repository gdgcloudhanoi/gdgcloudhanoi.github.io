---
title: How to public your posts to github-page?
tags: [tutorials]
keywords: tutorials, git, github
last_updated: November 23, 2018
summary: "This document is a guide to help you through the process of contributing to the VietKubers Tech Blog"
sidebar: mydoc_sidebar
permalink: 2018-11-23-public_to_github-page.html
folder: mydoc
---

1. **Fork** the repo https://github.com/vietkubers/vietkubers.github.io to your Github account
2. **Clone** the above forked repo to your local directory
```
$ git clone https://github.com/$USER/vietkubers.github.io.git
```
3. **Create** a [markdown](https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet) file in folder [pages/mydoc](https://github.com/vietkubers/vietkubers.github.io/tree/master/pages/mydoc) and name it as `yyyy-mm-dd-name-of-article.md`
```
$ cd vietkubers.github.io.git/pages/mydoc
$ touch yyyy-mm-dd-name-of-article.md
```
4. **Modify** the created post with declaration section as:
```yaml
---
title: Deploying multiple nodes with Kubeadm
tags: [linux, tutorials, kubernetes]
keywords: linux, networking
last_updated: November 21, 2018
sidebar: mydoc_sidebar
permalink: yyyy-mm-dd-name-of-article.html
folder: mydoc
---
```
Please refer to [this example post](https://raw.githubusercontent.com/vietkubers/vietkubers.github.io/master/pages/mydoc/2018-11-19-cgroups.md) for more details.  
5. **Commit**, **push** and create your **Pull Request**  
For the step-by-step process, refer to this article [1]

**Reference**

[1] https://vietkubers.github.io/2018-16-16-github-workflow.html



*Author: [truongnh1992](https://github.com/truongnh1992)*
