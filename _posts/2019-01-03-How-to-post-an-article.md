---
layout: post
title: "How to post an article to this blog?"
feature-img: "assets/img/pexels/computer.jpeg"
author: truongnh
tags: [tutorial]
---

*This document is a guide to help you through the process of posting an article to [https://blog.gdgcloudhanoi.com](https://blog.gdgcloudhanoi.com).*  



#### 1. Fork the repo to your github account

Repo: [https://github.com/gdgcloudhanoi/gdgcloudhanoi.github.io](https://github.com/gdgcloudhanoi/gdgcloudhanoi.github.io)

#### 2. Clone the above forked repo to your local directory

```sh
git clone https://github.com/$USER/gdgcloudhanoi.github.io.git
```

`$USER`: your github account

#### 3. Create an article 

Create a [markdown](https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet) file in folder [\_posts](https://github.com/gdgcloudhanoi/gdgcloudhanoi.github.io/tree/master/_posts) and name it as `yyyy-mm-dd-name-of-article.md`

```sh
cd gdgcloudhanoi.github.io/_posts
touch yyyy-mm-dd-name-of-article.md
```

#### 4. Modify the created post with declaration section

Layout: `post`

```yaml
---
layout: post
title: Hello World                                # Title of the page
author: truongnh                                  # show author avatar and name, for example: truongnh, huynq, hoanh, huynq
feature-img: "assets/img/pexels/computer.jpeg"    # Add a feature-image to the post
thumbnail: "assets/img/thumbnail/sample-th.png"   # Add a thumbnail image on blog view
tags: [tutorial, gcp]                             # Add involved tags for the article, for example: kubernetes, gcp, linux,...		
---
```

> Please refer to [the source of this article](https://raw.githubusercontent.com/gdgcloudhanoi/gdgcloudhanoi.github.io/master/_posts/2019-01-03-How-to-post-an-article.md) for more details.

#### 5. Commit, push and create your Pull Request

*Reference:*  
[1] [https://truongnh1992.github.io/articles/2018-09/how-to-create-this-site](https://truongnh1992.github.io/articles/2018-09/how-to-create-this-site) 

**Happy hacking :)**
