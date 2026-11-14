# 取得定義の書き方

ゲームのお知らせの取得定義を追加・修正するための手引きです。
書き方に迷ったら、`sources/` の中の似たソースを写して直すのがいちばん早いです。

## 用意するもの

- お知らせが載っているページの URL。ログインせずに見られるページだけが対象です。
- 書いた定義を確かめる `gnn` コマンド (後述の「手元で確かめる」)。
- あれば VS Code と [YAML 拡張](https://marketplace.visualstudio.com/items?itemName=redhat.vscode-yaml)。このリポジトリを開くと JSON Schema が効き、項目の補完と誤りの表示が出ます。

## ファイルの置き場所

```text
sources/<グループ>/<id>.yaml
```

- `<id>` は英小文字・数字・`_`・`-` で、ファイル名と中の `id` を同じにします。
- グループは `groups.yaml` にあるもの (`dmm`・`hoyoverse`・`steam`・`media` など) から選びます。新しいグループが要るときは `groups.yaml` にも足します。
  `default: true` のグループに足したゲームは、利用者のアプリが初めて取り込んだときに on になります。
- **公開した `id` は変えないでください。** アプリは記事と on/off を `id` で覚えているので、変えると利用者の記事と設定が引き継がれません。

## YAML の決まり

- 文字コードは UTF-8、改行は LF にします。
- **文字列は単一引用符 (`'...'`) で囲みます。** 特に URL・CSS セレクタ・テンプレート・正規表現・JSON Pointer です。
  `#` で始まるセレクタは引用符がないとコメントになり、`{{` で始まる値は別の書き方と読まれます。
  単一引用符の中では `\` をそのまま書けるので、正規表現は `'\d+'` と書きます。単一引用符そのものは `''` と 2 つ重ねます。
- 英字だけの短い値 (`kind: json` など) と数値・真偽値 (`true` / `false`) は引用符なしで構いません。
- アンカー (`&`)・エイリアス (`*`)・マージ (`<<`)・独自タグ (`!`)・1 ファイルに複数の文書 (`---`) は使えません。
- `yes` / `no` / `on` / `off` は真偽値になりません (YAML 1.2 で読みます)。真偽値は `true` / `false` で書きます。

## 全体の形

```yaml
# ゲームの名前 (先頭のコメントは自由に書けます)
# yaml-language-server: $schema=../../schema/source.schema.json
schema: 2                     # 書式の版 (必須)
min_app_version: '26.11.6'    # このソースに必要なアプリの最小の版 (必須)
id: example_game              # ファイル名と同じ (必須)
name: サンプルゲーム          # 画面に出す名前 (必須)
homepage: 'https://example.com/news/'   # 公式のお知らせのページ
tags: [example]
note: '利用者向けの説明'
channel: news                 # お知らせ (既定。省略可)
timezone: Asia/Tokyo          # 時差のない日時をどの地域の時刻として読むか (既定 Asia/Tokyo)
schedule:                     # 本文の予定 (メンテナンスなど) の日時を読む地域時刻 (任意。後述)
  timezone: Asia/Tokyo
interval_minutes: 60          # 取得間隔の目安 (分)。省略時はアプリの設定
max_items: 50                 # 1 回の取得で読む記事の上限 (既定 100)
adult: true                   # 成人向け (FANZA GAMES の版など)。任意。後述
general_edition: example_game # 同じゲームの一般向けの版の ID (成人向けの版だけ。任意。後述)

prepare: []                   # 1. 事前の取得 (任意)
lists: []                     # 2. 一覧 (必須。1 つ以上)
detail: {}                    # 4. 詳細ページ (任意)
body: {}                      # 5. 本文の整え方 (任意)
url_identity: {}              # 画像の URL の同一性 (任意。後述)
```

## 処理の流れ

1. **`prepare`**: 一覧の URL に入れる値を、先に別のページから取ります。
2. **`lists`**: 一覧を取得し、記事の候補を作ります。複数書くと宣言順にまとめます。一覧ごとにページ送りもできます。
3. **まとめ** (アプリが行う): 記事を見分け、重複・取得済み・古い記事を除きます。
4. **`detail`**: 残った記事だけ詳細ページを取り、本文などを補います。取得済みで変わっていない記事の詳細は取り直しません。
5. **`body`**: 本文を整えます。

記事は `id` → `url` → `title` の順で見分けます。同じ記事なら毎回同じ値になる項目を `id` にしてください。
複数の一覧で同じ値が別の記事を指すときは、`id: 'news-{{/id}}'` のように一覧ごとに接頭辞を付けます。

## 一覧 (`lists`)

一覧ごとに `kind` で読み方を選びます。どの一覧にも `request` (取得の仕方) が要ります。
`name` を書くと、失敗したときの表示に使います。

```yaml
request:
  url: 'https://example.com/api/news'   # 必須
  method: POST                          # 既定は GET
  query:                                # URL に足す問い合わせ (値は符号化して付ける)
    limit: '30'
  headers:
    Referer: 'https://example.com/'
  body: '{"page": 1}'                   # POST の本文
  encoding: shift_jis                   # 文字コードを固定する (既定は応答から判定)
```

### `feed`: RSS / Atom / JSON Feed

URL だけで記事になります。`content` で本文に使う要素を選べます (`auto` (既定)・`content`・`summary`)。

```yaml
lists:
  - kind: feed
    request:
      url: 'https://www.4gamer.net/rss/index.xml'
```

### `json`: JSON API

`items` に記事の配列を指す [JSON Pointer](https://www.rfc-editor.org/rfc/rfc6901) を書き、`fields` で項目を取り出します。

```yaml
lists:
  - kind: json
    request:
      url: 'https://example.com/api/news'
    items: '/data/articles'
    filter:
      - { pointer: '/visible', eq: true }    # 条件に合う記事だけを残す
    fields:
      id: '/id'
      title: '/title'
      url: 'https://example.com/news/{{/id}}'   # テンプレートで組み立てられる
      published: '/publishedAt'                  # 日時の文字列か UNIX 時刻 (秒・ミリ秒)
      body: '/content'                           # HTML の文字列
      image: '/bannerUrl'
      note: '/category'
```

項目の値は JSON Pointer (`/title`) か、`{{/pointer}}` を含むテンプレートです。
本文が「画像と文章のブロックの配列」なら、次のように書けます。

```yaml
      body:
        blocks: '/bodies'      # ブロックの配列
        text: '/text'          # ブロックの文章
        image: '/imageUrl'     # ブロックの画像
        image_when: { pointer: '/type', eq: 6 }   # 画像のブロックの条件 (省略時は画像があれば画像)
```

### `html`: HTML の一覧

`items` に記事 1 件に当たる要素の CSS セレクタを書きます。`fields` は記事要素から見たセレクタです。

```yaml
lists:
  - kind: html
    request:
      url: 'https://example.com/news/'
    items: 'ul.news-list > li'
    require: '.icon-new'          # これを含む記事だけを残す (任意)
    fields:
      title: '.title'             # セレクタだけなら要素の文字列
      url: { selector: 'a', attr: href }
      published: { selector: 'time', attr: datetime }
      image: { selector: 'img', attr: src }
```

`url` を省くと記事要素 (またはその中の最初の `<a>`) の `href`、`image` で属性を省くと `src` を取ります。
値を細かく取るときは次の項目を使います。

| 項目 | 意味 |
| --- | --- |
| `selector` | 記事要素から見た CSS セレクタ。省略時は記事要素そのもの |
| `attr` | 属性の値を取る |
| `extract` | `text` (文字列)・`html` (中身の HTML)・`outer_html` (要素ごと)。既定は本文が `html`、ほかは `text` |
| `remove` | 取り出す前に取り除く要素のセレクタの配列 |
| `template` | 取り出した値を埋め込む文字列。`{{value}}` が値になる |

### `html_json`: HTML に埋め込まれた JSON

`selector` で JSON を持つ要素を選び、`attr` (省略時は要素の中身) の JSON を読みます。あとは `json` と同じです。

```yaml
lists:
  - kind: html_json
    request:
      url: 'https://example.com/'
    selector: '#app-data'
    attr: data-announcements
    items: '/items'       # 省略時は JSON そのものが配列
    fields:
      id: '/id'
      title: '/title'
```

### ページ送り (`pages`)

`json`・`html`・`html_json` の一覧で使えます。どの方式も `max_pages` (最大 10) で読むページ数を限ります。
記事のないページ、同じ URL の繰り返しで止まります。お知らせでは、ページの記事がすべて取得済みか古ければ、それより先は読みません。

```yaml
    pages: { kind: page, param: page, max_pages: 3 }     # ?page=1, 2, 3
    pages: { kind: page, param: offset, start: 0, step: 30, max_pages: 3 }   # ?offset=0, 30, 60
    pages: { kind: next_link, selector: 'a.next', max_pages: 5 }   # HTML の「次へ」のリンク
    pages: { kind: next_link, pointer: '/links/next', max_pages: 5 }   # JSON の次のページの URL
    pages: { kind: cursor, pointer: '/nextCursor', param: cursor, max_pages: 5 }
```

`page` で `param` を省くと、URL か問い合わせの `{{page}}` に番号が入ります。

### 条件 (`filter`)

JSON の一覧で、条件をすべて満たす記事だけを残します。1 つの条件には `eq`・`ne`・`exists`・`contains` のどれか 1 つを書きます。
`true` と `"True"`、`6` と `"6"` は同じとみなします。

```yaml
    filter:
      - { pointer: '/visible', eq: true }
      - { pointer: '/type', ne: 3 }
      - { pointer: '/deletedAt', exists: false }
      - { pointer: '/category', contains: 'お知らせ' }   # 文字列に含む
```

## 事前の取得 (`prepare`)

一覧の URL に入れる値を先に取ります。取り出した値は、後の段で `{{vars.名前}}` として使えます。

```yaml
prepare:
  - request:
      url: 'https://example.com/api/catalog'
    bind:
      catalog: '/data/latest'           # kind が json (既定) なら JSON Pointer
  - kind: html
    request:
      url: 'https://example.com/'
    bind:
      build: { selector: 'meta[name=build-id]', attr: content }
      token: { selector: 'script#config', regex: 'token: "([^"]+)"' }   # 括弧の中を値にする
lists:
  - kind: json
    request:
      url: 'https://example.com/api/news/{{vars.catalog}}'
```

`kind` は `json`・`html`・`text` です。`text` では `regex` で応答全体から取り出します。

## 詳細ページ (`detail`)

一覧で本文が取れないとき、記事ごとに詳細ページを取ります。`request` を省くと記事の URL を取ります。
詳細ページから取った値は、一覧の値より優先します。

```yaml
detail:
  kind: html                 # html・json・html_json
  request:
    url: 'https://example.com/api/news?id={{item.id}}'
  fields:
    title: 'h1'
    body: { selector: '.article-body', remove: ['.share', '.related'] }
```

`html` で `fields.body` を省くと、ページの `<body>` 全体が本文になります。
`html_json` では `selector`・`attr` に加えて、記事を指す `item` (JSON Pointer) を書けます。

## 本文 (`body`)

本文は次の順に整えます。安全のためのタグ・属性・URL の除去は、設定によらず必ず行います。

1. `format: text` なら HTML の特殊文字を逃がす
2. `replace` の正規表現で置き換える
3. `image_in_body: true` なら記事の画像を先頭に入れる
4. `remove` のセレクタに合う要素を取り除く
5. 安全なタグ・属性・URL だけを残し、`tidy` (既定 `true`) なら空の要素と続く改行を整理する

```yaml
body:
  format: text               # html (既定) か text
  newlines_to_br: true       # 改行を <br> にする (text では常に行う)
  image_in_body: true
  base_url: 'https://example.com/news/'   # 相対 URL の基準 (既定は本文を取ったページ)
  remove: ['.ad', '.share-buttons']
  replace:
    - pattern: '([^\r\n]+\.jpg)[\r\n]+'
      with: '<img src="https://example.com/news/$1">'   # $1 は括弧の中
```

## 本文の予定の地域時刻 (`schedule`)

アプリは保存した本文から、メンテナンスの予定と結果 (完了・延長・中止など) と、イベントなどの期間を読み取ります。
一覧の「今日・明日の予定を上に」と、「メンテ中」などの表示に使います。
本文の日時に `(JST)` のような地域時刻の明記がないとき、どの地域の時刻として読むかを `schedule` に書けます。

```yaml
timezone: Asia/Shanghai              # 一覧の投稿日時は UTC+8
schedule:
  timezone: Asia/Tokyo               # 本文の日時は日本時間 (省略時は上の timezone)
  server_timezone: Asia/Shanghai     # 「サーバー時間」と書かれた日時 (省略時は決めない)
```

- `timezone`: 本文の日時に地域時刻の明記がないときに使います。省略すると、投稿日時と同じソースの `timezone` で読みます。
  一覧の日時と本文の日時で地域時刻が違う配信元 (一覧は UTC+8 で、本文は日本時間、など) で書きます。
  本文に `(JST)` などの明記がある日時は、明記のとおりに読みます。
- `server_timezone`: 本文に「サーバー時間」と書かれた日時の地域時刻です。省略すると、その日時の地域時刻を決めません (推測で補いません)。
- どちらも `Asia/Tokyo` のような IANA のタイムゾーン名で書きます。
- `schedule` を使うソースは `min_app_version` を `'26.11.6'` 以上にします。
- 書き換えても記事を取り直すことはありません。アプリは保存済みの記事を新しい地域時刻で読み直します。

## 画像の同一性 (`url_identity`)

画像の URL に署名・期限や、取得した時刻のキャッシュ破りの問い合わせが付き、同じ画像でも取得のたびに URL が変わる配信元があります
(`https://cdn.example.com/img/a.png?s=署名&t=期限` など)。そのままだと、アプリは取得のたびに記事を「更新」にし、同じ画像を取り直します。
`url_identity.images` に、違っても同じ画像とみなす問い合わせを書きます。

```yaml
url_identity:
  images:
    - origin: 'https://cdn.example.com'   # スキームとホスト (既定でないポートも)。末尾の / は書かない
      path_prefix: '/secure/img/'          # パスの先頭 (任意。省略時はそのオリジンのすべての画像)
      ignore_query: ['s', 't']             # 違っても同じ画像とみなす問い合わせの名前
```

- 一覧・詳細の `image` と、本文の `<img>` の画像にだけ効きます。記事の URL やリンクには効きません。
- アプリは、記事が変わったかを比べるときと画像のキャッシュでは、`ignore_query` の問い合わせを外して同じ画像として扱います。
  取りに行くときは、取得したときの配信元の URL (署名付き) をそのまま使います。署名を外すと画像を返さない配信元もあるためです。
- 書くのは、取得のたびに変わり、変わっても画像の中身は同じ問い合わせだけです。`?v=2` や `?t=20261030` のような版
  (中身が変わったときに変わる値) は書かないでください。書くと、画像が差し替わっても古い画像を出し続けます。
- 名前は大文字と小文字を区別します。同じオリジンで、パスの範囲が重なる決まりは書けません。
- 変わる問い合わせかどうかは、時間をおいて 2 回取得した URL を比べて確かめます。`gnn try` は、宣言に合った画像の数を
  `画像の同一性: 宣言に合う画像 N 件` と出します。1 件も合わなければ、`origin` と `path_prefix` を見直してください。
- `url_identity` を使うソースは `min_app_version` を `'26.11.6'` 以上にします。
- 書き足しても、保存済みの記事は「更新」になりません。アプリは次の取得のときに、保存済みの画像を同じ決まりで直して比べます。

## 一般版と FANZA 版 (`adult`・`general_edition`・`inherit_articles`)

DMM GAMES のゲームには、一般向けの版 (DMM GAMES) と成人向けの版 (FANZA GAMES) があり、お知らせの取得先が違うものがあります。
版ごとに別のソース (別のファイル・別の ID) にします。

```yaml
# サンプルゲーム (FANZA 版)
schema: 2
min_app_version: '26.11.6'
id: example_game_r18          # 一般向けの版の ID に _r18 を付ける
name: サンプルゲーム
adult: true                   # 成人向け。利用者が設定で 18 歳以上と同意したときだけ使う
general_edition: example_game # 同じゲームの一般向けの版の ID
lists:
  - kind: json
    request:
      url: 'https://api-r18.example.com/news'   # 成人向けの版の取得先
```

- 成人向けの取得先 (FANZA GAMES の版・R18 の API・`.pink` のドメインなど) を使うソースには、必ず `adult: true` を書きます。
  一般向けの版のソースに、成人向けの版の取得先を書かないでください。
- 一般向けの版は、今の ID のまま一般向けの取得先にします。成人向けの版の ID は、一般向けの版の ID に `_r18` を付けます。
- `general_edition` は、同じグループ・同じ `channel` の、成人向けでないソースを指します。1 つの一般向けの版を指す成人向けの版は
  1 つだけにします。合わない・重なると、アプリはその成人向けの版を使いません (`gnn check` が知らせます)。
- FANZA GAMES だけのゲームは `adult: true` だけを書きます。
- アプリは、同意した利用者には一般向けの版の代わりに成人向けの版を取得します (on/off とアイコンは一般向けの版の設定を使います)。
  FANZA GAMES だけのゲームは、同意した利用者のソース画面に off で出ます。同意していない利用者には、成人向けのソースの名前も出しません。
- `inherit_articles: true` (成人向けの版に書く) は、これまで一般向けの版の ID で成人向けの版の取得先を取得していたゲームを、
  版ごとに分けるときだけ書きます。アプリは組を初めて見たときに 1 度だけ、一般向けの版の ID の記事を既読・予定ごと成人向けの版へ
  移します。新しく版を足すときは書きません (一般向けの版で取得した記事まで移ってしまうため)。
- これらを使うソースは `min_app_version` を `'26.11.6'` 以上にします。
- `gnn try` は、`--adult-consent` を付けたときだけ成人向けのソースを試します (18 歳以上で、成人向けの内容を取得することに同意する
  場合だけ付けます)。GitHub Actions の中では、成人向けのソースの記事の題名と本文を出しません。

## 差し込み (テンプレート)

| 書き方 | 中身 | 使える場所 |
| --- | --- | --- |
| `{{vars.名前}}` | `prepare` で取り出した値 | その後のすべての段 |
| `{{/pointer}}` | JSON の記事の値 | JSON の項目。JSON の一覧に続く `detail` |
| `{{item.id}}` `{{item.url}}` `{{item.title}}` | 一覧で読んだ記事の値 | `detail` |
| `{{page}}` | ページ送りの番号 | `pages.kind: page` の一覧 |
| `{{value}}` | HTML から取り出した値 | HTML の項目の `template` |

URL に差し込んだ値は符号化します。URL 全体を `'{{item.url}}'` のように差し込むときはそのまま使います。
URL のスキームとホストの部分には差し込めません。

## 版 (`schema` と `min_app_version`)

- `schema` は書式の版で、今は `2` です。
- `min_app_version` は、このソースを読めるアプリの最小の版です。カタログでは必須です。
  使っている機能が入った版以上を書きます (今の機能はすべて `26.11.6` から使えます)。足りなければ `gnn check` が教えてくれます。
- アプリは、自分より新しい版を求めるファイルを読まず、そのソースの前の版を使い続けます。

## 安全のための決まり

- 秘密の値 (`{{secret:...}}`) は使えません。ログインや会員登録が必要なページは対象外です。
- localhost・LAN (プライベートアドレス)・リンクローカルへの取得はできません。アプリは名前解決したアドレスとリダイレクト先も確かめます。
- URL は http か https にします。
- 配信元に負担をかけないよう、`interval_minutes` を短くしすぎず、ページ送りは必要な分だけにします。

## 手元で確かめる

`gnn` コマンドは Game News Nexus のリポジトリに入っています。[Rust](https://rustup.rs/) を入れてから、次で入ります。

```sh
cargo install --locked --git https://github.com/owayo/game-news-nexus gnn-cli
```

```sh
gnn check .                     # 書式と決まりを検査する
gnn try . example_game          # 実際のサイトから取得してみる (ID を省くとすべて)
gnn try . example_game --limit 5
gnn try . example_game_r18 --adult-consent   # 成人向けのソース (18 歳以上で同意する場合だけ)
```

`gnn try` は、どの段で何件取れたかと、記事の先頭を表示します。

アプリで試す場合は、ソース画面で手元の取得定義を作成し、YAML タブに内容を貼り付けて「テスト取得」で確認します。
ソース画面の「テスト取得」で、段ごとの結果を確かめられます。

## PR を出す前に

- [ ] `gnn check .` が通る
- [ ] `gnn try . <ID>` で記事のタイトル・日時・本文が取れた
- [ ] ファイル名と `id` が同じで、既存のソースの `id` を変えていない
- [ ] `min_app_version` を書いた
- [ ] 成人向けの取得先には `adult: true` を書き、一般向けの版があれば `general_edition` で組にした
- [ ] 秘密の値・手元のネットワーク・ログインが必要なページを使っていない

CI でも `gnn check` と、変えたソースの `gnn try` が動きます。配信元によっては海外 (CI の実行環境) からの接続を断るため、`gnn try` の失敗だけでは PR を止めません。その場合は手元の `gnn try` の結果を PR に貼ってください。
