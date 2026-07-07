---
name: publish
description: ブログ記事をpublishする。ファイル名とfrontmatterの日時を現在時刻に更新してからnebel generateし、blog-sourceとmizzy.github.com（public/）の両方をmainに直接push する。記事を公開・デプロイしたいときに使う。
---

# ブログ記事のpublish手順

対象記事は、引数で指定されたもの、または会話の文脈で編集中の記事。不明なら`git status`で未コミットの`posts/*.markdown`を確認する。

以下の手順を**必ずこの順序で**実行する。

## 1. 日時の更新（generateより前に必ずやる）

1. `date "+%Y-%m-%d %H:%M:%S %z"`で現在日時を取得する。
2. 記事のfrontmatterの`date:`を現在日時に更新する。
3. ファイル名の日付プレフィックス（`YYYY-MM-DD-`）がfrontmatterの日付と一致しない場合は、ファイルをリネームして揃える。ファイル名はASCIIのみ。

## 2. サイト生成

```bash
nebel generate
```

## 3. 古い生成物の掃除

手順1で日付が変わった場合、`public/blog/<旧YYYY>/<旧MM>/<旧DD>/`配下に同じ記事の古いコピーが残る（nebelは削除されたパスを掃除しない）。

1. 旧パスの`index.html`の`og:title`を確認し、対象記事の古いコピーであることを確かめてから削除する。
2. 空になった日付ディレクトリも削除する。
3. `public/index.html`が新しいURL（`/blog/YYYY/MM/DD/N/`）を指していることを確認する。

## 4. コミットとpush

このリポジトリとmizzy.github.comは、ブランチもPRも作らず**mainに直接push**する（個人ブログなのでレビュー不要）。worktreeも作らない。

1. blog-source: 記事ファイルをコミットしてpush。pre-commitフックが走る。
2. mizzy.github.com（`public/`のsymlink先）: 生成物（記事ページ、ogp.png、index.html、atom.xml、前後記事のナビゲーション更新分）をコミットしてpush。

コミットメッセージは英語で`Add post about <topic>`の形式（既存のログに倣う）。

## 5. 報告

公開URL（`https://mizzy.org/blog/YYYY/MM/DD/N/`）を報告する。
