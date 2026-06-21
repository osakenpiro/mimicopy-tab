# 🏮 mimicopy-tab — 耳コピ→TAB 自動採譜 v0

音声ファイルを**ブラウザの中だけ**でMIDI化し、運指最適化でギターTAB譜に自動採譜するゼロコスト・ゼロメンテのWebツール。アップロードもサーバも不要。

**LIVE:** https://osakenpiro.github.io/mimicopy-tab/

## 何をする
1. 音声（mp3/wav/m4a/flac/ogg）をドロップ
2. [Spotify Basic Pitch](https://github.com/spotify/basic-pitch-ts)（TF.js）がブラウザ内で **音声→MIDIノート** に変換
3. 自前の**フレットボード運指ソルバ**が MIDI→TAB に変換（左手の移動を最小化）
4. ASCIIタブ譜・ノート一覧・MIDI書き出し

## 設計の核
- **採譜の鬼門（ポリフォニック音→音名）は自作しない** → Basic Pitch にアウトソース。
- **自前IP = MIDI→TAB 運指ソルバ**：
  - 単音 = フレットボードを層状グラフ化し **Viterbi(DP)** で総移動コスト最小の運指。
  - 和音 = 弦割当**バックトラック** ＋ スパン制約（≤4フレット）＋ 前和音への近さ。
  - **カポ＝移調**で対応（表示はカポ基準フレット）。
- 標準チューニング EADGBE。詳細は [`DESIGN.md`](./DESIGN.md)。

## 使い方のコツ
- 単旋律・ソロギター・ピアノ・ボーカルが最良。ドラム/濃密ミックス/強リバーブは苦手（Basic Pitchの限界）。
- 「音域外」が多い時はカポ番号・最大フレットを調整。
- オンセット/フレーム感度で検出のシビアさを調整。

## スタック
単一HTML / Spotify Basic Pitch (Apache-2.0, TF.js) / 自前ソルバ / GitHub Pages。ビルド不要・依存はCDN（esm.sh + jsDelivr）。

## ステータス
v0。運指ソルバは決定論的テスト 13/13 pass。音声→TAB通しは実ブラウザで確認。ロードマップは `DESIGN.md` §6。

---
#全人類UX改善計画 ・ Visionium / TRBP **T柱 Hi-Me TooL** ・ [@osakenpiro](https://github.com/osakenpiro)
