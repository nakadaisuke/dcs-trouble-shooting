# DCSノードのプロビジョニングとトラブルシューティング

DCSノードはベアメタル、仮想マシン、クラウドなど様々な場所に構築が可能です。ノードのプロビジョニングは大きく2つの方法があります。

1. ベアメタルか仮想マシンに初期イメージをインストールし、初期パラメータを入力してプロビジョニング
2. クラウド上の仮想マシン（インスタンス）をDCS Consoleからコンフィグを作成してプロビジョニング

これら2つの方法でのプロビジョニングのトラブルシューティングを行います。

## ベアメタルや仮想マシンのトラブルシューティング

ベアメタルや仮想マシンに初期イメージをインストールしたあと、各種パラメータを入力し、ConsoleでRegistrationを行うとプロビジョニングが開始されます。
ここでパラメータなどの入力が間違っているとプロビジョニングに失敗する場合があります。

### Console/REへの到達性がない

ルーティングとDNSの設定を確認します。特にProxyを経由している場合にConsole/REへの到達性がないためRegistrationの画面にノードがでてこない。というトラブルがあります。

<b>トラブルシューティング</b>

1. インターネット接続とDNSの確認 

Local CLIもしくはLocal GUIにログインして`console.ves.volterra.io`に到達性があるか確認します。

Local CLIの場合

``` bash
>>> ping console.ves.volterra.io

--- 18.190.3.39 ping statistics ---
5 packets transmitted, 0 packets received, 100% packet loss
round-trip min/avg/max/stddev = 0s/0s/0s/0s
```

Local GUIの場合

`Tools` -> `Host Ping`

![host-ping1](./pics/host-ping-1.png)

2. DCSノードのProxy設定の確認

ノードがProxy配下にある場合、Proxyの設定が必要です。

Local CLIの場合 `Configure-http-proxy`で設定できます。

```
>>> configure-http-proxy
? Set HTTP_PROXY (e.g. http://username:password@10.0.0.1:3129) or keep empty for current proxy http://test.proxy:user:password@10.0.0.1:3129
Node is not registered, going to configure local HTTP Proxy
proxy:
  httpProxy: http://test.proxy:user:password@10.0.0.1:3129
  httpsProxy: http://test.proxy:user:password@10.0.0.1:3129
? Confirm HTTP proxy configuration? Yes
```

確認は`get-config`で行います。`Vpm`にProxyの設定が入ります。パスワードはマスクされます。

```
Vpm:
  CertifiedHardware: vmware-regular-nic-voltmesh
  ClusterName: CHANGE-ME
  ClusterType: ce
  Config: /etc/vpm/config.yaml
  ContainerRuntime: docker
  MauriceEndpoint: https://register.ves.volterra.io
  MauricePrivateEndpoint: https://register-tls.ves.volterra.io
  PrivateNIC: eth0
  Proxy:
    httpProxy: http://*****:*****@10.0.0.1:3129
    httpsProxy: http://*****:*****@10.0.0.1:3129
```

Local GUIの場合は`Edit Configuration`のProxy configで設定します。

![proxy-config-1](./pics/proxy-config-1.png)

3. Proxyサーバの設定確認

上記2点を確認してもRegistrationにノードが表示されない場合、次の確認をしてください。

### Tokenの確認

Tokenが正しく設定していないとDCSシステムがどのテナントのノードが判断できないためRegistrationに表示させることができません。
Tokenを再度正しく入力してください。

### Registration時のエラー

Registrationしたあとに、以下のような表示されます。これは緯度経度（Latitude/Longtitude)が正しく入力されていないために発生します。正しい数値を入力してください。

```
Http failure response: 500 Server Error
There was a server error in processing request
Failed updating registraion via approval: Failed passport validation: Missing latitude or longitude (site name ts-node-1): lat 0.000000, lon 0.000000<2022-03-15 02:31:34.671253399 +0000 UTC m=+1768530.730375661

https://volt-field.console.ves.volterra.io/api/register/namespaces/system/registration/r-5f28e154-61e4-473b-b1ed-82d3c50923eb/approve
```

また、Site名が重複していると以下のようなエラーが発生します。正しいSite名を入力してください。
ハードウェア入れ替えなどで同一サイト名でプロビジョニングを行いたい場合、 `Registrations` -> `Other Registrations`から該当のSiteをDeccomissionしてからサイドRegistrationすることで、このエラーを回避できます。

```
Http failure response: 500 Server Error
There was a server error in processing request
unable to approve registration r-5f28e154-61e4-473b-b1ed-82d3c50923eb for cluster dupulicate-node-1 for hostname master-0, duplicite registration already present<2022-03-15 02:33:07.541511418 +0000 UTC m=+1768623.600633673

https://volt-field.console.ves.volterra.io/api/register/namespaces/system/registration/r-5f28e154-61e4-473b-b1ed-82d3c50923eb/approve
```

### Provisioning時のエラー

Provisioing時に Site Admin StateがFailedと表示される場合があります。
Failedと表示された場合、Siteが正常に上がっていないため、ノードの確認が必要です。

#### Statusの確認

正常な状態では以下の用にすべてのコンポーネントがReadyの状態になります。
ベアメタルでDPDK対応でないNICを使用していたり、KVMでHugepageが有効でない場合には`argo`が `Not Ready`のステータスになることがあります。

また、メモリが十分でない場合、いくつかのコンポーネントがNot Readyの場合があります。この場合はノードがマニュアルを確認し、Minimum requirementsを満たすように調整してください。

まれにDHCPサーバの仕様などでプロビジョニング中にIPアドレスの割当が変わってしまう場合があります。
この場合プロビジョニング失敗する場合があるため、DHCPサーバのLease timeが極端に短かい場合は十分なLease timeへの変更、同一MACアドレスからのDHCP Requestに対しては同一IPアドレスを払い出すように設定してください。

```
>>> volterra-status
│────────────│──────────│────────│─────────│
│ NAME (15)  │ NODE     │ STATUS │ MESSAGE │
│────────────│──────────│────────│─────────│
│ argo       │ master-0 │ Ready  │         │
│ bdbewaf    │ master-0 │ Ready  │         │
│ envoy      │ master-0 │ Ready  │         │
│ frr        │ master-0 │ Ready  │         │
│ ike        │ master-0 │ Ready  │         │
│ keepalived │ master-0 │ Ready  │         │
│ obelix     │ master-0 │ Ready  │         │
│ openvpn    │ master-0 │ Ready  │         │
│ opera      │ master-0 │ Ready  │         │
│ pmtud      │ master-0 │ Ready  │         │
│ ver        │ master-0 │ Ready  │         │
│ webroot    │ master-0 │ Ready  │         │
│ dnsmasq    │ master-0 │ Ready  │         │
│ gubernator │ master-0 │ Ready  │         │
│ ver-cfg    │ master-0 │ Ready  │         │
│────────────│──────────│────────│─────────│
````

Local UIの場合、`Services`で状態が確認できます。

![local-ui-service-1](./pics/local-ui-service-1.png)

Not Readyのコンポーネントはリスタートすることで解消する場合があります。Local CLIからは
`soft-restart XXX` で実行できます。

```
>>> soft-restart ver
Service ver restart scheduled
```

Local GUIからは `Tools` -> `Soft Restart`から　Networking Serviceをリスタートしてください。

![soft-restart-gui-1](./pics/softrestart-gui-1.png)

#### Tunnelの確認

各種ステータスが正常の場合でもFaildやHealth scoreが`0`の場合があります。
この場合はTunnel interfaceが正常に確立されていない場合があります。
ルーティングやFirewall/Proxyなどの設定を見直し、正常に通信できるか確認してください。
必要な宛先リストはF5のマニュアルに記載されています。

#### サポート問い合わせ

上記を実施しても問題が解決しない場合、Debuginfoを取得してサポートに問い合わせしてください。