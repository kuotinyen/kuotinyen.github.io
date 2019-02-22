---
title: "用 Travis CI 輕鬆部署 Hugo Blog 的 Github Page"
author: "kuotinyen"
tags: ["Hugo", "Github", "Travis"]
date: 2019-02-22T15:10:26+08:00
---

前陣子使用 Medium 寫文章感覺很不錯，<br/>
但是不支援程式碼 highlight 跟 markdown 還是有點困擾。<br/>
看過 jekyll, Hexo 跟 Hugo 後最後決定使用 Hugo。<br/>

折騰了兩天，參考一些網路上的資源順利完成了。<br/>
現在可以輕鬆寫 markdown 文章，然後自動部署到 Github Page 上了，開心。

<!--more-->

### TL;DR

這篇文章可以幫助你

- 一步一步建立 Hugo Blog
- 只需要管理一個 Repo，而不是編輯 Blog 一個 Repo，更新 Github Page 又一個 Repo
- 編輯完 Blog 後只要 commit 就會觸發自動化流程更新 Github Page

------

### Hugo

[官方網站](https://gohugo.io) <br/>
[官方網站 - Host on Github](https://gohugo.io/hosting-and-deployment/hosting-on-github/)

後起之秀的 Hugo stars 數已經屌打 Hexo，目錄結構又單純好懂， <br/>
缺點就是沒有那麼多 plug-in 跟前端沒那麼炫，但對我而言已經很足夠啦。

**建立一個 Blog**

在想要建立 Blog 的地方下指令：

```java
hugo new site my-blog
cd my-blog
```

**下載 Nuo theme**

清爽好用的 Hugo theme，還支援 RWD。

```java
cd themes
git clone git@github.com:laozhu/hugo-nuo.git
cd ..
echo 'theme = "hugo-nuo"' >> config.toml
```

**一為全，全為一的 Repo 管理**

多數網路上的教學文都會建議產生兩個 Repo，一個是編輯 Blog 的 Repo，<br/>
另一個則是負責更新 Github Page 的 xxx.github.io Repo。<br/>

Hugo 產生靜態 Blog 的原理是：<br/>

> 對 Hugo Blog project 做任何編輯變更後，<br/>
> 在其 public 目錄下產出 (html / css / js) 等網頁檔 output，<br/>
> 之後上傳這些 web output 到靜態空間中 (ex: Github Page)，<br/>

因此將編輯和 output 分為兩個 Repo 的確是很直覺的想法。<br/>

但這樣其實相當麻煩，<br/>
在編輯完 Blog 後不僅要手動產生 public 中的 web output，<br/>
又要手動上傳這些檔案到靜態空間上，<br/>
且 public 內的檔案其實在我們編輯 Blog 時完全用不到，<br/>
最後又要同時維護兩份 Repo，並不是很輕鬆的做法，我用 Hugo 就是想耍懶啊。 <br/>

幸好後來找到了這篇文章<br/>
[利用Travis CI和Hugo將Blog自動部署到Github Pages](https://axdlog.com/zh/2018/using-hugo-and-travis-ci-to-deploy-blog-to-github-pages-automatically/)，<br/>
給了我很大的幫助，感恩讚嘆 AxdLog 大大。

如果你悟性不錯可以直接看那篇就好，這邊簡單解釋一下。 <br/>

原理就是透過 git branch 將 Hugo Project 切割為，<br/>
編輯 Blog 的 **code branch (br-code)** 以及產生 output 用的 **master branch (br-master)**。<br/>

當 Hugo Project 編輯完畢後，在 **(br-code)** commit 並上傳到 Remote Repo，<br/>
此時會觸發 Travis 啟動 CI 流程，協助你用 **(br-code)** 的 Hugo Project 製造出 **(br-master)** 要的 web output。<br/>
最後 Travis 替你在 **(br-master)** 上面 commit， 自動更新 Github Page。<br/>

下面就來依序配置 **(br-code)** 和 **(br-master)**。

**配置 (br-code)**

首先新增 git 以及 (br-code)

```java
git init
git checkout -b code
```

讓 **(br-code)** 忽略 public 資料夾

```java
echo '/public/' >> .gitignore
gsed -r -i '/^\/public\/$/{$!d}' .gitignore
```

> 因為 sed 的指令無法使用，所以更新為 gsed。

將改變推到 remote 

```java
git add .

// 這邊可能會有 submodule 的 warning，因為之後要串 Travis 
// 所以這邊暫時將 hugo-nuo 去模組化，以資料夾的形式來 git add

git rm -f --cached themes/hugo-nuo
git add themes/hugo-nuo/ 

git commit -m "Hugo generator code"
git remote add origin git@github.com:kuotinyen/kuotinyen.github.io.git
git push -u origin code
```

產生 public folder 並暫時存放某資料夾 (之後會刪除)

```java
hugo
HUGO_TEMP_DIR=$(mktemp -d)
cp -R public/* "$HUGO_TEMP_DIR"
```

**配置 (br-master)**

產生 **(br-master)** 並 clean folder

```java
git checkout --orphan master
rm .git/index
git clean -fdx
```

將剛剛暫存的資料夾重新放回 public 

```java
cp -R "$HUGO_TEMP_DIR"/. .
```

將改變推到 remote 

```java
git add .

// 這邊可能會有 submodule 的 warning，因為之後要串 Travis 
// 所以這邊暫時將 hugo-nuo 去模組化，以資料夾的形式來 git add

git rm -f --cached themes/hugo-nuo
git add themes/hugo-nuo/ 

git commit -m 'Initial blog content'
git push -u origin master
```

砍掉剛剛的暫存資料夾

```java
[[ -d "$HUGO_TEMP_DIR" ]] && rm -rf "$HUGO_TEMP_DIR"
```

------

### Travis CI

首先打開 xxx.github.io 的開關，<br/>
之後如圖所示設定 Github 相關資訊：

![Travis Settings](/media/posts/use-travis-distribute-hugo-on-github-page/travis-settings.png)


- GITHUB_USERNAME 你的 github 名稱
- GITHUB_EMAIL 你的 Github email (ex: xxx@gmail.com)
- GITHUB_TOKEN 你的 Github Developer Token
- CNAME_URL 你想使用的 Custom Domain


最後，切回 **(br-code)** 並在 Blog 根目錄新增一個 .travis.yml 檔案，<br/>


```java
git checkout code
vim .travis.yml
```

將下面的 go 程式碼填入 yml 中，<br/>
然後 commit 到 remote 去，Travis 應該會在這次 commit 發動。<br/>

```java
# https://docs.travis-ci.com/user/deployment/pages/
# https://docs.travis-ci.com/user/reference/xenial/
# https://docs.travis-ci.com/user/languages/go/
# https://docs.travis-ci.com/user/customizing-the-build/

dist: xenial
language: go
go:
    - master

# before_install
# install - install any dependencies required
install:
    - go get github.com/gohugoio/hugo    # consume time 70.85s

before_script:
    - rm -rf public 2> /dev/null

# script - run the build script
script:
    - hugo
    - echo "$CNAME_URL" > public/CNAME

deploy:
  provider: pages
  skip-cleanup: true
  github-token: $GITHUB_TOKEN  # Set in travis-ci.org dashboard, marked secure
  email: $GITHUB_EMAIL
  name: $GITHUB_USERNAME
  verbose: true
  keep-history: true
  local-dir: public
  target_branch: master  # branch contains blog content
  on:
    branch: code  # branch contains Hugo generator code
```

如果有缺少 content 路徑的錯誤訊息，<br/>
那是因為新增第一篇文章後才會產生 content 目錄，<br/>
解決的方法很簡單，新增一篇測試的文章來產生 content 目錄：

```java
hugo new post/test.md
```

注意 commit 這個 test.md 時，在文章資訊的 draft 欄位不可為 true，<br/>
因為如果文章被 Hugo 視為草稿，就會在產生 web output 時被略過，<br/>
而不將這篇文章更新到 Github Page 上，無 content 目錄的錯誤會繼續出現。 <br/>

換句話說，如果你希望這篇文章暫時不要發布的話，<br/>
draft 欄位請設為 true。 <br/>

之後任何編輯 Blog 的行為都會透過 Travis CI 幫你自動化產生 Github Page 了，可喜可賀。

分享結束，祝你順利: ) <br/>

