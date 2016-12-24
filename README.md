# vagrant_firefly tips

# 検証構成
firefly1
- ge-0/0/0: DHCP
- ge-0/0/1: 192.168.34.16/24
- ge-0/0/2: 192.168.35.1/30
- as_num: 65001
firefly2
- ge-0/0/0: DHCP
- ge-0/0/1: 192.168.36.16/24
    - セグメントがfirefly1と重複しているとうまくいかない？
    -  もともと192.168.34.17/24を付けてたけど、上記に変更して上手くいった。
- ge-0/0/2: 192.168.35.2/30
- as_num: 65002
MacBook Air(host)
- vboxnet4: 192.168.34.1

メモ
firefly２台立ち上げると、Memoryが合計7.5Gも使ってしまい、
8.0G memory Macbook Aireではギリギリ。最初はPC固まった。
不要はプロセスは閉じて、2Gx2=4Gほどメモリ空きを確保してから実施しましょう


メモ
firefly２台立ち上げると、Memoryが合計7.5Gも使ってしまい、
8.0G memory Macbook Aireではギリギリ。最初はPC固まった。
不要はプロセスは閉じて、2Gx2=4Gほどメモリ空きを確保してから実施しましょう

試行錯誤メモ
- VirtualBoxのネットワーク設定がNATだとうまくいかない
  昨日HostOnlyAdaptにしたら、Firefly -> MacへのpingだけOKになった。
- private_network, inte_netでも失敗。通信もなにもできない
- trust networkに追加したが、通信もなにもできない
- セグメントを変えてみる private : 192.168.34.xにしてみる
    - あれ、そもそもホストOS側に192.168.34.xのIPが振られてない。そういうものか
    - ダメ。状況変わらず
- public_network にしてみた。
    - ダメ。
- そもそもゲスト-ホスト間で通信できないのが、Vagrantとしておかしい。
    - firefly設定ミスによるものか
    - junos-vagrantがそもそもおかしい


参考にしたVagrant設定ブログ
http://labs.septeni.co.jp/entry/20140707/1404670069


fireflyではVagrantFileで作ったinterfaceはデフォルトuntrustになっていて、通信できないようにしされている
とりあえずtrustにしてあげることで、全通信を許可させる

```
set security zones security-zone trust interfaces ge-0/0/1
set security zones security-zone trust interfaces ge-0/0/2
set security zones security-zone trust interfaces ge-0/0/1.0 host-inbound-traffic system-services ssh
set security zones security-zone trust interfaces ge-0/0/1.0 host-inbound-traffic system-services ping
set security zones security-zone trust interfaces ge-0/0/2.0 host-inbound-traffic system-services bgp
set security zones security-zone trust interfaces ge-0/0/2.0 host-inbound-traffic system-services ping
```

# これいれて、やっと通信できた。。。

```
#初期設定
    ( set system root-authentication plain-text-password )
    ( set system login user user1 authentication plain-text-password )
set system root-authentication encrypted-password "$1$wIdU4IBy$XV2B3LUKzHOf.VCbbcWus0"
set system login user user1 class super-user
set system login user user1 authentication encrypted-password "$1$1ABIAlLk$dTB8LJe00XWXe5GI/rjhO1"
set security zones security-zone trust interfaces ge-0/0/1
set security zones security-zone trust interfaces ge-0/0/2
set security zones security-zone trust interfaces ge-0/0/1.0 host-inbound-traffic system-services all
set security zones security-zone trust interfaces ge-0/0/2.0 host-inbound-traffic system-services all
set system time-zone Asia/Tokyo
set system services netconf ssh port 22
```

# BGP設定
```
set routing-options autonomous-system 65001
set protocols bgp family inet unicast
set protocols bgp group ge-0/0/2 type external
set protocols bgp group ge-0/0/2 neighbor 192.168.35.2 peer-as 65002
set protocols bgp group ge-0/0/2 export advertised_for_firefly2
------
set routing-options autonomous-system 65002
set protocols bgp family inet unicast
set protocols bgp group ge-0/0/2 type external
set protocols bgp group ge-0/0/2 neighbor 192.168.35.1 peer-as 65001
set protocols bgp group ge-0/0/2 export advertised_for_firefly1
```

# 経路広告するためのstatic経路つくり

```
set routing-options rib inet.0 static route 10.10.10.0/24 discard
set routing-options rib inet.0 static route 10.10.20.0/24 discard
set policy-options policy-statement advertised_for_firefly2 term 10 from route-filter 10.10.10.0/24 exact
set policy-options policy-statement advertised_for_firefly2 term 10 then accept
set policy-options policy-statement advertised_for_firefly2 term 999 then reject

----
set routing-options rib inet.0 static route 10.10.30.0/24 discard
set routing-options rib inet.0 static route 10.10.40.0/24 discard
set policy-options policy-statement advertised_for_firefly1 term 10 from route-filter 10.10.30.0/24 exact
set policy-options policy-statement advertised_for_firefly1 term 10 then accept
set policy-options policy-statement advertised_for_firefly1 term 999 then reject


# Flow modeからpacket modeに切替
delete security policies
set security forwarding-options family inet6 mode packet-based
set security forwarding-options family mpls mode packet-based

# 再起動 (パケット転送モードの変更のため)
run request system reboot


# BGP用のフィルタ穴あけ?
set firewall family inet filter ge-0/0/2 term bgp from source-address 192.168.35.x/32
set firewall family inet filter ge-0/0/2 term bgp from protocol tcp
set firewall family inet filter ge-0/0/2 term bgp from destination-port bgp
set firewall family inet filter ge-0/0/2 term bgp then accept
set firewall family inet filter ge-0/0/2 term tcp-established from protocol tcp
set firewall family inet filter ge-0/0/2 term tcp-established from tcp-established
set firewall family inet filter ge-0/0/2 term tcp-established then accept



# 上記のフィルタ削除
delete firewall family inet filter ge-0/0/2
```



コマンド

```
vagrant ssh firefly1
vagrant ssh firefly2
ping 192.168.34.16
ping 192.168.34.17
ping 192.168.33.15
ping 192.168.33.1

ping 192.168.34.16
ping 192.168.34.17
```


# firefly 同士でBGPが貼れない
これが必要
https://www.juniper.net/assets/jp/jp/local/pdf/others/usecase_firefly_router.pdf

```
delete security policies
set security forwarding-options family inet6 mode packet-based
set security forwarding-options family mpls mode packet-based
```

 これはだめ　delete security
 
なんばさんの記事を参考にしました。
http://blog.bit-isle.jp/tech/2014/08/112

mplsでOK??


# Mac側での準備
pip install junos-eznc

```:うまくいかない
*********************************************************************************

Could not find function xmlCheckVersion in library libxml2. Is libxml2 installed?

Perhaps try: xcode-select --install

*********************************************************************************

error: command 'cc' failed with exit status 1

----------------------------------------
Cleaning up...
Command /usr/bin/python -c "import setuptools, tokenize;__file__='/private/var/folders/1z/116vjww53yz3zf0nvbsp6dvm0000gn/T/pip_build_taiji/lxml/setup.py';exec(compile(getattr(tokenize, 'open', open)(__file__).read().replace('\r\n', '\n'), __file__, 'exec'))" install --record /var/folders/1z/116vjww53yz3zf0nvbsp6dvm0000gn/T/pip-9prVRB-record/install-record.txt --single-version-externally-managed --compile failed with error code 1 in /private/var/folders/1z/116vjww53yz3zf0nvbsp6dvm0000gn/T/pip_build_taiji/lxml
Storing debug log for failure in /Users/taiji/Library/Logs/pip.log
```

これでうまくいった。

```
xcode-select --install                                                
```

# 問題: sshパスワード認証でログインしようとすると、mac側に公開鍵を聞かれる。ツールでアクセスできない。
mac側の秘密鍵をキーチェインに登録しておく(Mac限定)
これではだめでした。
ssh-add -K ~/.ssh/id_rsa

ここに足してみることでいけた

```
vi /Users/taiji/.ssh/config
Host 192.168.34.16
  HostName 192.168.34.16
  IdentityFile      /Users/taiji/.vagrant.d/insecure_private_key
  User user1
  PasswordAuthentication no
```

ただしまだツールではうまくアクセスできない

Juniperドキュメント (ただしうまくアクセスできず)
http://forums.juniper.net/t5/Automation/Scripting-How-To-Junos-NETCONF-and-SSH-Part-2/ta-p/279102



#問題 MAC->fireflyへSSHできなくなった。

```
% ssh -vvv user1@192.168.34.16
OpenSSH_6.9p1, LibreSSL 2.1.8
debug1: Reading configuration data /Users/taiji/.ssh/config
debug1: Reading configuration data /etc/ssh/ssh_config
debug1: /etc/ssh/ssh_config line 21: Applying options for *
debug2: ssh_connect: needpriv 0
debug1: Connecting to 192.168.34.16 [192.168.34.16] port 22.
debug1: Connection established.
debug1: key_load_public: No such file or directory
debug1: identity file /Users/taiji/.ssh/id_rsa type -1
debug1: key_load_public: No such file or directory
debug1: identity file /Users/taiji/.ssh/id_rsa-cert type -1
debug1: key_load_public: No such file or directory
debug1: identity file /Users/taiji/.ssh/id_dsa type -1
debug1: key_load_public: No such file or directory
debug1: identity file /Users/taiji/.ssh/id_dsa-cert type -1
debug1: key_load_public: No such file or directory
debug1: identity file /Users/taiji/.ssh/id_ecdsa type -1
debug1: key_load_public: No such file or directory
debug1: identity file /Users/taiji/.ssh/id_ecdsa-cert type -1
debug1: key_load_public: No such file or directory
debug1: identity file /Users/taiji/.ssh/id_ed25519 type -1
debug1: key_load_public: No such file or directory
debug1: identity file /Users/taiji/.ssh/id_ed25519-cert type -1
debug1: Enabling compatibility mode for protocol 2.0
debug1: Local version string SSH-2.0-OpenSSH_6.9
ssh_exchange_identification: read: Connection reset by peer
```

もしかしたら、fireflyを同時に2台たちあげたときのみ事象が発生する？？
(IPが同じセグメントに存在しているのがよくないしてる？？)
でも必ず発生するわけではない。
MacBookを再起動したら、発生しないときもある。

MacBook固有の問題か？？？

# JSNAPyをプログラム上で実行するときにログを出さないようにする。

```
vi /etc/jsnapy/logging.yml
　61 root:
 62     level: DEBUG
 63     #handlers: [console, debug_file_handler]
 64     handlers: [debug_file_handler]
 ```

# JSNAPyプログラムを二回目呼び出したときに、CPU利用率が異常
わらからん。。。

# インストール
環境
- CetOS
- Juniper MX Series

```
jsnapy --snap pre -f sample/config_check.yml
```


# 実行

作成したファイル

```:sample/config_check.yml
hosts:
  - device:  xxx.xxx.xxx.xxx
    username : user1
    passwd: abcdef
tests:
  - /home/t-tsuchiya/sample_JSNAPy/sample/test_bgp_neighbor.yml
```

```:sample/test_bgp_neighbor.yml
test_bgp_neighbor:
  - command: show bgp neighbor
```

実行する

```
jsnapy --snap snap01 -f sample/config_check.yml


jsnapy --snap snap02 -f sample/config_check.yml
```

デフォルトでは/etc/jsnapy/snapshots ディレクトリにファイルが生成される
(jsnapy.cfgで生成場所を変更できるらしい)

```
 ls -al /etc/jsnapy/snapshots/
合計 52
drwxrwxrwx 2 root       root         121  9月 27 08:51 .
drwxrwxrwx 5 root       root          87  9月 26 17:34 ..
-rw-rw-r-- 1 t-tsuchiya t-tsuchiya 20984  9月 27 08:50 192.168.34.16_snap01_show_bgp_neighbor.xml
-rw-rw-r-- 1 t-tsuchiya t-tsuchiya 20984  9月 27 08:51 192.168.34.16_snap02_show_bgp_neighbor.xml
-rwxrwxrwx 1 root       root         146  9月 26 17:34 README
```

PRC形式で出力される。
show bgp neighbor コマンドで取得できる情報はすべて格納されていそうだ。
```
<bgp-information>
<bgp-peer style="detail">
<peer-address>10.160.21.2+34245</peer-address>
<peer-as>65500</peer-as>
<local-address>10.160.21.1+179</local-address>
<local-as>65500</local-as>
<description>dos_genie</description>
<peer-type>Internal</peer-type>
<peer-state>Established</peer-state>
<peer-flags>Sync</peer-flags>
<last-state>OpenConfirm</last-state>
<last-event>RecvKeepAlive</last-event>
<last-error>None</last-error>
<bgp-option-information>
<export-policy>
genie_export1
</export-policy>
<bgp-options>Preference HoldTime LogUpDown AddressFamily Refresh Confed</bgp-options>
<bgp-options2>DropPathAttributes</bgp-options2>
<bgp-options-extended/>
<address-families>inet-unicast inet6-unicast</address-families>
<drop-path-attributes> 128</drop-path-attributes>
<holdtime>90</holdtime>
<preference>170</preference>
</bgp-option-information>
<flap-count>1</flap-count>
<last-flap-event>RecvNotify</last-flap-event>
<bgp-error>
<name>Cease</name>
<send-count>0</send-count>
<receive-count>1</receive-count>
</bgp-error>
<peer-id>172.16.55.28</peer-id>
<local-id>10.160.99.1</local-id>
<active-holdtime>90</active-holdtime>
<keepalive-interval>30</keepalive-interval>
<group-index>2</group-index>
<peer-index>0</peer-index>
<bgp-bfd>
<bfd-configuration-state>disabled</bfd-configuration-state>
<bfd-operational-state>down</bfd-operational-state>
</bgp-bfd>
<peer-restart-nlri-configured>inet-unicast inet6-unicast</peer-restart-nlri-configured>
<nlri-type-peer>inet-unicast inet-vpn-unicast inet6-unicast inet-flow inet-vpn-flow</nlri-type-peer>
<nlri-type-session>inet-unicast inet6-unicast</nlri-type-session>
<peer-no-refresh/>
<peer-stale-route-time-configured>300</peer-stale-route-time-configured>
<peer-no-restart/>
<peer-no-helper/>
<peer-4byte-as-capability-advertised>65500</peer-4byte-as-capability-advertised>
<peer-addpath-not-supported/>
<bgp-rib style="detail">
<name>inet.0</name>
<rib-bit>10001</rib-bit>
<bgp-rib-state>BGP restart is complete</bgp-rib-state>
<send-state>in sync</send-state>
<active-prefix-count>0</active-prefix-count>
<received-prefix-count>0</received-prefix-count>
<accepted-prefix-count>0</accepted-prefix-count>
<suppressed-prefix-count>0</suppressed-prefix-count>
<advertised-prefix-count>4</advertised-prefix-count>
</bgp-rib>
<bgp-rib style="detail">
<name>inet6.0</name>
<rib-bit>20001</rib-bit>
<bgp-rib-state>BGP restart is complete</bgp-rib-state>
<send-state>in sync</send-state>
<active-prefix-count>0</active-prefix-count>
<received-prefix-count>0</received-prefix-count>
<accepted-prefix-count>0</accepted-prefix-count>
<suppressed-prefix-count>0</suppressed-prefix-count>
<advertised-prefix-count>4</advertised-prefix-count>
</bgp-rib>
<last-received>12</last-received>
<last-sent>22</last-sent>
<last-checked>72</last-checked>
<input-messages>10111</input-messages>
<input-updates>0</input-updates>
<input-refreshes>0</input-refreshes>
<input-octets>192167</input-octets>
<output-messages>11085</output-messages>
<output-updates>9</output-updates>
<output-refreshes>0</output-refreshes>
<output-octets>211002</output-octets>
<bgp-output-queue>
<number>0</number>
<count>0</count>
</bgp-output-queue>
<bgp-output-queue>
<number>1</number>
<count>0</count>
</bgp-output-queue>
</bgp-peer>
<bgp-peer style="detail">
<peer-address>10.160.43.2+179</peer-address>
<peer-as>65100</peer-as>
<local-address>10.160.43.1+64868</local-address>
<local-as>65518</local-as>
<description>dos_customer</description>
<peer-type>External</peer-type>
<peer-state>Established</peer-state>
<peer-flags>Sync</peer-flags>
<last-state>OpenConfirm</last-state>
<last-event>RecvKeepAlive</last-event>
<last-error>None</last-error>
<bgp-option-information>
<export-policy>
as65100-export1
</export-policy>
<import-policy>
as65100-import1
</import-policy>
<bgp-options>Preference HoldTime LogUpDown AddressFamily PeerAS Refresh</bgp-options>
<bgp-options2>DropPathAttributes</bgp-options2>
<bgp-options-extended/>
<address-families>inet-unicast</address-families>
<drop-path-attributes> 128</drop-path-attributes>
<holdtime>90</holdtime>
<preference>170</preference>
</bgp-option-information>
<flap-count>0</flap-count>
<peer-id>172.32.160.1</peer-id>
<local-id>10.160.96.1</local-id>
<active-holdtime>90</active-holdtime>
<keepalive-interval>30</keepalive-interval>
<group-index>3</group-index>
<peer-index>0</peer-index>
<bgp-bfd>
<bfd-configuration-state>disabled</bfd-configuration-state>
<bfd-operational-state>down</bfd-operational-state>
</bgp-bfd>

```


```
jsnapy --check snap01 snap02 -f sample/config_check.yml
```

作成されたスナップショット差分をjsnapy --checkコマンドでチェックしてみる。
snap01などのように、作成時に指定したsnapshot名で呼び出しが可能。

実行してみると下記のようにたくさんエラーがでる。
どの部分で発生しているかわからずにたくさん怒られてた。。
（実際はBGPのタイマーやribの変化を全部抽出している）


```
jsnapy --check snap01 snap02 -f sample/config_check.yml


Difference in pre and post snap file
0] <last-received> value different:
    Pre node text: '12'    Post node text: '23'    Parent node: <bgp-peer>
1] <last-sent> value different:
    Pre node text: '22'    Post node text: '7'    Parent node: <bgp-peer>
2] <last-checked> value different:
    Pre node text: '72'    Post node text: '23'    Parent node: <bgp-peer>
3] <input-messages> value different:
    Pre node text: '10111'    Post node text: '10112'    Parent node: <bgp-peer>
4] <input-octets> value different:
    Pre node text: '192167'    Post node text: '192186'    Parent node: <bgp-peer>
5] <output-messages> value different:
    Pre node text: '11085'    Post node text: '11087'    Parent node: <bgp-peer>
6] <output-octets> value different:
    Pre node text: '211002'    Post node text: '211040'    Parent node: <bgp-peer>
7] <last-received> value different:
    Pre node text: '16'    Post node text: '28'    Parent node: <bgp-peer>
8] <last-sent> value different:
    Pre node text: '22'    Post node text: '7'    Parent node: <bgp-peer>
9] <last-checked> value different:
    Pre node text: '38'    Post node text: '79'    Parent node: <bgp-peer>
10] <input-messages> value different:
    Pre node text: '11326'    Post node text: '11327'    Parent node: <bgp-peer>
11] <input-octets> value different:
    Pre node text: '215226'    Post node text: '215245'    Parent node: <bgp-peer>
12] <output-messages> value different:
    Pre node text: '11216'    Post node text: '11218'    Parent node: <bgp-peer>
13] <output-octets> value different:
    Pre node text: '213339'    Post node text: '213377'    Parent node: <bgp-peer>
14] <last-received> value different:
    Pre node text: '6'    Post node text: '21'    Parent node: <bgp-peer>
15] <last-sent> value different:
    Pre node text: '8'    Post node text: '24'    Parent node: <bgp-peer>
16] <last-checked> value different:
    Pre node text: '50'    Post node text: '12'    Parent node: <bgp-peer>
17] <input-messages> value different:
    Pre node text: '2379'    Post node text: '2380'    Parent node: <bgp-peer>
18] <input-octets> value different:
    Pre node text: '45337'    Post node text: '45356'    Parent node: <bgp-peer>
19] <output-messages> value different:
    Pre node text: '2347'    Post node text: '2348'    Parent node: <bgp-peer>
20] <output-octets> value different:
    Pre node text: '44716'    Post node text: '44735'    Parent node: <bgp-peer>
21] <last-received> value different:
    Pre node text: '3'    Post node text: '15'    Parent node: <bgp-peer>
22] <last-sent> value different:
    Pre node text: '23'    Post node text: '10'    Parent node: <bgp-peer>
23] <last-checked> value different:
    Pre node text: '20'    Post node text: '61'    Parent node: <bgp-peer>
24] <input-messages> value different:
    Pre node text: '11210'    Post node text: '11211'    Parent node: <bgp-peer>
25] <input-octets> value different:
    Pre node text: '213188'    Post node text: '213207'    Parent node: <bgp-peer>
26] <output-messages> value different:
    Pre node text: '11203'    Post node text: '11205'    Parent node: <bgp-peer>
27] <output-octets> value different:
    Pre node text: '213271'    Post node text: '213309'    Parent node: <bgp-peer>
28] <last-received> value different:
    Pre node text: '4'    Post node text: '19'    Parent node: <bgp-peer>
29] <last-sent> value different:
    Pre node text: '12'    Post node text: '24'    Parent node: <bgp-peer>
30] <last-checked> value different:
    Pre node text: '24'    Post node text: '65'    Parent node: <bgp-peer>
31] <input-messages> value different:
    Pre node text: '11326'    Post node text: '11327'    Parent node: <bgp-peer>
32] <input-octets> value different:
    Pre node text: '215272'    Post node text: '215291'    Parent node: <bgp-peer>
33] <output-messages> value different:
    Pre node text: '11215'    Post node text: '11216'    Parent node: <bgp-peer>
34] <output-octets> value different:
    Pre node text: '213493'    Post node text: '213512'    Parent node: <bgp-peer>
35] <last-received> value different:
    Pre node text: '15'    Post node text: '1'    Parent node: <bgp-peer>
36] <last-sent> value different:
    Pre node text: '23'    Post node text: '7'    Parent node: <bgp-peer>
37] <last-checked> value different:
    Pre node text: '40'    Post node text: '81'    Parent node: <bgp-peer>
38] <input-messages> value different:
    Pre node text: '2375'    Post node text: '2377'    Parent node: <bgp-peer>
39] <input-octets> value different:
    Pre node text: '45243'    Post node text: '45281'    Parent node: <bgp-peer>
40] <output-messages> value different:
    Pre node text: '2346'    Post node text: '2348'    Parent node: <bgp-peer>
41] <output-octets> value different:
    Pre node text: '44798'    Post node text: '44836'    Parent node: <bgp-peer>
42] <last-received> value different:
    Pre node text: '2'    Post node text: '14'    Parent node: <bgp-peer>
43] <last-sent> value different:
    Pre node text: '22'    Post node text: '7'    Parent node: <bgp-peer>
44] <last-checked> value different:
    Pre node text: '14'    Post node text: '55'    Parent node: <bgp-peer>
45] <input-messages> value different:
    Pre node text: '11207'    Post node text: '11208'    Parent node: <bgp-peer>
46] <input-octets> value different:
    Pre node text: '213041'    Post node text: '213060'    Parent node: <bgp-peer>
47] <output-messages> value different:
    Pre node text: '11201'    Post node text: '11203'    Parent node: <bgp-peer>
48] <output-octets> value different:
    Pre node text: '213332'    Post node text: '213370'    Parent node: <bgp-peer>
------------------------------- Final Result!! -------------------------------
Total No of tests passed: 0
Total No of tests failed: 1
Overall Tests failed!!!
```


今度はjsnapy --diff コマンドで差分を見てみる
実際どの部分に差分が発生しているのか、人間に見やすい形で表示してくれる。
とはいえRPC形式なので見づらいか


```
jsnapy --diff snap01 snap02 -f sample/config_check.yml


************************** Device: 192.168.4.16  **************************
Tests Included: test_bgp_neighbor
************************* Command: show bgp neighbor *************************
/etc/jsnapy/snapshots/192.168.4.16_snap01_show_bgp_neighbor.xml /etc/jsnapy/snapshots/192.168.4.16_snap02_show_bgp_neighbor.xml
<received-prefix-count>0</received-prefix-count>  <received-prefix-count>0</received-prefix-count>
<accepted-prefix-count>0</accepted-prefix-count>  <accepted-prefix-count>0</accepted-prefix-count>
<suppressed-prefix-count>0</suppressed-prefix-co  <suppressed-prefix-count>0</suppressed-prefix-co
unt>                                              unt>
<advertised-prefix-count>4</advertised-prefix-co  <advertised-prefix-count>4</advertised-prefix-co
unt>                                              unt>
</bgp-rib>                                        </bgp-rib>
<last-received>12</last-received>                 <last-received>23</last-received>
<last-sent>22</last-sent>                         <last-sent>7</last-sent>
<last-checked>72</last-checked>                   <last-checked>23</last-checked>
<input-messages>10111</input-messages>            <input-messages>10112</input-messages>
<input-updates>0</input-updates>                  <input-updates>0</input-updates>
<input-refreshes>0</input-refreshes>              <input-refreshes>0</input-refreshes>
<input-octets>192167</input-octets>               <input-octets>192186</input-octets>
<output-messages>11085</output-messages>          <output-messages>11087</output-messages>
<output-updates>9</output-updates>                <output-updates>9</output-updates>
<output-refreshes>0</output-refreshes>            <output-refreshes>0</output-refreshes>
<output-octets>211002</output-octets>             <output-octets>211040</output-octets>
<bgp-output-queue>                                <bgp-output-queue>
<number>0</number>                                <number>0</number>
<count>0</count>                                  <count>0</count>
</bgp-output-queue>                               </bgp-output-queue>
<bgp-output-queue>                                <bgp-output-queue>
```


snapshot出力ファイルの設定を変更したい
- /etc/jsnapy/snapshots/ の出力先ディレクトリを変更したい(作業順や対象がごちゃごちゃになる)
- ファイル名に日付,日時を入れたい 
- snap名とファイル名が紐付いてないのはどう定義するのがベストか考える必要がある

時間差でupしてくるようなものについてはどうチェックするか？
10秒単位で見に行く？
