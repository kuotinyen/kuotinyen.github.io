---
title: "用 Travis CI 輕鬆部署 Hugo Blog 到 Github Page"
author: "kuotinyen"
tags: ["Hugo", "Github", "Github Page", "Travis", "Blog"]
date: 2019-02-22T15:10:26+08:00
---

前陣子用 Medium 記錄學習的心得感覺不錯，<br/>
但是不支援 code highlight 和 markdown 在寫技術文章時不太方便。<br/> 
嘗試過 Hexo，也看了一些使用 Jekyll 的分享，甚至考慮用 github 的 issue 來寫 Blog，<br/>
最後用了 Hugo 覺得很順手就決定是它了。

折騰研究了幾天，但是不會太麻煩， 因為 Hugo 的 Blog 生成邏輯蠻簡單的。<br/>
現在發文只管把 markdown 生出來， commit 上 github.io 後，<br/>
Travis CI 就會自動把剛寫的文章生成網頁檔上傳到 Github Page 上，太爽了。


<!--more-->

### TL;DR

這篇文章可以幫助你

- 一步一步建立 Hugo Blog
- 只需要管理一個 github.io Repo，而不是在 github.io Repo 外還要多管理 Hugo Project Repo
- 編輯 markdown 文章或修改 Blog 設定後， 自動將更新部署到 Github Page 上

------

### Hugo

[官方網站](https://gohugo.io) <br/>
[官方網站 - Host on Github](https://gohugo.io/hosting-and-deployment/hosting-on-github/)

後起之秀的 Hugo 在 Github 的 stars 數已經屌打 Hexo，<br/>
但客製化和 plugin 的部分就少 Hexo 一些，不過對我來說已經足夠。<br/>
目錄結構單純好懂，是一個只想懶惰生 markdown 文章的好選擇。


**建立一個 Hugo Blog**

在想要建立 Hugo Blog Project 的地方下指令：

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

**合而為一的 Repo 管理**

多數網路上的教學文都會建議使用兩個 Repo，<br/>
一個是 Hugo Project 的 Repo，用於編輯 Hugo Blog；<br/>
另一個則是負責更新 Github Page 的 xxx.github.io Repo。<br/>

而 Hugo Project 產生靜態 Blog 的原理是：<br/>

> 對 Hugo Project 做任何編輯變更後，<br/>
> 對 Hugo Project 下指令 hugo ，會在 Hugo Project 的 public 目錄下產生網頁檔 output <br/>
> 之後將 public 目錄下的網頁檔上傳到靜態空間中(ex: Github Page)<br/>

因此將 Hugo Blog 和 Github Page Repo 分為兩個 Repo 管理是很直覺的做法。 <br/> 

雖然這樣很直覺，但也其實很麻煩，<br/>
因為修改 Hugo Project 後不但要自己下指令產生網頁檔，又要再上傳這些檔案到靜態空間， <br/>
最後還有兩份 Repo 要維護。

<del>我用 Hugo 就是想耍懶啊！ </del>

幸好找到了這篇文章<br/>
[利用Travis CI和Hugo將Blog自動部署到Github Pages](https://axdlog.com/zh/2018/using-hugo-and-travis-ci-to-deploy-blog-to-github-pages-automatically/)，<br/>
給了很大的幫助，感恩讚嘆 AxdLog 大大。

如果你悟性不錯可以直接看那篇就好，這邊簡單說明一下。 <br/>

原理就是透過 git branch 將 Hugo Project 切割為<br/>
編輯 Blog 的 **code branch (br-code)** 和 <br/>
產生 output 用的 **master branch (br-master)**。<br/>

當 Hugo Project 編輯完畢後，在 **(br-code)** commit 並上傳到 Remote Repo，<br/>
此時會觸發 Travis CI 上的腳本，執行以下操作：
> 切換到 **(br-master)** 自動產生網頁並 commit ，最後部署到 Github Page 上。


下面就來依序配置 **(br-code)** 和 **(br-master)**。

**配置 (br-code)**

首先 init git 以及 (br-code)

```java
git init
git checkout -b code
```

讓 **(br-code)** 忽略 public 資料夾

```java
echo '/public/' >> .gitignore
gsed -r -i '/^\/public\/$/{$!d}' .gitignore
```

將改變推到 remote 

```java
git add .

// 這邊可能會有 submodule 的 warning，因為之後要串 Travis CI 不想牽扯到 git submodule
// 所以這邊暫時將 hugo-nuo 去模組化，以資料夾的形式來做 git add
git rm -f --cached themes/hugo-nuo
git add themes/hugo-nuo/ 

git commit -m "Hugo generator code"
git remote add origin git@github.com:kuotinyen/kuotinyen.github.io.git
git push -u origin code
```

產生 public folder 並暫時存於某資料夾 (之後會再刪除)

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

// 和 code 的部分一樣，暫時解決 git submodule 的問題
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

在 Travis CI 後台打開 xxx.github.io Repo 的開關，<br/>
如圖所示填入一些 Github 相關資訊，這些資訊將被 Travis CI 更新 Github Page 的腳本使用：

![Travis Settings](/media/posts/use-travis-distribute-hugo-on-github-page/travis-settings.png)


- GITHUB_USERNAME：你的 Github 名稱
- GITHUB_EMAIL：你的 Github email
- GITHUB_TOKEN：你的 Github Developer Token
- CNAME_URL：你想使用的 Custom Domain


最後，切回 **(br-code)** 並在 Blog 根目錄新增一個 .travis.yml 檔案，<br/>


```java
git checkout code
vim .travis.yml
```

將下方的 go 腳本填入 .travis.yml 中，<br/>
並且在 **(br-code)** commit 到 remote，Travis CI 的腳本應該會被這個 commit 觸發。<br/>

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

之後任何修改 Blog 的行為都會透過 Travis CI 自動更新 Github Page 了，<br/>
恭喜你！從今以後不需要花任何伺服器費用，也可以只用 markdown 發佈新文章！

---

**小提醒**

如果有缺少 content 路徑相關的錯誤訊息，請確認是否 content 目錄下沒有任何文章。<br/>
因為 Hugo 會在新增第一篇文章後才產生 content 目錄。<br/>

解決的方法很簡單，只要新增一篇測試的文章來產生 content 目錄即可：

```java
hugo new post/test.md
```

請注意 commit 這個 test.md 時，文章資訊的 draft 欄位不可為 true，<br/>
因為如果該文章被 Hugo 視為草稿，就會在產生網頁檔時被略過，<br/>
最後這篇文章將不會被更新到 Github Page 上，缺少 content 目錄的錯誤會繼續存在。 <br/>

然而，如果你希望這篇文章暫時不要發布於 Blog 上的話，<br/>
draft 欄位請設為 true。 <br/>

分享結束，祝你順利: ) <br/>

