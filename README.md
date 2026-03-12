# わかった気になる Media over QUIC

## 要約

- MoQ は QUIC / MoQT / MSF の 3 層からなるプロトコル群
- MoQ は GoP を QUIC Stream にマッピングすることで、QUIC の性質をうまく利用している
- MoQT のデータモデルは Track > Group > Subgroup > Object の階層構造
- Relay を介した Pub/Sub により、fan-out でスケーラブルな配信を実現する
- Pull 型フローは、セッション確立 → Namespace の発見 → Track の購読 → データ転送の順に進む

## この資料は何か

ソフトウェアエンジニア向けに、MoQ (Media over QUIC) の概要を説明する資料。

この資料の目的は、読者が下記を理解すること。

- MoQ のプロトコルスタックの全体像 (QUIC / MoQT / MSF)
- MoQT のデータモデル (Track / Group / Subgroup / Object)
- MoQT の Pub/Sub の仕組み (セッション確立からデータ転送まで)

> [!NOTE]
> この資料では下記のテーマは扱わない。
> - MSF / LOC / CMSF の詳細
> - QUIC / WebTransport の詳細
> - WebRTC や HLS/DASH との比較

## MoQ とは何か

MoQ (Media over QUIC) とは、QUIC を上手く使ってメディア (映像・音声) を配信するためのプロトコル群である。

```
┌──────────────────────────────────────────┐
│    MoQT Streaming Format (MSF)           │ メディア形式
├──────────────────────────────────────────┤
│    Media over QUIC Transport (MoQT)      │ メディア配信プロトコル
├──────────────────┬───────────────────────┤ ┐
│                  │      WebTransport     │ │
│                  ├───────────────────────┤ │ 通信プロトコル
│                  │        HTTP/3         │ │
│                  └───────────────────────┤ │
│                   QUIC                   │ │
└──────────────────────────────────────────┘ ┘
```

QUIC は、TCP と同等の機能を持ちつつ HoL blocking を解消したプロトコルである。

- TCP と同様に信頼性のある通信 (再送・順序保証・輻輳制御・フロー制御) ができる
- 1 つの接続の中に複数の独立した Stream を持てる

> [!IMPORTANT]
> ある Stream のパケットロスが他の Stream をブロック (HoL blocking) しない。ここが TCP とは違う。
- ブラウザからは WebTransport (over HTTP/3) 経由、ネイティブでは QUIC を直接使う

MoQT (Media over QUIC Transport) は、QUIC の上に乗るメディア配信プロトコル。主に 2 つを定義している。

1. 映像・音声データをどう構造化するか (データモデル: Track / Group / Subgroup / Object)
2. 中継サーバーである Relay を介してスケーラブルに配信するための Pub/Sub の仕組み

MSF (MoQT Streaming Format) は、MoQT の上に乗るメディア形式の仕様。主に 2 つを定義している。

1. 配信中の映像・音声 Track を記述するカタログと、その通知の仕組み
2. 映像・音声をどう切り分けて MoQT に乗せるかのルール

## MoQ は QUIC をどう上手く使うのか

プロトコルの詳細に入る前に、MoQ の概要を把握してほしい。
MoQ が映像を QUIC Stream にどう乗せるかを説明する。


映像を送るには、映像データをどう分割して送るかが問題になる。
MoQ はこの分割単位として GoP を使い、各 GoP を 1 つの QUIC Stream にマッピングする。以下で詳しく見ていく。

映像は通常、フレーム (= 1 枚の画像) の連続として表現される。
フレームをそのまま送るとデータ量が大きいので、映像コーデックを使って圧縮する。
映像コーデックでは一般に、次の種類のフレームがある。

- I-frame (キーフレーム): 他のフレームに依存せず、単体でデコードできるフレーム
- P-frame: 直前のフレームとの差分だけを持つフレーム。単体ではデコードできない

```
  GoP 1                          GoP 2
  ┌───┬───┬───┬───┬───┐          ┌───┬───┬───┬───┬───┐
  │ I │ P │ P │ P │ P │  ...     │ I │ P │ P │ P │ P │  ...
  └───┴───┴───┴───┴───┘          └───┴───┴───┴───┴───┘
```

I-frame を先頭に、後続の P-frame をまとめた単位を GoP (Group of Pictures) と呼ぶ。
GoP 内のフレームは先頭の I-frame から順にデコードする必要がある。
例えば、30fps でキーフレーム間隔が 1 秒の映像なら、1 つの GoP は I-frame 1 枚 + P-frame 29 枚 = 30 フレームになる。

MoQ では、この GoP を 1 つの QUIC Stream に乗せる。

```
  GoP 1  ──→  QUIC Stream 1: [I][P][P][P][P]...
  GoP 2  ──→  QUIC Stream 2: [I][P][P][P][P]...
  GoP 3  ──→  QUIC Stream 3: [I][P][P][P][P]...
```

> [!TIP]
> このマッピングにより、QUIC Stream の性質を以下のように利用できる。
> - GoP 内のフレームは順序通りに届く必要がある → Stream は順序保証してくれる
> - GoP ごとに独立した Stream になる → ある GoP のパケットロスが、別の GoP の配信をブロックしない
> - 古くなった GoP を途中で破棄できる → Stream 単位でキャンセルできる。リアルタイム配信で遅延が蓄積した場合に、古い GoP を破棄できる

## Media over QUIC Transport (MoQT)

MoQT は主に 2 つを定義している。

1. 映像・音声データをどう構造化するか (データモデル)
2. Relay を介した Pub/Sub の仕組み

### データモデル

MoQT では、映像や音声のデータを Track > Group > Subgroup > Object の階層構造で表現する。
> [!NOTE]
> MoQT 自体は、各階層に何をマッピングするかを規定していない。ここでは、わかりやすさのために、MSF + LOC に沿って説明する。

```
// Track > Group > Subgroup > Object の階層構造
Track: "Alice の映像"
├── Group 0 (GoP)
│   └── Subgroup 0
│       ├── Object 0 (I-frame)
│       ├── Object 1 (P-frame)
│       ├── Object 2 (P-frame)
│       └── ...
├── Group 1 (GoP)
│   └── Subgroup 0
│       ├── Object 0 (I-frame)
│       ├── Object 1 (P-frame)
│       ├── Object 2 (P-frame)
│       └── ...
...
```

Track は、メディアの論理的な単位。
例えば「Alice の映像」や「Alice の音声」がそれぞれ 1 つの Track に対応する。
Subscriber は、この Track 単位でデータを購読する。

Group は、Track を区切る単位で、Track 内の join point (途中参加ポイント) になる。
途中から購読した Subscriber は、Group の境界から受信を開始できる。映像の場合、GoP が Group に相当する。

Subgroup は、Group 内の Object を細分化する単位。
1 Subgroup が 1 QUIC Stream にマッピングされる。
上の図では、シンプルな構成 (1 Group = 1 Subgroup) にしている。

Object は、MoQT における最小のデータ単位。
映像であれば、1 フレームが 1 Object に対応する。

#### より実践的な例: SVC を使う映像

シンプルな映像なら 1 Group に 1 Subgroup で十分である。しかし、SVC (Scalable Video Coding) のようにレイヤーが分かれる場合は、レイヤーごとに Subgroup を分ける。
Subgroup が分かれると QUIC Stream も分かれる。
そのため、帯域不足時に Enhancement Layer の Stream だけを破棄して、Base Layer だけで再生を続ける、といった制御ができる。

```
Track: "Alice の映像 (SVC)"
├── Group 0 (GoP)
│   ├── Subgroup 0 (Base Layer)
│   │   ├── Object 0 (I-frame)
│   │   ├── Object 1 (P-frame)
│   │   ├── Object 2 (P-frame)
│   │   └── ...
│   └── Subgroup 1 (Enhancement Layer)
│       ├── Object 0
│       ├── Object 1
│       ├── Object 2
│       └── ...
├── Group 1 (GoP)
│   ├── Subgroup 0 (Base Layer)
│   │   ├── Object 0 (I-frame)
│   │   ├── Object 1 (P-frame)
│   │   ├── Object 2 (P-frame)
│   │   └── ...
│   └── Subgroup 1 (Enhancement Layer)
│       ├── Object 0
│       ├── Object 1
│       ├── Object 2
│       └── ...
...
```

### Relay を介した Pub/Sub の仕組み

次に、データの流れを説明する。

MoQT では、Publisher (送信側) と Subscriber (受信側) が Relay (中継サーバー) を介して Pub/Sub でデータをやりとりする。

```
Publisher ──→ Relay ──→ Subscriber
```

Relay は、Publisher から見ると Subscriber、Subscriber から見ると Publisher として振る舞う。
この性質により、Relay を多段に配置して CDN のようなツリー構造を構成できる。

```
                    ┌──→ Subscriber
              ┌── Relay
              │     └──→ Subscriber
Publisher ── Relay
              │     ┌──→ Subscriber
              └── Relay
                    └──→ Subscriber
```

各 Relay が Object を下流に転送 (fan-out) することで、スケーラブルな配信を実現する。

> [!NOTE]
> MoQT では Track の購読開始方法に 2 種類ある。
> 1. Subscriber 側から始める方法 (Pull 型と呼ぶことにする)
> 2. Publisher 側から始める方法 (Push 型と呼ぶことにする)
>
> この資料では、わかりやすさのために Pull 型のフローのみを説明する。

### Pull 型フローの全体像

ここでは、データが流れるまでの一連のフロー (セッション確立 → Namespace の発見 → Track の購読 → データ転送) を説明する。

以下のシーケンス図が全体像である。
各ステップを順に説明する。

```
Publisher                      Relay                      Subscriber
    |                            |                            |
    | (1) セッション確立           |                            |
    |                            |                            |
    |==== QUIC/WebTransport ====>|                            |
    |<--------- SETUP ---------->|                            |
    |                            |<==== QUIC/WebTransport ====|
    |                            |<---------- SETUP --------->|
    |                            |                            |
    | (2) Namespace の発見        |                            |
    |                            |                            |
    |---- PUBLISH_NAMESPACE ---->|                            |
    |<------- REQUEST_OK --------|                            |
    |                            |<--- SUBSCRIBE_NAMESPACE ---|
    |                            |-------- REQUEST_OK ------->|
    |                            |-------- NAMESPACE -------->|
    |                            |                            |
    | (3) Track の購読とデータ転送  |                            |
    |                            |                            |
    |                            |<-------- SUBSCRIBE --------|
    |<-------- SUBSCRIBE --------|                            |
    |------ SUBSCRIBE_OK ------->|                            |
    |                            |------ SUBSCRIBE_OK ------->|
    |                            |                            |
    |========= Objects =========>|                            |
    |                            |========= Objects =========>|
    |                            |                            |
```

#### セッション確立

```
Publisher                      Relay                      Subscriber
    |                            |                            |
    |==== QUIC/WebTransport ====>|                            |
    |<--------- SETUP ---------->|                            |
    |                            |<==== QUIC/WebTransport ====|
    |                            |<---------- SETUP --------->|
    |                            |                            |
```

Publisher と Subscriber はそれぞれ Relay に対して QUIC または WebTransport の接続を確立する。
接続確立後、QUIC Stream 上で `SETUP` メッセージを交換する。
`SETUP` で通信に必要な設定 (拡張機能など) を交渉する。
これにより、MoQT のメッセージをやりとりする準備が整う。

#### Namespace の発見

Track は、Track Namespace と Track Name の組み合わせで一意に識別される。
(e.g, Namespace = "live/sports", Track Name = "video")

Subscriber が Track を購読するには、まずどんな Track が存在するかを知る必要がある。
そのために `PUBLISH_NAMESPACE` と `SUBSCRIBE_NAMESPACE` メッセージを使う。
<!-- 本当は? 別の方法で Namespace を知って、直接 Track を購読することも可能。この話をどうするか。 -->

```
Publisher                      Relay                      Subscriber
    |                            |                            |
    |---- PUBLISH_NAMESPACE ---->|                            |
    |    "live/sports"           |                            |
    |<------- REQUEST_OK --------|                            |
    |                            |<--- SUBSCRIBE_NAMESPACE ---|
    |                            |    prefix="live"           |
    |                            |-------- REQUEST_OK ------->|
    |                            |-------- NAMESPACE -------->|
    |                            |    "live/sports"           |
    |                            |                            |
```

例えば、Publisher が "live/sports" という Namespace の下に "video" と "audio" の Track を持っているとする。

1. Publisher が `PUBLISH_NAMESPACE` で "live/sports" を Relay に広告する
2. Subscriber が `SUBSCRIBE_NAMESPACE` で prefix "live" に一致する Namespace の通知を Relay に要求する
3. Relay が `NAMESPACE` で「"live/sports" がある」と Subscriber に通知する

これにより、Subscriber は利用可能な Namespace を把握できる。

#### Track の購読とデータ転送

```
Publisher                      Relay                      Subscriber
    |                            |                            |
    |                            |<-------- SUBSCRIBE --------|
    |                            |  "live/sports" + "video"   |
    |<-------- SUBSCRIBE --------|                            |
    |  "live/sports" + "video"   |                            |
    |------ SUBSCRIBE_OK ------->|                            |
    |                            |------ SUBSCRIBE_OK ------->|
    |                            |                            |
    |========= Objects =========>|                            |
    |                            |========= Objects =========>|
    |                            |                            |
```

Namespace を知った Subscriber は、Track を指定して `SUBSCRIBE` する。
例えば、"live/sports" の "video" を購読する。

Relay がまだその Track を購読していなければ、上流の Publisher に `SUBSCRIBE` を転送する。
購読済みであれば、Relay はすぐに `SUBSCRIBE_OK` を返せる。

Subscriber が `SUBSCRIBE_OK` を受け取ると、Publisher から Relay を経由して Object の転送が始まる。

### まとめ

この資料では、MoQ の全体像と MoQT の基本を説明した。

- MoQ は QUIC / MoQT / MSF の 3 層からなるプロトコル群
- MoQ は GoP を QUIC Stream にマッピングすることで、QUIC の性質をうまく利用している
- MoQT のデータモデルは Track > Group > Subgroup > Object の階層構造
- Relay を介した Pub/Sub により、fan-out でスケーラブルな配信を実現する
- Pull 型フローは、セッション確立 → Namespace の発見 → Track の購読 → データ転送の順に進む

### 話していないテーマ

この資料では MoQT の基本概念と Pull 型フローに絞って説明した。
MoQ には他にも多くのテーマがある。

- Push 型フロー — `PUBLISH` メッセージにより Publisher 側から購読を開始する仕組み
- `FETCH` — 過去のデータを取得する仕組み。途中参加時に過去の Group を取得するなど
- 優先制御 — Subscriber Priority / Publisher Priority / Group Order による送信順序の制御
- 輻輳制御・フロー制御 — バッファブロート対策やスループット制御
- Stream の種類と使い分け — 制御メッセージ用の双方向 Stream とデータ用の単方向 Stream
- Relay の詳細 — キャッシュ、複数 Publisher の扱い、認可、Relay 切り替え (switchover)
- 認証 — Authorization Token によるセッション・Track レベルの認証
- セッション管理 — `GOAWAY` によるセッション移行、エラーコード、タイムアウト
- MSF / LOC / CMSF — カタログ、タイムライン、コンテナフォーマットの詳細
