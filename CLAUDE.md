# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 概要

「きょうりゅうことばラボ」：こども向けに、恐竜を題材に日本語の文づくり・助詞・漢字・たし算ひき算を遊びながら学ぶ Web アプリ。
**アプリ全体が `index.html` 1ファイル**（CSS・JS・データ・恐竜写真を全部インライン）。ビルド・依存パッケージ・テスト・リンターはない。

## 動かし方

```bash
python -m http.server 8765
```

`.claude/launch.json` に `static` という名前で同じ設定があるので、確認はプレビューの `static` で開く。
自動テストはないので、変更はブラウザで実際に遊んで確かめる。

- ホームの「テストモードに する」：全恐竜が使え、ほし・ずかん・レベルなどの記録が進まない（リロードで戻る）。バトルの確認に必須。
- JS は全体が `(function(){"use strict"; ... })();` に包まれているので、`DINOS` などはコンソールから触れない。確認は画面の DOM（`#b-slots .slot`、`#b-bank .chip` など）をクリック・読み取りして行う。

## index.html の構成

上から順に：

1. `<style>`：テーマ変数、画面ごとの CSS。やくわりの色 `--r-sub`（だれが）`--r-par`（じょし）`--r-wrd` `--r-mod`（ようす）`--r-prd`（じゅつご＝赤）が全モード共通。
2. 画面：`<section class="screen" id="s-xxx">`。`go(name)` で切り替える（help / build / quiz / kanji / zukan / bpick / battle / react）。
3. `<script>`：
   - `DINOS`（恐竜ごとの名前・説明・手書きの例文 `sents`）、`FACTS`（全長・体重・時代・食べ物・move など。テンプレート文とバトルの両方が使う）
   - `ART`（手描き SVG）と `PHOTO`（透過 WebP の data URI）。`dinoSVG(id)` は `PHOTO` を優先し、なければ SVG。`PHOTO` は外部の `build_photos.py` が書き換える巨大な行なので、手で編集・Read しない（Grep も `-o` などで行全体を出さない）。
   - きほん：`$` / `$$` / `pick` / `shuffle` / `speak`（音声読み上げ）/ `go` / 保存。
   - モードごとのセクション（`/* ===== ぶんを つくろう ===== */` のような見出しコメントで区切られている）。

### 文のデータ形式（全モード共通）

文は `[ことば, やくわり]` の配列。やくわりは `sub`（主語・名前）/ `par`（助詞）/ `wrd`（ことば）/ `mod`（ようす。`markMods` が付ける）/ `prd`（述語）。
こどもはカードを枠に並べて文を作る。正解の並びが複数あるときは `alts`（別の正しい並び）を持ち、`cands` の中から絞り込んでいく。

ダミーカード（まちがいの選択肢）が**別の正しい文を作れてしまわない**ことが一番大事な設計ルール。ダミー選びのコードの多く（`BPLAUS`、`BON_NG`、`bLookFar`、`bMoreExtras`、ぶんを つくろう のグループ除外など）はこのためにある。言葉やわざを足すときは、ダミーとして出たときに正解になってしまわないかを必ず考える。

### モード

- **ぶんを つくろう**（`newSentence`）：`TEMPLATES`（`FACTS` から自動生成）と `DINOS[].sents` から1文。おてつきヒント `missHint` / `missAsk`、つぎの枠のヒント `slotAsk` / `showSlotHint` はバトルでも共用。
- **クイズ / かんじ / ずかん**：それぞれ独立したセクション。
- **バトルを じっきょう**（`startBattle` → `nextRound` → `bSentence` → `bRenderSentence` / `bTap` → `bFinish`）：2匹の戦うアニメを見て、何が起きたかを文にする。最後の述語（赤）は最初から枠に入っている。当たったら残りの体力を計算する。
  - `BATTLE[id]`：p＝つよさ、m＝わざ `MV(ぶき, どうし, full)`、d＝まもり、c / cw＝はんげきの確率・ぶき
  - `BVERBS`：どうしごとの「に／を」、ようすの言葉 `on`、うけみ `ps`。`sp` はダミーに出さない、`sig` はとくいわざ
  - `BMOVE`：どうし →（攻撃の動き `BATK`, 受けた側の動き `BHURT`）。動きは Web Animations のキーフレーム（`bKF`）で、prefers-reduced-motion でも止めない（動きを見て文を作るモードなので）
  - `BRESULT`：受けた側がどうなったかの文
- **はやうち たいけつ**（`r` で始まる関数群、`rs`）：バトルのデータ（`BATTLE` / `BMOVE`）を共用するので、わざを足すと こちらにも出る。

### 保存

`localStorage` の `dinolab_*` キー（stars, got＝ゲットした回数＝レベル, kanji_streak, bset, rset, rrec, rrules）。読み書きはすべて try/catch で囲む。テストモードでは記録しない。

## 書き方の決まり

- 画面の文字・読み上げ・**コードのコメント**は、こども向けに ひらがな中心・分かち書き（「つぎの わくに ヒントを だす」）。既存のコメントに合わせる。
- 既存コードは ES5 だけで書かれている（`var`、`function`。アロー関数・`let`・`const`・テンプレート文字列はない）。合わせる。
- コミットメッセージも ひらがな中心で「モード名: やったこと」の形（例：`バトルを じっきょう: とくいわざを ふやす`）。
- 作業はブランチを切って PR にし、master へは merge commit でマージする。
