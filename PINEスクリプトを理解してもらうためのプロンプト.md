# Pine Script v6　 ハマり集

**コード書く前に必ず一読すること。** 過去ハマったポイントを全部記録する

---

## 構文系

### ❌ セミコロンによる複数文記述
```pine
// NG: "no viable alternative at character ';'" エラー
p1 = arr.get(n-3); h1 = isH.get(n-3)
```
```pine
// OK: 1行1文
p1 = arr.get(n-3)
h1 = isH.get(n-3)
```
Pine Scriptはセミコロンを文区切りとして認識しない。**改行で分ける**。

### ❌ 関数からの外スコープ変数書き換え
```pine
// NG: lMtaは関数のローカルコピーで、外スコープのlMtaは変わらない
drawMta(price) =>
    lMta := line.new(...)  // 外のlMtaは更新されない
```
```pine
// OK1: インラインで書く
if condition
    lMta := line.new(...)

// OK2: UDTのメソッドで書く
type State
    line lMta
method draw(State this, price) =>
    this.lMta := line.new(...)  // UDTフィールドは書ける
```

### ❌ 削除後のline/labelをna判定
```pine
// NG: deleteしてもlMtaはna判定ではfalseのまま
line.delete(lMta)
if not na(lMta)  // true (削除されたlineオブジェクトの参照が残ってる)
    line.delete(lMta)  // ダブル削除でエラー
```
```pine
// OK: deleteしたら必ず := na で無効化
line.delete(lMta)
lMta := na
```

### ❌ 水平線でx1==x2
```pine
// NG: 点になってextend.rightしても線にならない
line.new(bar_index, mta, bar_index, mta, extend=extend.right)
```
```pine
// OK: 1バーずらす
line.new(bar_index, mta, bar_index + 1, mta, extend=extend.right)
```

### ❌ 逆順ループの`by -1`（ランタイムエラー RE10021）
```pine
// NG: ランタイムで "'step' in loop must be greater than zero" エラー
// Pine v6では `by` の値は正の整数のみ受け付ける
for i = n-1 to 0 by -1
    ...
```
```pine
// OK1: from > to なら自動で逆順になる（`by`省略）
for i = n-1 to 0
    ...
```
```pine
// OK2: 前方走査して最後にマッチしたindexを使う（直近の特徴ピボットを探す等）
result = -1
for i = 0 to arr.size() - 1
    if arr.get(i) > someThreshold
        result := i  // 最後にマッチしたものが残る = 直近
```
**重要**: pine_checkは通っても**ランタイムで死ぬ**。実機チャートで確認するまで完了とみなさない。

### ❌ 空配列のループ
```pine
// 危険: arr.size()==0 のとき "for i = 0 to -1" になる
for i = 0 to arr.size() - 1
    ...
```
```pine
// OK: ガード追加
if arr.size() > 0
    for i = 0 to arr.size() - 1
        ...
```

---

## ライブラリ系

### ZigZag v7（TradingView公式）
**⚠️ ピボット取得APIは公開されていない**（2026-05-16 `pine_check`で確認）。
- `getLastPivots()` も `getPivots()` も `.pivots` フィールドも**存在しない**
- このライブラリは **描画専用**。プログラム的にピボットを使いたい場合は **`ta.pivothigh()` / `ta.pivotlow()` で自前検出**
- 公開API: `Settings.new(...)`, `newInstance(settings)`, `instance.update()` のみ
- 自前検出時の交互保証パターンは `docs/zigzag-api.md` 参照

### 🔥 Pineの `or` / `and` は短絡評価しない（RE10108等）
**最重要級の罠。** 他言語と違い、Pineは `a or b` / `a and b` の**両辺を必ず評価**する。
```pine
// NG: size==0 でも zB.last() が評価され "Cannot call last() if array is empty"
if zP.size() == 0 or zB.last() != pBar
    ...
```
```pine
// OK: ネストif で空配列時に last() を呼ばせない
addIt = false
if zP.size() == 0
    addIt := true
else if zB.last() != pBar
    addIt := true
if addIt
    ...
```
配列の `.get()/.last()`、`array`/`map` アクセス、na前提の除算等を
`or`/`and` の右辺に置くと**ガードが効かずランタイムで死ぬ**。
ガードは必ず**ネストif**で書く。pine_checkは通るのでブラウザ実機まで気づけない。

### 🔥🔥 最重要: `for i = a to b` は a>b で「逆走」する（OOBの主犯）
Pineの`for`は **start>end のとき空ループにならず逆順に回る**。
```pine
// n=7, bIdx=6 → for j = 7 to 6 は j=7 から逆走
for j = bIdx + 1 to n - 1
    array.get(zH, j)   // j=7, size 7 → RE10045 "index 7, size 7"
```
RE10045「index N, size N」の典型パターンはこれ（j=N=size に到達）。
**必ず start<=end をガード**してからループに入る：
```pine
if bIdx + 1 <= n - 1
    for j = bIdx + 1 to n - 1
        ...
// 内側も同様: for j = bIdx+1 to dIdx-1 の前に if dIdx > bIdx + 1
```
`n=min(size)`等の境界対策をしても、**ループが逆走したら全部無意味**。
原因切り分けで最後まで見落としやすい。複数indexでループ境界を作る時の鉄則。

### ライブラリ返却UDTの履歴参照 `(libObj[1]).field` はランタイム破壊
ライブラリ（DevLucem/ZigLib等）が返す UDT/chart.point を直接
`(z2[1]).price` の形で**履歴参照すると、ライブラリ内部の遅延評価が
狂って無関係な行で RE10045（array.get OOB）**を誘発する。pine_checkは通る。
同じライブラリでも履歴参照しないスクリプト（ZigZag++等）は無事＝差分はこれ。
対策: ライブラリ出力の必要値を**毎バー普通の float/int 系列に退避**してから
`series[1]` 参照する：
```pine
zpx = z2.price        // 退避
zix = z2.index
... pPrice = zpx[1]    // UDTでなくfloat系列を履歴参照
```

### pine_check が通っても RE は出る（コンパイル≠ランタイム）
`pine_check`/`pine_compile` は**構文・型のみ**検証。配列範囲(RE10045)・
step>0(RE10021)・null参照などの**実行時エラーは検出不能**（バーを
実行しないため）。「チェックOKなのにエラー」は正常。実機ロード必須。

### UDTフィールドの履歴参照（CE）
ライブラリが返す `chart.point` 等のUDTオブジェクトのフィールドに `[1]` を直接付けると失敗：
```pine
// NG: "Cannot use the history-referencing operator on fields of user-defined types"
prev = z2.price[1]
```
```pine
// OK: オブジェクトを先に履歴参照し、その後フィールド
prev = (z2[1]).price
```

### 🔥 チャートのスタディは「古いコンパイル済み」が残る
`pine_set_source`+保存は **Pineエディタ** を更新するだけ。ユーザーが
チャートに乗せた **スタディ（インジ本体）は別インスタンス** で、
**削除→再追加するまで古いコンパイル済みコードが走り続ける**。
→ ランタイムエラーが「直したのに消えない」時はこれ。**毎回スタディを
一旦削除→マイスクリプトから再追加**させる。エディタの行番号と実機の
エラー行(`#main():NNN`)が食い違う＝古い版が動いてる動かぬ証拠。

### 配列indexの防御ガード（var indexは必ず境界チェック）
`var int aIdx` 等でindexを跨バー保持する設計は、配列サイズとズレると
`array.get index out of bounds`。アクセス前に必ず
`idx >= 0 and idx < array.size(a)` を確認し、外れたら安全に状態リセット。

### 🔥 並走する複数配列は size の「最小値」をループ境界に
zP/zH/zB を毎バー一緒に push していても、ライブラリ（DevLucem/ZigLib等）
が絡むと**実行時に一瞬サイズがズレて RE10045**（"index N, size N"）。
静的解析では等しく見えても起きる。対策は決定版：
```pine
n = math.min(array.size(zP), math.min(array.size(zH), array.size(zB)))
```
以降の全ループ/アクセスを `n` 基準にすれば、どの配列でも絶対OOBしない。
pine_checkは通るので実機でしか出ない。複数配列を index 同期運用する時の鉄則。

### v5ライブラリは v5 スクリプトで使う（条件評価の不整合）
v5ライブラリを v6 スクリプトで import すると
「v5 library ... evaluates conditional logic differently than v6 ... might cause
inconsistent logic evaluation」警告。**ロジックがズレる実害あり**（短絡有無等）。
→ ライブラリのバージョンに**スクリプト本体を合わせる**（v5ライブラリ→`//@version=5`）。
v5では配列は関数形 `array.push(a,x)/array.size(a)/array.get(a,i)/array.last(a)` が安全
（メソッド形 `a.push()` は版差リスク）。

### DevLucem/ZigLib/1（MT4式ZigZag、ピボット取得可能）
TradingView公式ZigZagと違い**ピボットが取れる**：
```pine
import DevLucem/ZigLib/1 as ZigZag
[direction, z1, z2] = ZigZag.zigzag(low, high, Depth, Deviation, Backstep)
// direction反転 = 確定ピボット。(z2[1]) がその確定点。
// direction[1]>0 → 確定高値 / <0 → 確定安値
// z1/z2 は chart.point（.price / .index あり、v6でimport可）
```
ユーザーの「ZigZag++ [LD]」がこれ。設定 Depth/Deviation/Backstep をそのまま渡せば視覚完全一致。

### 配列削除（Pine v5の罠）
- Pine v5の `array.remove()` はランタイムエラーを出すケースあり
- v6では改善されてるが、削除は最小限に。pushとgetだけで済む設計を優先

---

## TradingView MCP

### ❌ capture_screenshotの濫用
- 1回あたり画像が大きく、API画像数制限を超える
- デバッグはコードに `barstate.islast` で `label.new()` を仕込む方が確実

### ✅ pine_check / pine_compile 必須
- コード書いたら必ず `pine_check` を回してエラーゼロにする
- ユーザーに見せる前にコンパイル通す

---

## パフォーマンス

### 配列の無制限成長
- `cpP.push()` を毎バー繰り返すと、ヒストリカルチャートで巨大化
- 必要なら直近N本（200程度）でキャップする

### 関数内の毎バーループ
- 確定ピボットリストに対して毎バーO(n)スキャンは、n<500なら許容
- 巨大配列なら early-break / 二分探索を検討

---

## 過去事例

### MTA (2026-05-16)
- セミコロン複数文 → 8件のCE10005エラー → 1行1文で修正
- 関数で外スコープのlineを書き換えようとして失敗 → インライン化
- **ZigZag v7のgetLastPivots()は実は存在しなかった**（pine_checkで7件エラー） → `ta.pivothigh/pivotlow` + 自前交互保証に変更で解決
- `by -1` 逆順ループが**ランタイムRE10021**（compileは通る） → 前方走査方式へ
- **描画用ライブラリと判定ロジックで別ピボットを使うと挙動が乖離**（白ZigZagは速い/MTAは遅延で固着）→ 自前ZigZagで両方を1つの配列から駆動するのが正解
- 強トレンドでMTA固着 → 更新トリガを「新ピボット待ち」でなく「基準値をcloseで抜けたら」にすると即追従
