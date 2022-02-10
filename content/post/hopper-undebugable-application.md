---
title: "Hopperで特定のアプリケーションをデバッグできなかった話"
date: 2022-02-10T20:38:45+09:00
draft: false
categories: [セキュリティ]
tags: ["デコンパイラ"]
---

### 前置き

あ、どうも暇人です。最近は[Wordle](https://www.powerlanguage.co.uk/wordle/)がアツいですね、世界大会出場を目指していきたいと思います。  
さて、長らくマルウェアの解析等に[IDA無料版](https://hex-rays.com/ida-free/)や[Ghidra](https://ghidra-sre.org/)を使ってきたのですが、今回はMacBook Proを購入したということもあり、  
どうせならと以前から気になっていた[Hopperapp](http://hopperapp.com/)を使ってみることにしました。  

### 原因不明…ってコト！？

しばらく使っているうちに、デバッガーがエントリーポイントを読み込んだ直後に停止してしまう問題に直面しました。  
類似したケースも見つからずに困っていたところ、[IDAの記事](https://hex-rays.com/wp-content/static/tutorials/mac_debugger_primer2/mac_debugger_primer2.html)にそれらしきものが書かれていたので試してみることに。

>Despite the fact that mac_server64 is codesigned, it still failed to retrieve permission from the OS to debug the target app.  
This is because Calculator.app and all other apps in /System/Applications/ are protected by System Integrity Protection and they cannot be debugged until SIP is disabled.  
Note that the error message is a bit misleading because it implies that running mac_server64 as root will resolve the issue - it will not.  
Not even root can debug apps protected by SIP.

>System Integrity Protectionによってアプリケーションが保護されているため、mac_server64が署名されているにも関わらず権限取得に失敗してしまう。  
SIPで保護されたアプリケーションはrootでもデバッグすることができない。

はい、ということで[リカバリーモード](https://support.apple.com/ja-jp/HT204904)からSIPをオフにしていきますが、  
実機では推奨できるようなものではないので無難に仮想環境でも使ったほうがよいでしょう。

```console
~ % csrutil disable
Successfully disabled System Integrity Protection. Please restart the machine for the changes to take effect.
~ % csrutil status
System Integrity Protection status: disabled.
```
SIPが無効化されていることを確認してデバッガーを使ったところ、正常に動作しました。  
ようやくEmotetをしばき倒すことができますね、お疲れさまでした。