---
title: "「Appleデバイス」アプリでiPhoneを復元しようとするとエラー番号1109を吐いて文鎮化する現象の対処法（私の場合）"
emoji: "🔁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["windows", "apple", "ios", "iphone", "エラー解決"]
published: true
---

公式マルウェアと名高い[Appleデバイス](https://apps.microsoft.com/detail/9np83lwlpz9k)。
iPhoneのアップデート・復元を同ソフトで行うと、高確率で処理中に同エラーを吐くのがその所以。

## 今回の発生理由

公式のサポートページ「[iPhoneまたはiPadの復元時にエラー3194、エラー1109、またはその他のネットワーク接続エラーが表示される場合](https://support.apple.com/ja-jp/108396)」を見るに、`gs.apple.com`に到達できないと出るエラーのよう。

`curl`で覗くと、証明書が信頼されておらず、[Secure Channel](https://learn.microsoft.com/ja-jp/windows/win32/secauthn/secure-channel)が拒否していることがわかる。

```
PS> curl -v https://gs.apple.com
* Host gs.apple.com:443 was resolved.
* IPv6: (none)
* IPv4: 17.111.103.15
*   Trying 17.111.103.15:443...
* schannel: disabled automatic use of client certificate
* ALPN: curl offers http/1.1
* schannel: SEC_E_UNTRUSTED_ROOT (0x80090325) - 信頼されていない機関によって証明
* closing connection #0
curl: (60) schannel: SEC_E_UNTRUSTED_ROOT (0x80090325) - 信頼されていない機関によって証明
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the webpage mentioned above.
```

公式の[Apple PKI](https://www.apple.com/certificateauthority/)ページにあるApple Root Certificatesを全て「信頼されたルート証明機関」に入れ、[リカバリモード](https://support.apple.com/ja-jp/118430)にし、再度やると成功した。


## 参考

https://github.com/libimobiledevice/idevicerestore/issues/697#issuecomment-2568445357