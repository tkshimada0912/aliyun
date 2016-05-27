# SLB

SLBとは・・・Server Load Balancer。１つのサービスポートで、バックエンドの複数のサーバーにリクエストを分散することで、
サービスの可用性UPと性能UPを実現する仕組み。

AliCloudでは、SLBサービスでTCP、HTTPによる負荷分散をサポートする。

NWコンフィグレーションによって、Classic/VPCの別があり、かつ、Public/Private向けの負荷分散動作が可能なので、
このマトリックスで動作検証を行った。

## Classic network with Public IP address, HTTP

Classic network configurationにて生成されたECSに対して、public IPアドレスで負荷分散を行う構成。

<img src="https://drive.google.com/uc?export=view&id=0Bxf736wnLrNVYzFxWnVnTWRfR00">

3台作って、SLBのインスタンスを起こす。

<img src="https://drive.google.com/uc?export=view&id=0Bxf736wnLrNVV1RQMHA5ak1VUG8">

SLBインスタンスはまずはInternet。ここでは特に選択肢は出ない。

<img src="https://drive.google.com/uc?export=view&id=0Bxf736wnLrNVSDRXaDBON0Z6Q28">

グローバルIP（Internet）がアサインされる。pingは応えるが、Listnerが無いため、まだ何もサービスは動いていない状態。

<img src="https://drive.google.com/uc?export=view&id=0Bxf736wnLrNVa1ZDeU1aUGFwWEE">

Listnerを追加。TCP/HTTP/HTTPS/UDPが選べる。HTTP/HTTPSでは、ヘルスチェック用のファイルURIが指定できる。

「Obtain true IP」はドキュメント上、OFF（※OFFの場合：L7でのフォワーディングとなり、X-Forwarded-forヘッダで
実クライアントIPを取る必要がある、と書いてある）があると書いてあるが、今のところどの設定でも「open(default)」のまま。
Classic Networkの場合、LBのアドレスがアクセス元となり、X-Forwarded-Forに実クライアントIPが入る。

<img src="https://drive.google.com/uc?export=view&id=0Bxf736wnLrNVTEx2Uk9pc3JQSEk">

次の画面でなんだかいろいろ言われるが、ポイントは以下。
- Domain Name:バックエンドサーバーがマルチドメイン対応していることを想定し、Hostヘッダを言ってくれる機能。
- Check route:ヘルスチェック用のファイルのURI。check.htmlとか指定しておくと、このファイルの有無でサーバーへの振り分けを制御できる。

これで設定は完了。このままだとSLBは動かない。なぜなら、バックエンドサーバーでサービスが上がってないから。

<img src="https://drive.google.com/uc?export=view&id=0Bxf736wnLrNVRTE4emE5ajZkZ0U">

バックエンドサーバーでapaheを上げてみる。

    yum install httpd
    systemctl enable httpd
    systemctl start httpd
    touch /var/www/html/check.html

すべてのECSインスタンスを「Backend Server」に追加すると、これでサービスが有効になる。

<img src="https://drive.google.com/uc?export=view&id=0Bxf736wnLrNVcXR0WWdGWndzdmM">

ヘルスチェックは結構バラバラといろんなアドレスから来る。うっとおしいのでログはSetEnvIf設定などでファイルを分けるか、
捨てるかしないと散らかる。

    100.109.80.59 - - [27/May/2016:12:49:39 +0800] "HEAD /check.html HTTP/1.0" 200 - "-" "-"
    100.109.80.24 - - [27/May/2016:12:49:39 +0800] "HEAD /check.html HTTP/1.0" 200 - "-" "-"
    100.109.80.34 - - [27/May/2016:12:49:39 +0800] "HEAD /check.html HTTP/1.0" 200 - "-" "-"
    100.109.80.12 - - [27/May/2016:12:49:41 +0800] "HEAD /check.html HTTP/1.0" 200 - "-" "-"
    100.109.80.49 - - [27/May/2016:12:49:41 +0800] "HEAD /check.html HTTP/1.0" 200 - "-" "-"
    100.109.80.30 - - [27/May/2016:12:49:41 +0800] "HEAD /check.html HTTP/1.0" 200 - "-" "-"
    100.109.80.45 - - [27/May/2016:12:49:41 +0800] "HEAD /check.html HTTP/1.0" 200 - "-" "-"
    100.109.80.10 - - [27/May/2016:12:49:43 +0800] "HEAD /check.html HTTP/1.0" 200 - "-" "-"

このゾーンの設定（singapore）では、「100.109.80.0/26」のレンジからヘルスチェックが来ている。
おそらく全IPからヘルスチェックが来るような気配（check.htmlを削除して、死んでいる状態にすると
ヘルスチェック元のIPが一部変わるが、レンジには変更無し）。

アクセスはprivate側、eth0にアクセスが入ってくる。ECSインスタンスにこの静的経路が切られている。

check.htmlを削除すると、そのサーバーはBackendから削除される。curlで応答を見る限り、特にSLBが入ることで
応答速度が落ちるような気配は見えない（小さなコンテンツを指定した場合）。
この状態だと負荷分散はランダムで、ラウンドロビン等の法則性は無さそう。

この状態でForwarding rulesをmininum number of linksに動的に変更可能。特にlinkを専有するような
トランザクションが無い限り、やはりランダム。

Backend ServerへのCookieベースでのセッション固定も可能で、ここではCookieを書き換えるか、
Insertするかを指定できる。

<img src="https://drive.google.com/uc?export=view&id=0Bxf736wnLrNVYUpvYl95bHkteUE">

レスポンスにCookieが返される。このCookieを次のリクエストに入れることで、Backend Serverがひも付けされる。

    HTTP/1.1 200 OK
    Date: Fri, 27 May 2016 05:09:58 GMT
    Content-Type: text/html; charset=UTF-8
    Content-Length: 2
    Connection: keep-alive
    Last-Modified: Fri, 27 May 2016 05:00:19 GMT
    ETag: "2-533cbc8869906"
    Accept-Ranges: bytes
    Set-Cookie: SERVERID=35d4357ba2e057e332013422fc97ec5f|1464325798|1464325798;Path=/

このCookie（SERVERID）は、クライアントIPには関係無く、都度新規にランダムなIDを別セッションとして生成してくる。
NAT環境などからのアクセスでもきちんと機能するCookie制御が実現されている。

## Classic network with Public IP address, TCP

HTTPではCookieに対応したL7 proxy動作によるSLBになっていたが、TCP(UDPも）を指定すると、動きが若干変わる。
新たにport 8080でバックエンドは80を指定したlistnerを上げてみる。

<img src="https://drive.google.com/uc?export=view&id=0Bxf736wnLrNVZFI1a01KSW9aUmc">

keep sessionを有効化すると、TCPの場合、IPベースの固定しかできないと表示される（Cookie等を入れることができないため）。
また、アクセス元IPアドレスが直接ログで見えるようになる（DNATによる負荷分散動作）。

    202.45.12.145 - - [27/May/2016:13:30:25 +0800] "GET /test.html HTTP/1.1" 200 2 "-" "curl/7.43.0" "-"
    202.45.12.145 - - [27/May/2016:13:30:26 +0800] "GET /test.html HTTP/1.1" 200 2 "-" "curl/7.43.0" "-"
    202.45.12.145 - - [27/May/2016:13:30:26 +0800] "GET /test.html HTTP/1.1" 200 2 "-" "curl/7.43.0" "-"
    202.45.12.145 - - [27/May/2016:13:30:27 +0800] "GET /test.html HTTP/1.1" 200 2 "-" "curl/7.43.0" "-"
    202.45.12.145 - - [27/May/2016:13:30:28 +0800] "GET /test.html HTTP/1.1" 200 2 "-" "curl/7.43.0" "-"
    202.45.12.145 - - [27/May/2016:13:30:29 +0800] "GET /test.html HTTP/1.1" 200 2 "-" "curl/7.43.0" "-"
    202.45.12.145 - - [27/May/2016:13:30:33 +0800] "GET /test.html HTTP/1.1" 200 2 "-" "curl/7.43.0" "-"
    202.45.12.145 - - [27/May/2016:13:31:16 +0800] "GET /test.html HTTP/1.1" 200 2 "-" "curl/7.43.0" "-"

経路としては、eth0（private）から入り、default routeに従い、戻りはeth1（Public）から出て行く非対称経路になる。

この場合も、「Obtain true IP」は「open(default)」から変更できない。

ドキュメントには「Backend ECS configuration」として、一般的なSLB環境下のECS設定（OS設定）が指定されているが、
おそらくClassic Networkではこれを指定することは必要ないと思われる。

## Classic network with Intranet IP address

次はintranetを試してみる。

<img src="https://drive.google.com/uc?export=view&id=0Bxf736wnLrNVcXViVFdEQUU4NjQ">

intranetを指定すると、ClassicとVPCが指定できるが、ここではClassicを選ぶ。

Listnerの設定で、HTTPを指定した場合、普通にアクセスが可能。ECSインスタンスからIntranetアドレス経由で（100.X側）
アクセスすることができる。

ただし、Listnerの設定でTCP、UDPを指定したL4設定でのアクセスは不可能。これはFAQ「Why does the endpoint fail to get through when I telnet the backend ECS of 4-layer SLB?」に記載されている通り、戻りパケットがSLBに戻らないため。
おそらくVPCではこれを回避する経路が組めると想定する。

## VPC network with Intranet IP address

VPC配下にECSを配置して、SLBからアクセスさせる。

まず、default vswitch以下に2台、別のvswitch以下に１台のECSインスタンスを起動。

<img src="https://drive.google.com/uc?export=view&id=0Bxf736wnLrNVMV9qdkNsUk1UN1U">

SLBのインスタンスを起こす。NW設定は「Intranet」の「VPC」で、default vrouter、default vswitchを指定。

<img src="https://drive.google.com/uc?export=view&id=0Bxf736wnLrNVNkxlWjZha3E2RlU">

これでSLBができるが、このSLBインスタンスはVIPがvswitch以下に現れる。

<img src="https://drive.google.com/uc?export=view&id=0Bxf736wnLrNVNFdhOWpDbzMyYnM">

ちょっと想定外な構成。とりあえずListenerを設定して、まずはHTTP（L7）を設定してみる。

<img src="https://drive.google.com/uc?export=view&id=0Bxf736wnLrNVVGxhdDQyM1hRY0k">

同じvrouter以下なら、vswitchが別でもBackend Serversに指定は可能。

HTTP(L7)負荷分散を設定すると、同じvswitch配下からもSLB経由でアクセス可能。TCP（L4）では通信できない。
これは前述の「Why does the endpoint fail to get through when I telnet the backend ECS of 4-layer SLB?」と
同じ問題と思われる。

つまり、VPC設定におけるSLB設定では、内部トラヒックの分散にしか使えない。EIPの付与等は不可能となっており、
外部アクセスとの連携ができないのは、意図的な設計か否か不明。

## Certificates設定によるSSL SLB動作について

SLBのメニューには「Certificates」の設定があり、SSLアクセラレータとしての機能を具備している。

    openssl genrsa 2048 > privkey.pem
    openssl req -new -key privkey.pem > server.req
    openssl x509 -days 3650 -req -signkey privkey.pem < server.req > server.crt

できたkeyとcsrをSLBのCertificates設定に追加する。

<img src="https://drive.google.com/uc?export=view&id=0Bxf736wnLrNVZHN4TGxRdHpORG8">

設定が完了する。

<img src="https://drive.google.com/uc?export=view&id=0Bxf736wnLrNVQzVmT1RSbVpsbTA">

これをSLBでHTTPSのlistnerに設定する。

<img src="https://drive.google.com/uc?export=view&id=0Bxf736wnLrNVdTFDTzJlMTVGdWM">

そうすると、設定したSSL証明書でのSSLアクセスが可能になる。

    [root@iZ229z7584uZ ~]# openssl s_client -connect 172.21.0.6:443
    CONNECTED(00000003)
    depth=0 C = JP, ST = Some-State, O = Internet Widgits Pty Ltd, CN = test.test.jp
    verify error:num=18:self signed certificate
    verify return:1
    depth=0 C = JP, ST = Some-State, O = Internet Widgits Pty Ltd, CN = test.test.jp
    verify return:1
    ---
    Certificate chain
     0 s:/C=JP/ST=Some-State/O=Internet Widgits Pty Ltd/CN=test.test.jp
       i:/C=JP/ST=Some-State/O=Internet Widgits Pty Ltd/CN=test.test.jp
    ---
    Server certificate
    -----BEGIN CERTIFICATE-----
    MIIDNDCCAhwCCQCUG5ENHVYvBzANBgkqhkiG9w0BAQUFADBcMQswCQYDVQQGEwJK
    UDETMBEGA1UECBMKU29tZS1TdGF0ZTEhMB8GA1UEChMYSW50ZXJuZXQgV2lkZ2l0
    cyBQdHkgTHRkMRUwEwYDVQQDEwx0ZXN0LnRlc3QuanAwHhcNMTYwNTI3MDgwMTIx
    WhcNMjYwNTI1MDgwMTIxWjBcMQswCQYDVQQGEwJKUDETMBEGA1UECBMKU29tZS1T
    dGF0ZTEhMB8GA1UEChMYSW50ZXJuZXQgV2lkZ2l0cyBQdHkgTHRkMRUwEwYDVQQD
    Ewx0ZXN0LnRlc3QuanAwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQDL
    We5D1sXBJi0F8jHmJR4/vW/WGBle7ZUwg3z6ji292t0N7/RrIY0gomsgMJbuwss+
    GHXilJAathpEkDuVotROoIU7hlQZcEBFBPAp/pAhTIKqD05W3f8DJT58I37zofhW
    F/TtI96LUvOWwvAAQb+B2yfNmH5HA2dgBSG/gLvUyNmWxLcdtMUQxZJps/IBhzWp
    CmO2+tCXRiBho+bo5f4Wm/lBQ73qE+hCazabiDIfwnlfhOSZk0YBdwB63bAkY0W9
    o0xNuzS8aZ/1m2rvj/ZR3JOytPptWb0C7bjL6ShB5XdWeAaWBF8YbrW/99sdlvp1
    IG0Y8KnrEN6/6qdTm+mbAgMBAAEwDQYJKoZIhvcNAQEFBQADggEBAE7XAu1F86EA
    wDEz5axzvsuywOrELT0y3M/E0JuPxAXkLmw5OUw9vhGAht7e64uooZ10gm+S5Zr6
    JQr8d2M0nNVEY7TH0d7ot4EG05a1WTmB7Mi6zYmEpvFCRp7rZHhg4WmrESeWtIIa
    1/wtE+3M7iudv+6LEuCSuBatSsgYgaXAFI5SdxVrN95owhOP/Dhban3/m21zn2o6
    nsWVfaHHQBaojcrB/xzGnW56O8+vtp3lYr/JAm/v+pDrADcwo4AQr98ZWs9NIYwp
    Qwar6uDBFBi/pr6jlcCJejIcUhOZWypb2d0pXn7E/bhCh6Uzz9awmGY3y3JeMTUV
    mRLSoKWCQNg=
    -----END CERTIFICATE-----
    subject=/C=JP/ST=Some-State/O=Internet Widgits Pty Ltd/CN=test.test.jp
    issuer=/C=JP/ST=Some-State/O=Internet Widgits Pty Ltd/CN=test.test.jp
    ---
    No client certificate CA names sent
    Server Temp Key: ECDH, prime256v1, 256 bits
    ---
    SSL handshake has read 1499 bytes and written 375 bytes
    ---
    New, TLSv1/SSLv3, Cipher is ECDHE-RSA-AES128-GCM-SHA256
    Server public key is 2048 bit
    Secure Renegotiation IS supported
    Compression: NONE
    Expansion: NONE
    SSL-Session:
        Protocol  : TLSv1.2
        Cipher    : ECDHE-RSA-AES128-GCM-SHA256
        Session-ID: 6D39E1D177ACEF7AD82F7F356293CE39699F26C58F4F49A2836E874B4F3BEB27
        Session-ID-ctx:
        Master-Key: BA495CFE1AE084A1A885CCFBEBF897D10A683F99E09DC08EDF0C5096680262CB2C684E9FFEB5CBF2CA31FE3B2C0A2E8A
        Key-Arg   : None
        Krb5 Principal: None
        PSK identity: None
        PSK identity hint: None
        TLS session ticket lifetime hint: 300 (seconds)
        TLS session ticket:
        0000 - e4 70 18 49 46 d0 35 d1-7d 4e e8 0f 5f 60 cf 55   .p.IF.5.}N.._`.U
        0010 - 82 d8 d7 8f 31 07 0a f8-67 5c 2c c8 a8 e6 3e d6   ....1...g\,...>.
        0020 - f3 c1 56 22 f1 af 99 ec-69 3e 7e e2 a0 bc 66 5e   ..V"....i>~...f^
        0030 - 6a 77 c5 36 cc 0e 96 13-e4 b8 1a e0 fd 95 c0 da   jw.6............
        0040 - 2c 23 3d 7c 6c 23 53 8f-ad e4 47 94 74 4f 81 19   ,#=|l#S...G.tO..
        0050 - 6b 36 07 b9 db c4 dd 42-c7 a0 3f a2 2a 0a 29 7b   k6.....B..?.*.){
        0060 - 24 fd 15 17 a6 05 ee f1-98 1b b5 ce c1 bb f5 82   $...............
        0070 - 60 e0 bb fb 36 21 40 63-f0 af 62 a4 85 41 bc a6   `...6!@c..b..A..
        0080 - 2f 45 c7 19 80 d9 ba 85-2a 9d 1b c7 43 38 e9 3f   /E......*...C8.?
        0090 - cc ca ce 92 3e 5b cc 21-21 e5 06 43 f0 7c 8c 84   ....>[.!!..C.|..
        00a0 - 73 12 24 23 5a 14 d0 d8-72 86 19 13 6f bb a0 77   s.$#Z...r...o..w
    
        Start Time: 1464336444
        Timeout   : 300 (sec)
        Verify return code: 18 (self signed certificate)
    ---

## Conclusion

ヘルスチェック型の負荷分散、SSLアクセラレーションが可能。
エンドポイントとしてはグローバルとプライベートが設定可能だが、TCP（L4）の設定では通信に若干の制限が出る。
HTTP/HTTPSでの接続（L7）であればソースアドレスによらずアクセスが可能。

特にVPC環境ではグローバル側に代表アドレスを設定できないのがよくわからない。
