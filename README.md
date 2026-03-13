# わかった気になる Media over QUIC

## この資料は何か

ソフトウェアエンジニア向けに、MoQ (Media over QUIC) の概要を説明した資料。

この資料の目的は、読者が MoQ (特に Media over QUIC Transport) を概要レベルで理解できるようになること。

具体的には、
1. 映像・音声データをどのようにモデリングするか
2. それをどのように配信するか
の 2 つ を特に理解してもらいたい。

> [!NOTE]
> この資料では下記のテーマは扱わない。
> - MoQ が生まれた背景や既存技術 (WebRTC, HLS/DASH) との比較
> - MoQ に関連する仕様 (MSF, LOC, CMSF, QUIC, WebTransport) の詳細

## 要約

時間がない人向けに、この資料のポイントをまとめる。

- MoQ は QUIC (トランスポート), MoQT (データモデル + Pub/Sub), MSF (メディアの記述 + MoQT への乗せ方) で構成されるプロトコル群
- MoQ は GoP を QUIC Stream にマッピングすることで、QUIC の性質をうまく利用する
- MoQT のデータモデルは Track > Group > Subgroup > Object の階層構造
- Relay を介した Pub/Sub により、fan-out でスケーラブルな配信を実現する
- Pull 型フローは、セッション確立 → Namespace の発見 → Track の購読 → データ転送の順に進む

## MoQ とは何か

MoQ (Media over QUIC) とは、QUIC を上手く使ってメディア (映像・音声) を配信するためのプロトコル群である。

MoQ のプロトコルスタックは、シンプルに捉えると以下の 3 層で構成されている。

```
┌──────────────────────────────────────────┐
│    MoQT Streaming Format (MSF)           │ メディアの記述方法 + MoQT への乗せ方
├──────────────────────────────────────────┤
│    Media over QUIC Transport (MoQT)      │ データモデル + Pub/Sub
├──────────────────┬───────────────────────┤ ┐
│                  │      WebTransport     │ │
│                  ├───────────────────────┤ │
│                  │        HTTP/3         │ │ トランスポート
│                  └───────────────────────┤ │
│                   QUIC                   │ │
└──────────────────────────────────────────┘ ┘
```

> [!NOTE]
> ブラウザからは WebTransport over HTTP/3 経由で QUIC を使う。

それぞれを順に説明する。

### QUIC

QUIC は、TCP のような信頼性のある通信と、独立したストリームの多重化を提供するトランスポートプロトコルである。

- TCP と同様に、信頼性のある通信 (再送・順序保証・輻輳制御・フロー制御) ができる
- TCP と異なり、1 つの接続の中に複数の独立した Stream を持てる (Stream 多重化)

MoQ は Stream の独立性を活かして設計されている。
この性質は理解しておいてほしいので、TCP 上での Stream 多重化と QUIC の Stream 多重化の違いを図で説明する。

TCP 上で複数の Stream を多重化する (HTTP/2 がこれにあたる) と、TCP は全データを 1 本のバイトストリームとして扱うため、1 つのパケットロスが全 Stream をブロックする (Head-of-Line blocking)。

```
  // TCP バイトストリーム (受信側)
  [A][B][A][C][✕][B][C][A]...
                ↑ ロス → 再送されるまで後続のデータがブロックされる

  // HTTP/2 Stream (アプリケーションから見た状態)
  Stream A: ■ ■ □ □ □   ← ブロック
  Stream B: ■ □ □ □ □   ← ブロック
  Stream C: ■ □ □ □ □   ← ブロック

  ■ 受信済み  □ ブロック中  ✕ ロスしたパケット
```

一方、QUIC は UDP ベースのプロトコルで、Stream ごとに独立したバイトストリームを持てる。パケットがロスしても、影響は該当 Stream に閉じる。

```
  // UDP データグラム (受信側)
  [A][B][A][C][✕][B][C][A]...
                ↑ ロス (後続のデータはブロックされない)

  // QUIC Stream (アプリケーションから見た状態)
  Stream A: ■ ■ □ □ □   ← 再送待ち
  Stream B: ■ ■ ■ ■ ■   ← 影響なし
  Stream C: ■ ■ ■ ■ ■   ← 影響なし

  ■ 受信済み  □ 再送待ち  ✕ ロスしたパケット
```

MoQ はこの性質を活かし、映像・音声のデータを適切な粒度で Stream に分けて送信する。詳しくは後述する。

### MoQT

MoQT (Media over QUIC Transport) は、QUIC 上で汎用的なデータ配信を行うためのプロトコルである。
主に下記の 2 つを定義している。

1. データをどう構造化するか (Track / Group / Subgroup / Object)
2. 中継サーバーである Relay を介してスケーラブルに配信するための Pub/Sub の仕組み

MoQT は、データをどう構造化し、どう届けるかという配信のアーキテクチャそのものを定義しており、MoQ の中核にあたる。
この資料では、MoQT を中心に説明する。

### MSF

MSF (MoQT Streaming Format) は、MoQT 上にどう映像・音声を乗せるかを定めた仕様で、主に下記の 2 つを定義している。

1. どのような映像・音声を配信しているかの記述方法
2. 映像・音声をどう区切ってMoQT上で送るかのルール

この資料では、それぞれ雰囲気だけ伝える。

記述方法については、配信したい映像のコーデック・解像度・フレームレートなどの情報を、カタログと呼ばれる JSON に記述する。
配信側はこのカタログを Track (MoQT のデータ構造の一つ) として配信する。
受信側はその Track を購読することで、どのような映像・音声が配信されているかを知ることができる。

乗せ方については、例えば映像の 1 フレームを Object (これも MoQT のデータ構造の一つ) にマッピングして送る。

> [!NOTE]
> MSF では、LOC (Low Overhead Media Container) という仕様に基づき、エンコード済みの 1 フレームを 1 Object にマッピングする。
> また、HLS/DASH などで使われる fMP4 (fragmented MP4) セグメントを Object にマッピングする CMSF という MSF の拡張仕様もある。

### MoQ は QUIC をどう上手く使うのか

プロトコルの詳細に入る前に、MoQ がメディアデータをどのように QUIC Stream に乗せるかのイメージを掴んでほしい。
ここでは、映像を例に、メディアデータがどのように分割され QUIC Stream にマッピングされるかを説明する。


映像を送るには、映像データをどう分割して送るかが問題になる。
MoQ はこの分割単位として GoP (Group of Pictures) を使い、各 GoP を 1 つの QUIC Stream にマッピングする。
以下で詳しく見ていく。

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
GoP 内のフレームは先頭の I-frame から順にデコードする必要がある (P-frame は前のフレームとの差分なので、先行するフレームがないとデコードできないため)。
例えば、30fps でキーフレーム間隔が 1 秒の映像なら、1 つの GoP は I-frame 1 枚 + P-frame 29 枚 = 30 フレームになる。

MoQ では、この GoP を 1 つの QUIC Stream に乗せる。

```
  GoP 1  ──→  QUIC Stream 1: [I][P][P][P][P]...
  GoP 2  ──→  QUIC Stream 2: [I][P][P][P][P]...
  GoP 3  ──→  QUIC Stream 3: [I][P][P][P][P]...
```

このマッピングにより、QUIC Stream の性質を以下のように利用できる。

- GoP 内のフレームは順序通りに届く必要がある → Stream は順序保証してくれる
- GoP ごとに独立した Stream になる → ある GoP のパケットロスが、別の GoP の配信をブロックしない。例えば、古い GoP で再送待ちが発生しても、新しい GoP はブロックされずに届く

## Media over QUIC Transport (MoQT)

ここからは、MoQT のデータモデルと Pub/Sub の仕組みを詳しく説明する。

### データモデル

MoQT のデータモデルは、映像や音声などのデータを Track > Group > Subgroup > Object の階層構造で表現する。

> [!WARNING]
> MoQT 自体は各階層構造それぞれに何をマッピングするかを規定していない。
> 以降では、わかりやすさのために MSF に沿って映像を例に説明する。

例えば、Alice の映像配信は下記の階層構造になる。

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

Track は、メディアの論理的な単位である。
例えば「Alice の映像」や「Alice の音声」がそれぞれ 1 つの Track に対応する。
Subscriber は、この Track 単位でデータを購読する。

Track の中は Group で区切られる。
Group は途中参加ポイント (join point) になり、途中から購読した Subscriber は Group の境界から受信を開始できる。
映像の場合、GoP が Group に相当する。

Group の中はさらに Subgroup で区切られる。
1 Subgroup が 1 QUIC Stream にマッピングされる。
[MoQ は QUIC をどう上手く使うのか](#moq-は-quic-をどう上手く使うのか) で説明した通り、1 Group = 1 Subgroup の場合、GoP がそのまま 1 つの QUIC Stream に対応する。
1 Group に複数の Subgroup がある場合は、1 つの GoP が複数の QUIC Stream に分かれる (後述の [SVC を使う映像](#より実践的な例-svc-を使う映像) を参照)。

Object は MoQT における最小のデータ単位で、MSF では映像の 1 フレームが 1 Object に対応する。

#### より実践的な例: SVC を使う映像

先ほどの例では 1 Group = 1 Subgroup にしていた。SVC (Scalable Video Coding) のように、1 つの映像を Base Layer (低品質) と Enhancement Layer (高品質) のように複数のレイヤーに分けてエンコードする場合は、レイヤーごとに Subgroup を分けることができる。Subgroup が分かれると QUIC Stream も分かれるため、Base Layer を優先して配信したり、帯域不足時に Enhancement Layer だけを落としたりといった制御ができる。
Base Layer だけでも再生は可能なので、品質を下げつつ配信を継続できる。

```
Track: "Alice の映像 (SVC)"
├── Group 0 (GoP)
│   ├── Subgroup 0 (Base Layer)
│   │   ├── Object 0 (I-frame)
│   │   ├── Object 1 (P-frame)
│   │   ├── Object 2 (P-frame)
│   │   └── ...
│   └── Subgroup 1 (Enhancement Layer)
│       ├── Object 0 (enhancement data)
│       ├── Object 1 (enhancement data)
│       ├── Object 2 (enhancement data)
│       └── ...
├── Group 1 (GoP)
│   ├── Subgroup 0 (Base Layer)
│   │   ├── Object 0 (I-frame)
│   │   ├── Object 1 (P-frame)
│   │   ├── Object 2 (P-frame)
│   │   └── ...
│   └── Subgroup 1 (Enhancement Layer)
│       ├── Object 0 (enhancement data)
│       ├── Object 1 (enhancement data)
│       ├── Object 2 (enhancement data)
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

## まとめ

この資料では、MoQ の全体像と MoQT の基本を説明した。

- MoQ はトランスポート (QUIC) / データモデル + Pub/Sub (MoQT) / メディアの記述 + MoQT への乗せ方 (MSF) の 3 層からなるプロトコル群
- MoQ は GoP を QUIC Stream にマッピングすることで、QUIC の性質をうまく利用している
- MoQT のデータモデルは Track > Group > Subgroup > Object の階層構造
- Relay を介した Pub/Sub により、fan-out でスケーラブルな配信を実現する
- Pull 型フローは、セッション確立 → Namespace の発見 → Track の購読 → データ転送の順に進む

## 話していないテーマ

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

## 参考仕様

- [MoQT (Media over QUIC Transport)](https://datatracker.ietf.org/doc/draft-ietf-moq-transport/)
- [MSF (MoQT Streaming Format)](https://datatracker.ietf.org/doc/draft-ietf-moq-msf/)
- [LOC (Low Overhead Container)](https://datatracker.ietf.org/doc/draft-ietf-moq-loc/)
- [CMSF (Common MoQT Streaming Format)](https://datatracker.ietf.org/doc/draft-ietf-moq-cmsf/)
- [QUIC (RFC 9000)](https://datatracker.ietf.org/doc/rfc9000/)
- [WebTransport (RFC 9220)](https://datatracker.ietf.org/doc/rfc9220/)
