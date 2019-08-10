# WireGuardを使ってみた

author
:   Kazuhiro NISHIYAMA

content-source
:   LILO&東海道らぐオフラインミーティング

date
:   2019/08/10

allotted-time
:   15m

theme
:   lightning-simple

# WireGuard とは?

- 暗号化などは最近のアルゴリズムを採用
  - ChaCha20, Curve25519, BLAKE2, SipHash24, HKDF
- セキュアな設計
- セキュリティ監査後にプロトコルは変更される可能性あり
- 別実装も存在

# インストール

- https://www.wireguard.com/install/
- Debian (buster) なら unstable から
- Ubuntu なら ppa:wireguard/wireguard から
- その他ディストリビューションにも対応
- Windows, macOS, Android, iOS にも対応

# 懸念点

- dkms を使っているので Secure Boot 環境だと動かないかも?
- Userspace 実装の https://github.com/cloudflare/boringtun なら動く? (未確認)

# ネットワークのイメージ

- WireGuard をハブとして繋げる感じ
- 例: 10.192.122.0/24 のネットワークをスター型で接続
- 例: 10.192.124.1/32 と 10.192.124.2/32 を P2P 接続

# 基本的な使い方

- wg コマンド : 鍵の設定など
- wg-quick コマンド : IP アドレスの設定などを含むラッパーコマンド

ip コマンドで直接設定する場合は <https://www.wireguard.com/quickstart/> 参照

# wg(8) Interface

- PrivateKey: 自分側の秘密鍵
- ListenPort: 待ち受けポート (固定しない場合は不要)
- FwMark (iptables などと組み合わせるためのもの)
  - a 32-bit fwmark for outgoing packets. If set to 0 or "off", this option is disabled. May be specified in hexadecimal by prepending "0x". Optional.

# wg(8) Peer

- PublicKey
- PresharedKey
- AllowedIPs
  - ',' 区切りの IP/CIDR
  - 複数回 OK

# wg(8) Peer

- Endpoint
  - IP アドレスまたはホスト名 + ':' + ポート番号
  - IPv6 なら `Endpoint = [XXXX:YYYY:ZZZZ::1]:51820` のように `[]` でくくる
- PersistentKeepalive
  - NAT 向け

# wg-quick(8)

- wg と ip のラッパー

# wg-quick(8) Interface

- SaveConfig : 設定の自動保存
- Address : 自分側 IP アドレス
- DNS : 相手に使わせる DNS 設定?
- MTU
- Table
- PreUp, PostUp, PreDown, PostDown
  - iptables の NAT 設定などに使う

# 使用例

- `umask 077` して `wg genkey | tee privatekey | wg pubkey | tee publickey` で鍵ペア作成
- `/etc/wireguard/wg0.conf` 作成
- `systemctl enable wg-quick@wg0 --now`

# VPS 側 Interface 設定例

Address には `/24` のように範囲を設定
(PostUp, PostDown は実際には1行)

    [Interface]
    Address = 10.10.0.1/24
    Address = fd86:ea04:1111::1/64
    SaveConfig = true
    PostUp = iptables -A FORWARD -i wg0 -j ACCEPT;
      iptables -t nat -A POSTROUTING -o en+ -j MASQUERADE;
      ip6tables -A FORWARD -i wg0 -j ACCEPT;
      ip6tables -t nat -A POSTROUTING -o en+ -j MASQUERADE
    PostDown = iptables -D FORWARD -i wg0 -j ACCEPT;
      iptables -t nat -D POSTROUTING -o en+ -j MASQUERADE;
      ip6tables -D FORWARD -i wg0 -j ACCEPT;
      ip6tables -t nat -D POSTROUTING -o en+ -j MASQUERADE
    ListenPort = 51820
    PrivateKey = XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX=

#  VPS 側 Peer 設定例

AllowIPs には接続先での Address と同じものを指定

    [Peer]
    PublicKey = ugpA/M4UKHyPX9ymXI2ntHJ+uHbdUpK6duGnjj9QGnI=
    AllowedIPs = 10.10.0.2/32, fd86:ea04:1111::2/128

#  VPS 側 Peer 設定例 (2)

外から接続できる Peer なら Endpoint も設定するとどちらからでも接続可能

    [Peer]
    PublicKey = HXuu2SFfqOSN2iifw3Mh7J6rDRMjIXAR3wQvkyYrKyA=
    AllowedIPs = 10.10.0.3/32, fd86:ea04:1111::3/128
    Endpoint = XXX.YYY.ZZZ.123:46242

# 端末側 Interface 設定例

`/32` で特定のアドレスのみを設定

    [Interface]
    PrivateKey = XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX=
    Address = 10.10.0.2/32, fd86:ea04:1111::2/128

# 端末側 Peer 設定例

Peer にルーティングさせたいアドレスを AllowedIPs に設定

    [Peer]
    PublicKey = 4TBu1wBKSvH/Bl14hyxSTq1AEx3mOTxiR5e7Vpd13ng=
    AllowedIPs = ::/0, 10.10.0.0/24
    Endpoint = XXX.YYY.ZZZ.111:51820

# 疑問点

以下の設定で 10.10.0.2 へのパケットがローカルの wg0 直接ではなくサーバー経由になってしまう
(ping で確認すると応答速度が往復している時間かかっている)

`[Interface]` で `Address = 10.10.0.2/32`
`[Peer]` で `AllowedIPs = 10.10.0.0/24`

# トラブルシューティング

- `ip a` で IP アドレス確認
- `wg` コマンドで送受信の量を確認
- 送信できていなければ送信側の設定確認
- 送信しているのに受信していなければ途中の firewall などを確認

# 参考

- <https://wiki.archlinux.jp/index.php/WireGuard>
- 作って理解するWireGuard <https://www.youtube.com/watch?v=grDEBt7oQho>
- The Unofficial Wireguard Documentation <https://github.com/pirate/wireguard-docs>
