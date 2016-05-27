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

HTTP(L7)負荷分散を設定すると、同じvswitch配下からもSLB経由でアクセス可能。














