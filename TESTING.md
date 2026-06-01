# SIPp SHA-256 Digest Authentication 試験手順

## 事前準備

### 依存パッケージのインストール

```bash
# Ubuntu/Debian
sudo apt-get install -y cmake libssl-dev libncurses-dev libpcap-dev
```

### サブモジュールの初期化

```bash
git submodule update --init
```

### ビルド

```bash
cmake . -DUSE_SSL=ON -DUSE_SCTP=OFF -DUSE_PCAP=OFF
make -j$(nproc)
```

---

## 試験1: REGISTER + SHA-256 認証（簡易版）

UAS（サーバ）と UAC（クライアント）を起動し、REGISTER の Digest 認証で SHA-256 が選択されることを確認します。

### 手順

**ターミナル1（UAS）:**

```bash
./sipp -sf test_server.xml -i 127.0.0.1 -p 5060 \
  -trace_msg -message_file server_sip.log \
  -default_behaviors none -aa -m 1
```

**ターミナル2（UAC）:**

```bash
./sipp -sf test_client.xml -i 127.0.0.1 -p 5061 \
  127.0.0.1:5060 \
  -trace_msg -message_file client_sip.log \
  -default_behaviors none -r 1 -m 1 -l 1
```

### 期待されるシーケンス

```
REGISTER → 401 (MD5 + SHA-256) → REGISTER (Authorization: algorithm=SHA-256) → 200 OK
```

### 確認ポイント

- `client_sip.log` の Authorization ヘッダに `algorithm=SHA-256` が含まれていること
- `response="<64桁の16進数>"` であること（SHA-256 は 64 hex = 32bytes）
- 認証に使用された nonce が SHA-256 チャレンジのものであること

---

## 試験2: REGISTER + INVITE + 5秒通話 + BYE（フル版）

UAS と UAC を起動し、REGISTER と INVITE の両方で SHA-256 認証が行われ、セッション確立→5秒通話→切断までを確認します。

### 手順

**ターミナル1（UAS）:**

```bash
./sipp -sf test_server_full.xml -i 127.0.0.1 -p 5060 \
  -trace_msg -message_file server_full_sip.log \
  -default_behaviors none -aa -m 1
```

**ターミナル2（UAC）:**

```bash
./sipp -sf test_client_full.xml -i 127.0.0.1 -p 5061 \
  127.0.0.1:5060 \
  -trace_msg -message_file client_full_sip.log \
  -default_behaviors none -r 1 -m 1 -l 1
```

### 期待されるシーケンス

```
REGISTER → 401 (MD5+SHA-256)
  → REGISTER (Authorization: SHA-256) → 200 OK

INVITE → 407 (Proxy-Authenticate: MD5+SHA-256)
  → INVITE (Proxy-Authorization: SHA-256) → 200 OK (with SDP)

ACK → [5秒 pause] → BYE → 200 OK
```

### 確認ポイント

- REGISTER の 401 応答: Authorization ヘッダ（`algorithm=SHA-256`）
- INVITE の 407 応答: Proxy-Authorization ヘッダ（`algorithm=SHA-256`）
- レスポンスハッシュが 64 hex 桁であること
- ACK, BYE の CSeq が正しいこと
- Successful call = 1, Failed call = 0 となること

---

## オプションの説明

| オプション | 説明 |
|-----------|------|
| `-sf <file>` | シナリオファイルを指定 |
| `-i <ip>` | ローカル IP アドレス |
| `-p <port>` | ローカルポート（UAS は待受ポート） |
| `-trace_msg` | SIP メッセージをログファイルに出力 |
| `-message_file <file>` | メッセージログの出力先ファイル |
| `-default_behaviors none` | デフォルト動作（自動応答など）を無効化 |
| `-aa` | 自動 200 OK 応答を有効化 |
| `-m <n>` | 最大コール数 |
| `-r <rate>` | コールレート（cps） |
| `-l <n>` | 最大同時コール数 |
| `-timeout <sec>` | タイムアウト |
