# PC推薦パラメーター微調整手順書

この手順書は、顧客向け `index.html` とスタッフ向け `public/staff/index.html` のどこを変更すると、質問回答から作る希望ベクトルや、PCカタログとの一致度が変わるかを説明します。**同じ推薦尺度を両ページで保つため、正規化範囲や重みを変更するときは必ず両方を同時に直してください。**行番号は現在の版の目安です。将来行がずれたときは、記載した関数名・定数名で検索してください。

## 1. まず変更したい内容を分類する

| 変更したいこと | 顧客 `index.html` | スタッフ `public/staff/index.html` | 仕様書 |
|---|---|---|---|
| PCベクトルの正規化範囲 | `bounds` | `MATCH_RANGES` | 第2節 |
| 各項目の重要度/採点対象 | `MATCH_WEIGHTS` と `MATCH_SCORE_INDICES` | `MATCH_WEIGHTS` と `MATCH_SCORE_INDICES` | 第3節 |
| 優先条件フィルター | `recommendationSizeBand()` / `rankProducts()` | `recommendationSizeBand()` / `rankProducts()` | 第3・7節 |
| 回答からCPU/GPU/NPU/RAM/SSD/重量等を作る規則 | `makeVector()` | 旧QR対応が関係する場合は `npuTOPSFromAnswers()` | 第2・3節 |
| 質問文・回答選択肢 | `questions` | `QR_QUESTIONS` | 第3・7節 |
| PC登録フォームの入力欄・許容値 | — | `FIELDS` と `validateVector()` | 第8節・Rules |
| 商品項目・Firestoreキー・CSV列順 | `vectorKeys` / `fieldKeys` | `fieldKeys` / `FIELDS` / `HEADER` | 第8節 |
| QRの項目順・単位・回答コード | `QR_SCHEMA_VERSION` / `QR_ANSWER_CODES` | `QR_SCHEMA_VERSION` / `QR_QUESTIONS` / `parseQr()` | 第6・7節 |

普段の微調整では、質問やQRの項目を変えず、**正規化範囲・重み・`makeVector()` の需要式**だけを変更するのが安全です。

## 2. 正規化範囲を変える場所

顧客ページでは `index.html` の `const bounds`、スタッフページでは `public/staff/index.html` の `const MATCH_RANGES` が14成分の固定範囲を持ち、`normalizeMatchValue(index, value)` が0〜1へ線形正規化します。成分番号と順番は両方で次の14項目に固定されています。

| index | キー | 現在の範囲・比較方法 |
|---:|---|---|
| 0 | `cpuSingle` | 310–492（実機39機種） |
| 1 | `cpuMulti` | 1420–2850（実機39機種） |
| 2 | `npuTOPS` | 0–60 TOPS |
| 3 | `gpu` | 0–3000 |
| 4 | `ramGB` | 0–32 GB。充足型採点（必要量以上は満点） |
| 5 | `ssdGB` | 128–2048 GB |
| 6 | `weightG` | 700–2500 g |
| 7 | `design` | 1–10 |
| 8 | `displayInch` | 13.3–16 inch |
| 9 | `aspectRatio` | 16:9／16:10カテゴリ一致をスコア化（同じ1、異なる0）。ハード除外なし |
| 10 | `manufacturerMask` | メーカー優先/補完の候補選択。スコア対象外 |
| 11 | `failureFrequency` | Q14の故障少希望時に指定メーカー群を除外。スコア対象外 |
| 12 | `readability` | 1–10 |
| 13 | `positioning` | 0–10 |

数値項目は範囲内に線形正規化し、範囲外は0または1へクリップします。範囲を変更すると、商品値・顧客希望値の相対距離が変わります。アスペクト比はカテゴリ一致/不一致の0/1としてスコアに使いますが、メーカーと故障頻度はカテゴリ項目であり、スコアへ加えず候補条件に使います。いずれも数値範囲に置き換えないでください。

**変更手順:** 顧客側の `bounds` とスタッフ側の `MATCH_RANGES` を、同じ順序・同じ上下限にします。14要素であることを確認します。重み配列は両ページとも `MATCH_WEIGHTS`、採点対象は `MATCH_SCORE_INDICES` です。現在値の根拠と校正理由を[ベクトル設定基準書](選択肢からのベクトル設定基準.md)の第4節にも記録します。

CPU範囲は2026-11-10提供の実機CSV39機種（single 310〜492、multi 1420〜2850）を校正基準として固定しています。商品を追加するたびに範囲を自動変更することは避けます。Cinebench 2026の測定条件・商品群が変わったときは新しい実データで再校正してください。RAM範囲は0〜32GBで、要求16/32GBに対し必要量を満たせば満点、不足時だけ減点します。positioningは0〜10、デザイン/見やすさは従来どおり1〜10です。

## 3. 項目ごとの重みと採点対象を変える場所

顧客ページの `MATCH_WEIGHTS` とスタッフページの `MATCH_WEIGHTS` に、14成分の元の重みを同じ順序で置きます。元配列は `[1, 1, 0.8, 0.6, 1, 0.9, 0.7, 0.5, 1, 0.7, 0.65, 0.55, 0.65, 0.5]` のままです。

実際のスコア対象は両ページの `MATCH_SCORE_INDICES` が指定する `[0,1,2,3,4,5,6,7,8,9,12,13]` です。メーカーindex 10と故障頻度index 11はスコアに加えず、メーカー優先/補完と故障頻度の候補フィルターとして扱います。index 8の正確なインチ値とindex 9の画面比率はスコアに含みますが、Q2/Q3が作るサイズ帯やQ14の故障条件に違反する候補を点数で救済することはありません。

有効な重み合計は **9.35** です。点数は各成分の0〜1類似度に有効重みを掛けた合計をこの値で割り、100を掛けます。重みを調整するときは14要素・順序・非負値を保ち、両ページの元配列と基準書をそろえてください。スコア対象から項目を増減するときは、`MATCH_SCORE_INDICES` と有効重み合計も両ページ・基準書・この手順書で同期します。

メーカー、故障頻度、画面比率は別の意味を持ちます。画面比率はカテゴリ一致を0/1の類似度にしてスコアへ含めます。メーカーは指定候補を先に順位付けして件数不足分を他社で補います。故障が少ない希望ではASUS/Lenovo/HP/Dell/MSI/Acerを候補から外します。これらの条件を重み変更で代替しないでください。

## 4. 質問回答から希望スペックを作る場所

顧客ページの `makeVector()` が質問回答を14項目の希望ベクトルへ変換します。スタッフ側の `recalibratePreferenceVector()` はQR回答コードから旧版ベクトルのCPU/GPU/RAM/positioningを現行尺度へ合わせます。主な調整点は以下です。

- `purposeMap`：仕事・勉強・趣味のCPU需要基礎値。
- `taskMap`：各作業の `cpu`、`gpu`、`ram`、`ssd` の目安。ユーザー編集済みCSVのQ5 RAM候補とGPU係数を正本とし、例として「文書作成・編集」と「動画視聴」はRAM候補16GB、画像・動画編集のGPU係数は0.85、ゲームは0.90です。
- `yearsAdd`、`multiAdd`、`dCpu`：利用年数・マルチタスクをCPU需要へ加算する規則。
- `cpuSingle`、`cpuMulti`：実機39機種で校正したCPU需要式。`x=clamp((Dcpu−0.20)/0.80,0,1)`、single=`round(310+182×x)`、multi=`round(1420+1430×x)`。
- `gpu`：作業ごとの最大GPU需要 `taskGpu` からGPU目標を作る式。
- `npuTOPS`：利用年数・対象作業の基礎需要と、Q9電源なし／Q11静かな環境の加算。
- `purposeRamMap`、`taskMap.*.ram`、`q7Ram`、`yearsRam`、`ramNeed`、`ramGB`：RAM候補を最大集約し、`ramGB=min(32,max(16,purposeRam,taskRam,q7Ram,yearsRam))`。常に最低16GB、6年以上/マルチタスク頻繁/画像動画編集/ゲームは32GB。64GB要求は出しません。
- `ssdBase`、`ssdGB`：保存容量の目安。
- `baseWeight`、`carryCeiling`、`unpluggedAdj`：重さ・持ち運び・電源なし利用の目安。
- `displayInch`、`aspectRatio`、`design`、`readability`、`manufacturerMask`、`failureFrequency`、`positioning`：質問回答からカテゴリ・主観評価目標を決めます。Q2/Q3の厳守サイズ帯は`recommendationSizeBand()`が回答を解釈し、Q4の希望インチはスコア目標に残します。

CPUと商品positioningは39機種を使って相関を校正しています。商品側CPU二項目を別々に固定min-max正規化して平均したxに対して `positioning≈2.061+6.198×x`（RMSE約1.159点）。顧客のpositioning希望は `max(yearsPosition,round(2.061+6.198×x))`、positioningの登録/正規化範囲は0〜10です。GPU係数はユーザー編集済み質問CSVを尊重し、画像・動画編集0.85、ゲーム0.90を含む全作業値を顧客コード・スタッフ再計算・CSV・基準書で一致させます。質問5の「文書作成・編集」「動画視聴」のRAM候補は添付CSVどおり16GBです。スタッフは回答コード付きQRを受信した際、`recalibratePreferenceVector()`で旧version 2〜4もCPU/GPU/RAM/positioningを現行尺度へ再計算します。回答コードなし旧14数値QRのCPU値は`recalibrateLegacyVector()`が旧レンジを読み取り、表示用に現行CPU範囲へ変換します。

### NPU TOPSを調整する場合

新しい回答QRのTOPS希望値を変える主箇所は顧客側 `makeVector()` 内の `npuBase`／`npuFrequencyAdd()`／`npuTOPS` です。スタッフ側の `npuTOPSFromAnswers(codes)`（約292行目）は**旧version 1〜4のQRを復元する互換処理**です。新旧QRで希望値の意味をそろえる必要がある変更では、両方を同じ規則へ合わせます。

NPU値を0から80へ広げる等、顧客ベクトルの許容値を変える場合は、顧客の `bounds`、スタッフの `normalizeMatchValue()`、`npuTOPS` 上限（顧客式の上限・スタッフの `validateVector()`・フォーム `FIELDS`）、基準書、Firestore Rulesを一括確認します。商品登録フォームの `FIELDS` は現在0〜200 TOPSを許容しますが、推薦上限は0〜60です。200を変更する場合も、データを壊さないかRulesと既存Firestore文書を確認してください。

## 5. フォーム、項目順、質問コードの注意

スタッフの商品登録フォームは `FIELDS`（約230行目）で表示ラベル・単位・入力型・最小／最大／刻みを定義し、`validateVector()`（約278行目）で保存前の形式を検証します。値の範囲だけでなく**登録時の許容範囲も変える**場合は、両方を確認してください。

Firestoreの項目キーは両ページの `fieldKeys`／`vectorKeys` で定義されています。スタッフのCSV列は `HEADER` が `model, jan, FIELDSの順序, highlights` から作られます。項目名や順序を変えると、Firestore Rules、旧データ互換、CSV、推薦比較、仕様書に影響します。通常の微調整では変更しないでください。

質問文・選択肢を変更する場合は、顧客側 `questions` と `QR_ANSWER_CODES`、スタッフ側 `QR_QUESTIONS` を一致させます。選択肢の内部番号を変えると、過去QRの回答解釈が変わります。単なる重み・正規化範囲の変更では `QR_SCHEMA_VERSION`（現在5）を変更する必要はありません。ベクトル順・単位・回答コードの意味を変更する場合は、versionを上げ、スタッフの `parseQr()` と旧version変換、基準書まで更新します。QRは従来どおり数値30項目を維持します。

## 6. 推奨する作業順

1. 変更目的を1つに絞ります（例：ゲーム希望時にGPUの差を大きく反映する、TOPS上限を現カタログに合わせる）。
2. まず添付・登録予定の製品データの単位と範囲を確認し、変更前後の代表機種を決めます。
3. 該当する識別子を顧客・スタッフの両方で検索し、同期して変更します。質問からの需要式は顧客側が主、旧QR変換が関係する場合はスタッフ側も更新します。
4. [ベクトル設定基準書](選択肢からのベクトル設定基準.md)の同じ範囲・式・重み・QR互換説明を更新します。
5. 14項目の順序、配列の要素数、Firestoreフィールド名、CSV列順が変わっていないか照合します。
6. Q2/Q3の組合せ、メーカー候補不足、Q14の故障除外、画面比率、RAM16/32GBの境界、CPU低・中・高需要とpositioning回帰を含む代表回答で候補条件・スコアを確認し、客用3機種・スタッフ5機種の順位を同一カタログで照合します。
7. 変更後にプロジェクトルートで以下を実行し、エラーがないことを確認します。

```bash
pnpm check
pnpm build:static
git diff --check
```

8. Firestore本番データを書き換えず、CSV fixtureやローカルの模擬データで比較確認します。Firestore Rulesやスキーマまで変える場合は、Rulesのテストと承認済みの手順を別途確認してから公開します。

## 7. 最低限のレビュー項目

- 顧客 `bounds` とスタッフ `MATCH_RANGES` の範囲が同じ（CPU single 310–492、multi 1420–2850、RAM 0–32、positioning 0–10）で、成分順が14項目定義と一致する。
- 顧客・スタッフの `MATCH_WEIGHTS` が14個で同じ値・同じ順序、`MATCH_SCORE_INDICES` が同じ12項目、有効重み合計が9.35になっている。
- `makeVector()` が出すCPU/GPU/NPU/RAM/SSD/重量の値がFirestore商品と同単位で、CPU範囲・ユーザー編集GPU値・positioning回帰が一致する。RAMは常に16GB以上、条件時32GBで、必要量以上は不利にならない。
- 画面比率はカテゴリ一致/不一致としてスコアに使い、ハード除外しない。メーカーと故障頻度はスコアに含めず、メーカー優先/補完とQ14の候補除外条件へ分離している。
- 質問の選択肢順やQR回答コードを変えた場合、スタッフで旧QRも正しく読める。
- `FIELDS`／`fieldKeys`／CSV `HEADER`／Firestore Rulesのキーが一致し、既存値やJAN文字列を壊さない。
- 同一回答・同一カタログで顧客とスタッフの順位/一致度が同じである（表示上限は顧客3件、スタッフ5件）。条件候補が不足する場合は条件を緩和せず、不足数だけ表示する。
