---
title: "HugoとGitHub Pagesでブログを作成した話"
date: 2021-07-07T13:58:40+09:00
draft: false
category: プログラミング
tag: ["Hugo", "GitHub"]
---

### Q. Web 系の知識なしでブログを作るには

WordPress ってなに！！！助けて！！

不安にかられているそこの貴方  
そんな非 Web 系エンジニアでも

- [x] 絶対 100%
- [x] 完全放置
- [x] 不労所得を生み出す

ブログを簡単に展開することができます。  
~~世の中そんなに甘い話はないので頑張りましょう。~~

### Hugo とは？

最近話題の静的サイトジェネレーター  
テンプレートを選ぶと簡単にサイトが構築できるぞ ♨

### GitHub Pages とは？

レポジトリからサイトを公開できる静的なホスティングサービス。  
カスタムドメインも使える優れもの。

### もう待ちきれないよ！早く教えてくれ！

まずは Hugo を用意するところから。  
筆者の環境は Windows なので Chocolately を使っています。  
他環境の方は別途で[インストール方法](https://gohugo.io/getting-started/installing)を参照してください。

コマンドプロンプトを管理者権限で実行したら下記のコマンドを入力します。

```cmd
$ choco install hugo -confirm
```

```cmd
$ choco install hugo-extended -confirm
```

hugo-extended は scss 対応版になります。  
よく分からない場合は両方入れましょう。  
そしたら適当なディレクトリで

```cmd
$ hugo new site blog (自分が分かりやすいものでOK)
$ cd blog
$ git init

themes.gohugo.io
自分はm10cというテーマを選びました。
$ git submodule add https://github.com/vaga/hugo-theme-m10c themes/m10c

scssを編集するのであればforkした後にsubmoduleとして追加します。
$ git submodule add https://github.com/Lutwidse/hugo-theme-m10c themes/m10c
```

さて、ここまで来たらローカルでサーバーを立ち上げることが可能ですが、  
その前に[m10c のサンプル](https://github.com/vaga/hugo-theme-m10c/tree/master/exampleSite)を見ながら config.toml の設定を変えておきましょう。

当サイトでは以下のようになっています。

```toml
baseURL = "https://lutwidse.github.io"
languageCode = "ja-jp"
title = "水の飲み方を忘れないための備忘録"
theme = "m10c"
paginate = 8

[menu]
  [[menu.main]]
    identifier = "home"
    name = "Home"
    url = "/"
    weight = 1

[params]
  author = "Lutwidse"
  description = "メモ帳として使う予定"
  # blog/static/に置くとよい。
  avatar = "/site/avatar.png"
  menu_item_separator = " - "
  [[params.social]]
    icon = "github"
    name = "GitHub"
    url = "https://github.com/Lutwidse"
  [[params.social]]
    icon = "twitter"
    name = "Twitter"
    url = "https://twitter.com/Lutwidse0xff"

  [params.style]
    darkestColor = "#242930"
    darkColor = "#353b43"
    lightColor = "#afbac4"
    lightestColor = "#f5fffa"
    primaryColor = "#c2c4c6"
    # scssを編集したため追記。後ほど記載。
    textColor = "#fffff0"
```

では、サーバーを立ち上げてみましょう。

```cmd
$ hugo server -dW
```

ブラウザで[127.0.0.1:1313](http://127.0.0.1:1313)に接続するとサイトが表示されます。  
されない場合はエラーログと丸太を持って筆者の Twitter アカウントに DM で突撃してください。

そしたら今度は記事を生成してみます。

```cmd
$ hugo new post/HelloWorld.md
```

blog/content/に置かれるので編集して、書き終わったら draft:false に変更。
公開用の静的ページをビルドします。

```cmd
$ hugo
```

blog/public/に出力されます。  
Github Pages として公開するには public を push する必要がありますが、  
管理がとても面倒になるのでサブモジュールとして追加しておきます。

```cmd
$ git remote add origin https://github.com/Lutwidse/blog.git
$ git submodule add -b main https://github.com/Lutwidse/lutwidse.github.io.git public
```

```cmd
$ cd public
$ git add .
$ git commit -m "initial commit"
$ git push origin main
```

これにてサイトの公開は完了です。  
お疲れさまでした 😁👊

### テーマで編集したところ

現時点で編集したところを書いておきます。  
[Lutwidse/hugo-theme-m10c](https://github.com/Lutwidse/hugo-theme-m10c)

#### layouts/\_default/baseof.html

```html
<!--冗長だったのでスッキリさせた-->
<h1>{{ .Site.Title }}</h1>
<!-- <h1>{{ .Site.Title }}</h1> -->
...
<!-- 一応の著作権表記 -->
<div class="app-footer-copyright">
  <p>© Lutwidse All rights reserved.</p>
</div>
```

#### assets/css/\_extra.scss

```scss
.app-footer-copyright {
  background: $darkest-color;
  text-align: center;
  bottom: 0;

  p {
    margin: 0 0.1em;
  }

  // オーバーライド
  h1,
  h2,
  h3,
  h4,
  h5,
  h6 {
    border-left-width: 0.2em;
    border-left-style: solid;
    border-left-color: #66daf6;
    padding: 0.5em 0.5em;
  }
}

// オーバーライド
body {
  margin: 0;
  font-family: sans-serif;
  background: $dark-color;
  color: $text-color;
}
```

#### assets/css/main.scss

```scss
// 文字色追加
$text-color: {{ .Site.Params.style.textColor | default "#ffffff" }};
```
