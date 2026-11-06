# game-news-nexus-config

[Game News Nexus](https://github.com/owayo/game-news-nexus) が読む、ゲームのお知らせの取得定義 (カタログ) です。
1 つのゲーム (取得元) を 1 つの YAML ファイルに書きます。

## アプリでの使われ方

- アプリはこのリポジトリの写しを同梱していて、初めて起動したときはそれを使います。
- 起動時と 1 日 1 回、このリポジトリの `main` を取り込みます (設定画面の「カタログ」から今すぐ取り込むこともできます)。
- 利用者はソース画面でゲームごとに on/off します。初めて起動したときは `groups.yaml` で `default: true` のグループが on になります。
  あとからそのグループに足したゲームも、利用者のアプリが初めて取り込んだときに on になります (off にしたものは off のまま)。
- 取り込みはファイルごとに判断します。書式が誤っているファイルや、そのアプリより新しい版を求めるファイル (`min_app_version`) は、同じソースの前の版を使い続けます。

## 構成

```text
groups.yaml                    グループ (ソース画面の見出しと並び順)
sources/<グループ>/<id>.yaml    取得定義。1 ファイル 1 ソース
schema/source.schema.json      取得定義の JSON Schema (エディタの補完と検査に使う)
```

取得定義の例 ([sources/hoyoverse/genshin_impact.yaml](sources/hoyoverse/genshin_impact.yaml)):

```yaml
schema: 2
min_app_version: '26.11.6'
id: genshin_impact
name: 原神
homepage: 'https://genshin.hoyoverse.com/ja/news'
lists:
  - kind: json
    request:
      url: 'https://sg-public-api-static.hoyoverse.com/content_v2_user/app/a1b1f9d3315447cc/getContentList?iAppId=32&iChanId=395&iPageSize=30&iPage=1&sLangKey=ja-jp'
    items: '/data/list'
    fields:
      id: '/iInfoId'
      title: '/sTitle'
      url: 'https://genshin.hoyoverse.com/ja/news/detail/{{/iInfoId}}'
      published: '/dtStartTime'
      body: '/sContent'
```

処理の流れは `prepare` (事前の取得) → `lists` (一覧) → `detail` (詳細ページ) → `body` (本文の整形) の段で決まっていて、各段で何をするかを書きます。
スクリプトのような任意の処理は書けません。

## ゲームを追加・修正したい

[CONTRIBUTING.md](CONTRIBUTING.md) に書き方と確かめ方をまとめています。PR を歓迎します。
自分で書くのが難しい場合は、Issue の「ゲームの追加のお願い」からお知らせのページを教えてください。

## 安全のための決まり

カタログの定義は第三者の PR から入るため、次のものは使えません。CI とアプリの両方で確かめます。

- 秘密の値の参照 (`{{secret:...}}`)。ログインが必要なページも対象外です。
- localhost や LAN (プライベートアドレス) への取得。アプリは名前解決後のアドレスとリダイレクト先も確かめます。
- http / https 以外の URL。

## ライセンス

[MIT](LICENSE)。取得定義が指すお知らせの内容は、それぞれの配信元のものです。
