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


### 参考

https://github.com/libimobiledevice/idevicerestore/issues/697#issuecomment-2568445357

## 追記

前述のように証明書を用意してもなお、エラー「`1109`」が送出される場合がある。

[「エンタープライズネットワークでApple製品を使う」のソフトウェアアップデート節](https://support.apple.com/ja-jp/101555#software)に、アップデート・復元時に使うホストが、`gs.apple.com`含め羅列されている。
つまり、このいずれかのホストを解決できない時に`1109`が送出されるものと思われる。

アップデート・復元時のログは`%LOCALAPPDATA%\Packages\AppleInc.AppleDevices_xxxxxxxxxxxxx\LocalCache\Roaming\Apple\AMPDevicesAgent\iPhone Updater Logs\`以下にある。
1109を送出した際のログを遡ると、次のような記述がある。

```plaintext
[xx:xx:xx.xxxx] amai: tss_submit_job_with_retry: TSS Connection attempt 1 of 3.  (Will retry if TSS_ERR_SERVER_NOT_REACHABLE.)
…
[xx:xx:xx.xxxx] _AMSupportHttpMessageSendSyncRetry: !!!!!!!!!!!!!!!!!!!!!!!!!! Retries exhausted on attempt 3 !!!!!!!!!!!!!!!!!!!!!!!!!!
…
[xx:xx:xx.xxxx] WKMS: No authorization token available for wkms.sd.apple.com; failing
…
[xx:xx:xx.xxxx] CFError domain:AMRestoreErrorDomain code:1109 description:Error retrieving WKMS decryption key
```

ログより、今回の場合の「1109」とは、インストール作業中（`amai`）に、ウェブ鍵管理サービス（`WKMS`）に復号鍵を問い合わせたが、その問い合わせ先「`wkms.sd.apple.com`」に到達できないという意味であることがわかる。
`wkms.sd.apple.com`は前述の羅列の中に存在せず、アクセスも不可。

```plaintext
PS> ping wkms.sd.apple.com
ping 要求ではホスト wkms.sd.apple.com が見つかりませんでした。ホスト名を確認してもう一度実行してください。
```

似たような`wkms-public.apple.com`というホストはあることから察するに、恐らくどこかで取り違えたのだろう。

ならば、一つの解決策として「[Unbound](https://nlnetlabs.nl/projects/unbound/)等を噛ませ、`wkms.sd.apple.com. IN CNAME wkms-public.apple.com.`等とし、前者への問い合わせを強制的に後者へとすり替える」が挙げられる。
私は試していないし、試す気にもならない。「何故、わざわざここまでせねば」と足踏みしてしまう。そして恐らく、同じ考えの者は多いはず。

### 参考

https://discussions.apple.com/thread/256344675