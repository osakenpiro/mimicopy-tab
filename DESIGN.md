# 耳コピ→TAB自動採譜 v0 — 設計書（mimicopy-tab）

> #全人類UX改善計画 / TRBP T柱（Hi-Me TooL）
> 摩擦の塊「耳コピ」を、ポリフォニック採譜AIに肩代わりさせ、運指は自前ソルバで最適化する。

## まとめ（3行）
- **採譜の鬼門（音→音名）は自作せず Spotify Basic Pitch（OSS/TF.js）に丸投げ**。音声→MIDIノートはブラウザ内で完結（サーバ・アップロード不要）。
- **自前IPは MIDI→TAB 運指ソルバ**。フレットボードを層状グラフ化し、DP（Viterbi）で手の移動を最小化。和音は弦割当バックトラック＋スパン制約。カポ＝移調で対応。
- **完了条件＝音声file→MIDI→TAB表示の最小LIVE**（単音→和音）。本書はその設計と、検証済み/要実機確認の切り分け。

---

## 1. アーキテクチャ（5段パイプライン）

```
[audio file]
   │  ① Web Audio API: decodeAudioData → mono mix → 22050Hz 線形リサンプル (Float32Array)
   ▼
[Float32 @22050]
   │  ② Basic Pitch: basicPitch.evaluateModel(audio, frameCb, progressCb)
   ▼
[frames / onsets / contours]  ← 神経網の確率行列
   │  ③ outputToNotesPoly(frames,onsets, onsetTh, frameTh, minLen)
   │     → addPitchBendsToNoteEvents(contours, …) → noteFramesToTime(…)
   ▼
[NoteEventTime[]]  {startTimeSeconds, durationSeconds, pitchMidi, amplitude}
   │  ④【自前】MIDI→TAB 運指ソルバ（単音Viterbi / 和音バックトラック）
   ▼
[ (string,fret) 列 ]
   │  ⑤ ASCIIタブ譜レンダ（6弦×時間列）＋ノート一覧＋MIDI書き出し
   ▼
[TAB / MIDI download]
```

①②③は **Basic Pitch（Apache-2.0）** が担当。④⑤が本ツールの実装。

### 依存（検証済み）
- `@spotify/basic-pitch@1.0.1`（esm.sh で ESM 取得）。`@tensorflow/tfjs@^3.2.0` は**通常依存**なので esm.sh が依存閉包ごと解決（自前で tfjs を読む必要なし）。
- モデルは npm パッケージに同梱 → jsDelivr が CORS 付きで配信：
  `https://cdn.jsdelivr.net/npm/@spotify/basic-pitch@1.0.1/model/model.json`（+ 重み shard 742KB を同ディレクトリから自動取得）。
- モデル入力規格：**22050Hz / mono / Float32Array**（要厳守。違う rate を渡すと音程がずれる）。

---

## 2. 自前IP：MIDI→TAB 運指ソルバ

### 2.1 フレットボードモデル
- 標準チューニング EADGBE = 開放MIDI `[40,45,50,55,59,64]`（弦 index 0=6弦/低E … 5=1弦/高e）。
- **カポ C**：開放音 += C（半音）。表示フレットはカポ基準（カポ上を0とする標準TAB流儀）。`カポ=移調`の原理そのまま。
- ある音 `p` の候補位置：全弦で `fret = p − (open[s]+C)`、`0≤fret≤maxFret` のものだけ採用。全弦で負＝音域外（フラグ）。1音につき候補は通常2〜4個。

### 2.2 単音：層状グラフ × DP（Viterbi）
ノート列を時間順に並べ、各ノートの候補位置をノードとする層状グラフ上で**総移動コスト最小**の経路を解く。

- **ノードコスト**（その位置の弾きにくさ）：`fret==0 ? 0 : POS_W·fret`（低フレット/開放を緩く優先）
- **エッジコスト**（前位置→今位置の手の移動）：
  - 水平：どちらか開放(fret0)なら0、そうでなければ `|fret_a − fret_b|`
  - 弦移動：`|string_a − string_b|`
  - `cost = FRET_W·水平 + STRING_W·弦移動`
- **DP**：`dp[i][k] = node(k) + min_j( dp[i-1][j] + edge(j,k) )`、backpointer で復元。計算量 O(N·K²)、K≤6＝瞬時。
- 効果：左手が**同一ポジションに留まろうとする**自然な運指が出る（飛び回らない）。

既定重み：`FRET_W=1.0 / STRING_W=0.4 / POS_W=0.15`（チューニング可能）。

### 2.3 和音：弦割当バックトラック＋スパン制約
- **チャンク化**：onset が近接（既定60ms以内）のノートを1和音に束ねる。同一pitchは重複除去。
- 各和音で「各pitch→相異なる弦、有効fret」をバックトラックで全列挙：
  - 制約：押弦フレット（非0）の**スパン ≤ HAND_SPAN（既定4）**。開放弦はスパン計算から除外。
  - スコア：`スパン罰 + 前和音の手位置からの距離 + ポジション罰`。最小を採用。
- 次和音の手位置＝押弦フレットの最小値（人差し指基準）で連続性を担保。
- pitch>6 or 実現不能は下声部を間引き／partialフラグ。
- v0.2予定：**和音間もDP**で通し最適化（現状は前和音バイアスのグリーディ）。

### 2.4 レンダリング
6弦×時間列のASCIIタブ（高e上・低E下）。各イベント=1列、複数桁fretは桁揃え。タイミングは厳密量子化せず逐次配置（Basic Pitch出力は非量子化＝原演奏のマイクロタイミング保持）。別途ノート一覧（秒/pitch/string-fret）も出す。

---

## 3. 検証ステータス（正直版）

| 項目 | 状態 |
|---|---|
| Basic Pitch ブラウザAPI（evaluateModel/outputToNotesPoly）| ✅ 公開実装・複数ソースで確認 |
| モデルCDN URL + CORS + 22050規格 | ✅ jsDelivr file tree / 公式README で確認 |
| tfjs 依存解決（esm.sh）| ✅ 通常依存と確認（peerでない＝自前ロード不要） |
| 運指ソルバ（単音DP / 和音BT）| ✅ ロジック設計確定・実装済（決定論的・単体検証可） |
| **実ブラウザでの音声→TAB通し** | ⏳ **要実機確認**（モデル読込/推論/描画はデプロイ後にChromeで疎通確認。実音声投入はユーザ操作ゲート） |

> サンドボックスでブラウザ音声パイプラインをフル実行できないため、③までの「実音声通し」は**ユーザの実機 or Chrome MCPでの最終確認**に委ねる。ソルバ部（④⑤）は純ロジックで決定論的に検証可能。

## 4. 最初のテスト素材
- Vaundy 耳コピ・**カポ2**（ユーザ指定）。まず単音メロディ→次に疎な和音。
- ドラム/濃密ミックス/強リバーブは苦手（Basic Pitchの既知限界）。単旋律・ソロギター・ピアノが最良。

## 5. 地続き
- Strudel / MIDIパース筋と同じ「時間×pitch」表現。将来 Strudel パターン出力や五線譜化に接続可能。

## 6. ロードマップ
- v0：単一HTML、単音→和音、カポ、ASCIIタブ、MIDI書き出し。（本リリース）
- v0.2：和音間DP通し最適化／ドロップDなど別チューニング／量子化（BPM推定→グリッド整列）。
- v0.3：運指のセーハ検出・指番号付与／タブのSVG綺麗版／localStorage設定永続。
