# Pippin_Atmark_Controller

## 概要

Apple Pippin @WORLD の純正コントローラー（ADB接続、製造元 Atmark 社）を、自作の ADB→USB 変換アダプタ（Arduino Pro Micro／ATmega32U4）を介して Linux で使用するためのプロジェクトです。

[TMK](https://github.com/tmk/tmk_keyboard) の `converter/adb_usb` ファームウェアに Pippin コントローラー専用の処理を追加し、トラックボールと全13ボタンを USB マウス＋キーボードとして使えるようにします。ボタンは Quake の初期設定のキーに割り当てているため、Quake（Quakespasm 等）を設定変更なしでコントローラーだけで遊べます。

### 仕組み

- コントローラーは接続直後、ADB アドレス `$3`・ハンドラーID `0x01`（Classic Mouse）として動作し、このモードでは十字キーや色ボタンの入力を区別できません
- 本ファームウェアはコントローラーを検出するとハンドラーIDを `0x46`（コントローラーモード）に切り替え、4バイトの入力データからトラックボール・全ボタンの状態を読み取ります
- データ形式の詳細は `pippin_protocol.md` を参照してください

## ファイル構成

| ファイル・フォルダ | 内容 |
|---|---|
| `adb_usb/` | TMK の ADB→USB 変換ファームウェア一式。`matrix.c` は Pippin 対応の改修済み版（基準：tmk_keyboard コミット `99c12879`）。ビルド済みバイナリは含まないため、下記の手順でビルドする |
| `patches/` | 改修内容のパッチ（`p2_pippin_probe.patch` → `e2a_pippin_mouse.patch` → `e2b_pippin_keys.patch` の順に適用） |
| `pippin_protocol.md` | Atmark Pippin Controller の ADB プロトコル資料（ハンドラーID `0x01`／`0x46` のデータ形式、ボタン一覧、出典） |
| `device_list.md` | 既知の ADB デバイス一覧（`lampmerchant/tashnotes` 由来の参考資料） |
| `購入リスト.md` | 必要な部材（Arduino Pro Micro、Mini-DIN コネクタ、抵抗等）の一覧 |
| `IIgs/` | 参考として同梱している別プロジェクト（Apple IIgs キーボード用 TMK 実装）のソース。本プロジェクトの対象ではない |

## 使い方

### 必要なもの

- Arduino Pro Micro（ATmega32U4、5V/16MHz 版）
- ADB 用 Mini-DIN 4pin コネクタ、1kΩ 抵抗
- Linux（動作確認：Linux Mint x86_64）。`gcc-avr avr-libc avrdude make git` をインストール

### 配線

| ADB コネクタ | 接続先 | コントローラー側の線の色 |
|---|---|---|
| Pin1（DATA） | Arduino D3（PD0） | 青 |
| Pin2（PSW） | 未接続 | — |
| Pin3（+5V） | USB の 5V（VBUS）から直接供給 | 赤 |
| Pin4（GND） | Arduino GND | 緑 |

- DATA 線と Arduino VCC の間に 1kΩ のプルアップ抵抗を外付けする（必須。内蔵プルアップでは弱い）
- Pro Micro の互換品は、ボード上の保護素子により VCC が約4Vまで下がるため、Pin3 にはボードを通る前の USB 5V をつなぐ（外部電源の GND は Arduino の GND と共通にする）
- コネクタの接触不良があると ADB 機器が一切応答しない。応答がない場合は、まずコネクタと配線の接触を確認する

### ファームウェアのビルド・書き込み

```bash
git clone https://github.com/tmk/tmk_keyboard.git
cd tmk_keyboard
git submodule update --init
git switch -c pippin 99c12879   # 改修の基準コミット
```

本リポジトリの `patches/` の3ファイルを順に `git apply` するか、本リポジトリの `adb_usb/matrix.c` を `converter/adb_usb/matrix.c` に上書きします（改修後の `matrix.c`：988行、MD5 `ba4c1e9cd79e6c275f69e289c43017f1`）。

```bash
cd converter/adb_usb
make -f Makefile.rev1   # ATmega32U4 向け。無印の Makefile は ATmega32U2 向けなので使用しない
```

生成された `adb_usb_rev1.hex` を、Pro Micro をブートローダーモード（RST と GND を短絡）にした状態で書き込みます。

```bash
avrdude -p atmega32u4 -c avr109 -P /dev/ttyACM0 -U flash:w:adb_usb_rev1.hex
```

### ボタンの割り当て

| コントローラー | USB 出力 | Quake での動作（初期設定） |
|---|---|---|
| トラックボール | マウス移動 | 視点操作 |
| 左ショルダー（下面） | マウス左ボタン | 攻撃 |
| 右ショルダー（下面） | マウス右ボタン | 前進 |
| 十字キー 上・下 | ↑・↓ | 前進・後退 |
| 十字キー 左・右 | `,`・`.` | 左右に横移動 |
| 赤 | Ctrl | 攻撃 |
| 緑 | Space | ジャンプ |
| 黄 | `/` | 次の武器 |
| 青 | Shift | 走る |
| □ | Esc | メニュー |
| ○ | Enter | メニューの決定 |
| ◇ | なし | — |

割り当てを変更する場合は、`adb_usb/matrix.c` の `pippin_keycode[]`（ADB キーコードの表）を編集します。

### 動作確認

```bash
lsusb            # USB デバイス（feed:0adb、t.m.k.）として認識されているか確認
sudo evtest      # 入力イベントの確認
sudo hid_listen  # ファームウェアのデバッグ出力の確認
```

- `hid_listen` を先に起動してから変換アダプタの USB を挿すと、起動時のログを確認できます。正常であれば `M:$F:Pippin` と、機器一覧に `F:46 3:01` が表示されます
- `evtest` でボタンを確認するときは、`sudo evtest --grab /dev/input/eventN`（`N` は `t.m.k.` のキーボードの番号）とすると、入力がターミナルに送られません

### Quake で使う

- Quakespasm 等のソースポートとゲームデータ（`id1/pak0.pak` 等）を用意し、初期設定のまま起動します
- トラックボールの上下で視点が上下に動かない場合は、メニューの Options → Mouse Look を On にします  

### In Background

I connected a Bandai/Apple **Pippin atmark** controller to a modern PC. The adapter is an **Arduino Pro Micro** with one resistor and an ADB connector, running the **TMK ADB-to-USB** firmware.

At first the controller returned `80 80` for every input. The cause: it identifies itself as an ordinary mouse until you switch its ADB **handler ID to 0x46**, after which it acts as a game pad and each button returns a distinct value.

The host is a 2006 **Dell Precision M90** running Linux Mint XFCE, themed to look like Mac OS 9. The controller is mapped to Quake's default keys and played through **QuakeSpasm**.

My firmware changes are on [GitHub](https://github.com/dokachin/Pippin_Atmark_Controller). Fittingly, both the Pippin and Quake were released in 1996.

Full write-up (Japanese): https://dokachin.hatenablog.jp/entry/2026/11/02/165514
