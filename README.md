==== MOD_DOSDETECTOR2 ====

このソフトウェアは下記のソースを参考に作成されています。

- [mod_dosdetector](https://github.com/stanaka/mod_dosdetector)
- [mod_dosdetector-fork](https://github.com/tkyk/mod_dosdetector-fork)
- [DoS攻撃からサーバーを守る、mod_dosdetector導入事例のご紹介 : Raccoon Tech Blog [株式会社ラクーン 技術戦略部ブログ]](http://techblog.raccoon.ne.jp/archives/36202484.html)

オリジナルのmod_dosdetectorと同じライセンス（mod_dosdetector.cに記載）で公開されます。


## mod_dosdetectorとは？
mod_dosdetectorはDoS攻撃を検出するためのApacheモジュールです。
検出結果は環境変数に設定されるため、
他のモジュールと連携することで柔軟な防御・回避アクションを実行すること可能です。

## インストール

```
tar xvzf mod_dosdetector2-X.Y.Z.tar.gz
cd mod_dosdetector2-X.Y.Z
sudo make install
```

apxsコマンドのパスが環境変数PATHに含まれていない場合、
makeに対する引数として設定してください。

```
sudo make PATH=/usr/sbin:$PATH install
```

下記の設定例に従って設定を行った後、
Apacheを再起動してください。


## 設定例

どのような基準でDoS攻撃と見なすかを設定します。
httpd.confに記述してください

```
# mod_dosdetectorを有効にする
DoSDetection on

# 画像、CSS、JavaScriptファイルをDoSチェックの対象としない
<IfModule setenvif_module>
    SetEnvIf Request_URI "\.(gif|jpe?g|png|ico|css|js)$" DoSIgnoreEnv
</IfModule>

# 1. 同一IPアドレスから5秒間に20回以上のアクセスがあった場合
#  「DoS攻撃の疑いあり」と見なし、その後30秒間のアクセスに対しては
#   環境変数 SuspectDoS をセットする（値は”1″）
# 2. さらに5秒間のアクセス回数が35回に達した場合、
#   「激しいDoS攻撃の疑い」と見なし、環境変数 SuspectHardDoS もセットする（値は”1″）
# 3. 初めに「DoS攻撃の疑いあり」と判定してから30秒が経過したら、
#    次のアクセスでもう一度判定をやり直す。直近5秒のアクセスが20回を下回っていれば、疑いは晴れる
DoSPeriod        5
DoSThreshold     20
DoSHardThreshold 35
DoSBanPeriod     30

# 同時に追跡するIPアドレスは100件まで
DoSTableSize     100

# X-Forwarded-Forの判定を有効にする
DoSForwarded     on

# モジュールが使用する共有メモリの名称
DoSShmemName	 mod_dosdetector

# setenvifモジュールが使用不可能な場合はcontent-typeを用いて
# DoSチェックの対象としないアクセスを指定することもできる
# （ただしパフォーマンスは劣る）
<IfModule !setenvif_module>
    DoSIgnoreContentType  image|javascript|css
</IfModule>


##### DoS攻撃と見なされたアクセスに対して、どのように対応するかを設定
##### 必要に応じて<VirtualHost>セクションや.htaccessに記述してください

# 「DoS攻撃の疑いあり」と見なされたアクセスを通常のログとは分けて記録する
LogFormat "%{SuspectHardDoS}e %h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\"" dosdetector
CustomLog logs/dos_suspect_log dosdetector env=SuspectDoS

# 「激しいDoS攻撃の疑い」と見なされたクライアントに対し、503エラーレスポンスを返す
# （mod_rewriteモジュールが必要）
RewriteEngine On
RewriteCond %{ENV:SuspectHardDoS} =1
RewriteRule .*  - [R=503,L]
```
