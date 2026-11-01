# pippin_protocol.md
# Atmark Pippin Controller（AppleJack）ADBプロトコル資料

2026-11-01 資料調査の結果をまとめたもの。一次情報は末尾の出典を参照。

## 1. 概要

| 項目 | 内容 | 出典 |
|---|---|---|
| デフォルトアドレス | `$3`（マウスと同じ） | [1]、実機確認済み（2026-11-01） |
| 初期ハンドラーID | `0x01`（Classic Mouse、100cpi） | [1]、実機確認済み（2026-11-01） |
| コントローラーモード | ハンドラーIDを`0x46`に設定すると、Talk 0の応答が4バイトに拡張される | [1] |
| 配線（ADB版） | 緑＝GND（Pin4）、青＝DATA（Pin1）、赤＝+5V（Pin3）、黒＝シールド、Pin2未接続 | [2]、配線色は実機確認済み |
| ADB版とAppleJack（Pippin版）の違い | 基板は同一。ケーブル（プラグ）のみ異なる | [2] |

## 2. マウスモード（ハンドラーID `0x01`）

Talk 0に2バイトで応答する（データが変化したときのみ）。

| Bit | 内容 | 値 |
|---|---|---|
| 15 | 左ショルダー（左トリガー）ボタン | 0＝押下、1＝離し |
| 14-8 | トラックボールY移動量 | 2の補数。負＝上、正＝下 |
| 7 | 右ショルダー（右トリガー）ボタン | 0＝押下、1＝離し |
| 6-0 | トラックボールX移動量 | 2の補数。負＝左、正＝右 |

- それ以外のボタン（十字キー・色ボタン等）を押す／離すと、データの変化として2バイトを返すが、**内容には反映されない**（実機の`80 80`と一致）

## 3. コントローラーモード（ハンドラーID `0x46`）

Talk 0に4バイトで応答する。ボタンはすべて **0＝押下、1＝離し**。

| Byte | Bit | 内容 |
|---|---|---|
| 0 | 31 | 左ショルダー（左トリガー）ボタン |
| 0 | 30-24 | トラックボールY移動量（2の補数、負＝上、正＝下） |
| 1 | 23 | 右ショルダー（右トリガー）ボタン |
| 1 | 22-16 | トラックボールX移動量（2の補数、負＝左、正＝右） |
| 2 | 15 | 緑（1）ボタン |
| 2 | 14 | 赤（2）ボタン |
| 2 | 13 | 十字キー 下 |
| 2 | 12 | 十字キー 右 |
| 2 | 11 | 十字キー 左 |
| 2 | 10 | 十字キー 上 |
| 2 | 9 | 黄（4）ボタン |
| 2 | 8 | 青（3）ボタン |
| 3 | 7-3 | 常に1 |
| 3 | 2 | ◇（ダイヤ）ボタン |
| 3 | 1 | ○（丸）ボタン |
| 3 | 0 | □（四角）ボタン |

- Byte 0〜1は、マウスモードの2バイトと同じ形式
- 何も押していないときの想定値：Byte 2＝`FF`、Byte 3＝`FF`（Bit 7-3は常に1、ボタンはすべて1）

## 4. ボタン一覧（Apple技術資料の名称との対応）

| 本資料の名称 | Apple技術資料[3]の名称 | 位置 | Pippin ROMの初期割り当て[3] |
|---|---|---|---|
| 左ショルダー | S1 Left Trigger／Left Fire | 本体下面 | マウスボタン |
| 右ショルダー | S2 Right Trigger／Right Fire | 本体下面 | マウスボタン |
| 十字キー 上・左・右・下 | S3〜S6 Directional Pad | 左側 | 矢印キー |
| 緑 | S7 | 右側（Primary Action） | K |
| 赤 | S8 | 右側（Primary Action） | F8 |
| 青 | S9 | 右側（Primary Action） | F14 |
| 黄 | S10 | 右側（Primary Action） | J |
| □・○・◇ | S11〜S13（Secondary Action） | 手前側 | F11・F12・F13 |

- 十字キーのS3〜S6と各方向の対応は、技術資料の図からは確定できない（本プロジェクトでは[1]のビット定義に従えば不要）

## 5. TMKファームウェアとの関係（`adb_usb/matrix.c`）

- 現行の`mouse_setup()`は、ハンドラー`0x01`の機器をExtended Mouse（`0x04`）等へ切り替えようとするが、Pippinは応じないためClassic（`0x01`）のまま動作する（実機確認済み）
- 現行の`mouse_proc()`はTalk 0を最大5バイト読み、Apple Extended Mouse形式として解釈する。`0x46`の4バイト形式をそのまま通すと、Byte 2〜3（ボタン）が移動量の上位ビット等として誤って解釈されるため、**専用の分岐が必要**
- TMK作者による既知情報：Pippinはマウスモードで2つ目のボタン（右ショルダー）を正しく通知するが、TMKの修正（ADB_MOUSE_2ND_BUTTON_QUIRK関連、コミット`0c25a46d`）で2つ目のボタンが無効化される[4]

## 6. 実機との差異（要確認）

- 資料[1]ではマウスモードでもトラックボールの移動量が返るはずだが、実機（2026-11-01）ではトラックボール操作時も`80 80`（移動量0）だった。`0x46`切り替え後の調査用ファームウェアで再確認する

## 出典

1. lampmerchant/tashnotes「Atmark Pippin Controller」（最終更新 2024-06-05、commit `61ac3721`）
   https://github.com/lampmerchant/tashnotes/blob/main/macintosh/adb/protocols/atmark_pippin_controller.md
2. retrostuff.org「Pippin Atmark ADB controller」 https://retrostuff.org/?p=4315
3. Apple Computer「Applejack Input Device Driver」Pippin Technical Notes PPBD-JHACEF003（1996-04-11）
   http://absurdengineering.org/library/MASTER%20Developer%20Notes/Pippin/Applejack%20Tech%20Note.pdf
4. tmk/tmk_keyboard Issue #724 https://github.com/tmk/tmk_keyboard/issues/724
5. 参考：ato-net.jp「アットマークワイヤレスコントローラの改造」 http://ato-net.jp/column/pippicon/pippicon.html （Mac接続時はトラックボールとトリガーがマウスとして動作、配線色は赤・青・緑）
