# CLAUDE.md — イベント用写真スロット（PWA）

飲み会・子ども会で使う「写真スロット」。参加者の顔写真でリールを回し、当選者と罰ゲーム（お題）を決める。
GitHub Pages で公開し、スマホのホーム画面に追加してオフラインで動かす前提。

- リポジトリ: https://github.com/miuratoshiroh-beep/slot （既定ブランチ main）
- 公開URL: https://miuratoshiroh-beep.github.io/slot/event_slot_pwa_20261002_v06/

## ファイル構成

すべて `event_slot_pwa_20261002_v06/` 配下。

| ファイル | 内容 |
|---|---|
| `index.html` | アプリ本体（HTML/CSS/JS 全部入りの1ファイル、ビルド工程なし） |
| `manifest.webmanifest` | PWA定義 |
| `sw.js` | サービスワーカー（キャッシュ優先・オフライン用） |
| `icon-192.png` / `icon-512.png` / `icon-512-maskable.png` / `apple-touch-icon.png` | アイコン |

**フォルダ名 `event_slot_pwa_20261002_v06` は絶対に変えない。**
変えるとインストール済みPWAのURLが変わり別アプリ扱いになり、利用者の写真データが消える。
中身のバージョンは `sw.js` の `CACHE`（現在 `event-slot-v08`）で管理。フォルダ名の v06 と一致しないのは意図的。

## index.html の主な構成

- 状態 `S`（title / term / people / rate / reach / revRate / excludeHit / autoStop / pen / penRankOn / penStep / history）
- 保存: `db()` `idbGet()` `idbSet()` `save()`（200msデバウンス）→ `flushSave()`、起動時 `load()` / `loadPresetsInit()`
- リール: `reels[]`（p=位置, v=速度, state=idle/spin/stopping）、`order[i]`＝各リールの並び（シャッフル済み人物index）
- レイアウト: `layout()` → `setCell()` が CSS変数 `--cell` `--rh` `--off` 等を設定
- 抽選: `decide()` が開始時に結果（cells / hit / revenge）を確定 → `tryStop()` がその位置に止める
- 進行: `startOrStop()` → `requestStop()` / `tryStop()` → `onReelStop()` → （逆転時 `startSlip()`）→ `finish()` → `revealResult()` → `showResult()`
- 罰ゲーム: `autoRank()` / `pickPenalty()`（空ランクは隣から拾う）、強制指定 `forceId` / `forceRank`（保存されない・1回で解除）

## 壊してはいけない不変条件

### (a) リールの停止位置
`render()` の `translateY(OFF - pm * CELL)` と `tryStop()` の `dist` 計算は対になっている。
CSS変数 `--cell` / `--rh` / `--off` と `layout()` / `setCell()` を触ったら、
**「画面中央に表示されている画像」と「内部で決めた当選者」が一致することを必ずテストで確認する。**
ズレると揃っていないのに当選に見える致命的バグになる（最重点の検証箇所）。

### (b) 保存層
IndexedDB（DB名 `event_slot` / ストア `kv` / キー `event_slot_v1`・`event_slot_presets_v1`）。
`load()` に localStorage からの移行パスがある。壊すと既存利用者の写真が消える。
**localStorage と IndexedDB へのアクセスは必ず try/catch（Promise なら .catch）で囲む。**
囲み忘れで管理画面の初期化が止まる事故を実際に起こしている。

### (c) サービスワーカーのキャッシュ名
`index.html` 等を変更したら `sw.js` の `const CACHE = 'event-slot-vNN'` の番号を必ず上げる。
上げないとキャッシュ優先で古い画面が出続け、利用者に更新が届かない。

### (d) 演出中の操作ガード
3つ揃ってから結果画面が開くまでの約1.1秒（`pendingResult` / `flashTimer`）と、
結果画面表示直後0.6秒（`resultAt` / `RESULT_GUARD`）に操作ガードがある。
外すと「結果画面が出る前にSTARTを押すと次のゲームが始まり、前の結果が後から割り込む」不具合が再発する。

## 検証のやり方

ローカルサーバ＋Playwright。`file://` ではサービスワーカーが動かない。

    cd ~/Developer/slot/event_slot_pwa_20261002_v06
    python3 -m http.server 8777
    # → http://localhost:8777/index.html

Playwright はリポジトリに入れない（作業用ディレクトリに `npm i playwright` して使う）。

**注意:** localStorage にシードして reload する方式は効かない（IndexedDB が優先される）。シード前に必ず DB を消す。

    await page.evaluate(() => new Promise(r => {
      const q = indexedDB.deleteDatabase('event_slot');
      q.onsuccess = q.onerror = q.onblocked = () => r();
    }));

検証観点:
- 停止位置と内部判定の一致（最重要）。各リールについて、ライン枠（`.line`）の中心にある `<img>` と `order[i][pos]` / `result.cells[i]` が一致すること
- 手動/自動モード、連打・先行クリック
- 逆転演出が当たり時のみ発生すること
- 画面サイズ: iPhone SE / iPhone 13 / Pixel 7 / iPad / PC 1440x900 ではみ出しなし
- 保存不可環境（localStorage と indexedDB を両方ブロック）でも警告バーを出して動作すること

## 反映手順

1. `index.html` 等を編集
2. `sw.js` の CACHE 番号を上げる
3. コミット
4. push（GitHub Pages が自動反映、1〜2分）
5. スマホのアプリを2回起動すると新版に切り替わる（1回目は裏で取得）

## Git運用

- 作業が一区切りしたら、指示がなくてもコミットする
- 1コミット＝1つの意味のある変更
- コミット前に `git status` と `git diff --stat` で対象を確認する
- `git add -A` / `git add .` は使わない。必ず `git add <パスを明示>`
- **push は行わない。** コミットまでで止め、「pushしますか？」と確認する
- コミットメッセージ: 1行目に `種別: 日本語の要約`、本文に理由を1〜3行
  - 種別: feat / fix / refactor / docs / chore / test

## 回答の仕方

- 日本語で、結論から先に
- 未検証の箇所は「推測」と明記し、検証手順を併記する
- 設計だけ・方針だけの回答は不要。動く状態まで出す
- 手作業のほうが早い場合はそう指摘してよい
