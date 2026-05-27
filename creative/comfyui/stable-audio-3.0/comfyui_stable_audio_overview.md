# ComfyUI × Stable Audio 3.0 概要まとめ

## 1. このドキュメントの目的

このドキュメントは、**Stable Audio 3.0 を ComfyUI で使う場合の概要**を整理したものです。

特に、次の用途を想定しています。

- 手描きアニメ用の効果音（SE）を作る
- 短い環境音を作る
- 短いBGMやループ素材を作る
- ノードベースで音声生成ワークフローを組む

---

## 2. ComfyUIとは

**ComfyUI** は、ノードをつなげて画像生成・動画生成・音声生成などのワークフローを作れるツールです。

もともとは Stable Diffusion 系の画像生成でよく使われていましたが、現在は動画生成・音声生成にも対応範囲が広がっています。

---

## 3. Stable Audio 3.0とは

**Stable Audio 3.0** は、Stability AI が公開した音声生成モデルファミリーです。

テキストプロンプトから、次のような音を生成できます。

- 効果音
- 環境音
- 短いループ音楽
- 長めの音楽
- 音声素材の編集・継続生成

Stable Audio 3.0 は、Small / Medium / Large などのモデルサイズで構成されています。

---

## 4. ComfyUIで使えるStable Audio 3.0系モデル

| モデル | 主な用途 | ローカル実行 | パラメータ規模 | 向いている用途 |
|---|---|---|---:|---|
| Stable Audio 3.0 Small SFX | 効果音・短い環境音 | 可 | 約0.6B規模 | アニメSE、UI音、短い環境音 |
| Stable Audio 3.0 Small Music | 短い音楽・ループ | 可 | 約0.6B規模 | 短いBGM、ループ、ジングル |
| Stable Audio 3.0 Medium | 長めの音楽・構造的な音 | 可 | 約1.4B規模 | 長めのBGM、より複雑な音楽 |
| Stable Audio 3.0 Large | 高品質生成 | API中心 | 約2.7B規模 | 商用API利用、高品質生成 |

アニメ用SEが目的なら、最初に試すべきは **Stable Audio 3.0 Small SFX** です。

---

## 5. ComfyUI × Stable Audio 3.0 の基本構成

概念的には、次のようなワークフローになります。

```text
テキストプロンプト
  ↓
Stable Audio 3.0 ノード
  ↓
音声生成
  ↓
プレビュー / 保存
  ↓
Premiere Pro などで編集
```

もう少し具体的にすると、次のような流れです。

```text
Prompt入力
  ↓
モデル読み込み
  ↓
サンプラー / 生成設定
  ↓
音声データ生成
  ↓
MP3 / WAV などで保存
```

---

## 6. アニメSE生成でのおすすめ構成

| 用途 | 推奨モデル | 理由 |
|---|---|---|
| 短い効果音 | Stable Audio 3.0 Small SFX | SE向けで軽い |
| ドア、足音、衝撃音 | Stable Audio 3.0 Small SFX | 短尺SEに合う |
| 雨音、街中、レストランの環境音 | Stable Audio 3.0 Small SFX / Medium | 短ければSmall SFX、長めならMedium |
| 短いBGM | Stable Audio 3.0 Small Music | ループ・短い曲向け |
| 長めのBGM | Stable Audio 3.0 Medium | 構造のある音楽向け |

---

## 7. 手描きアニメ制作との組み合わせ例

### 例：財布を忘れたことに気づくシーン

```text
映像:
主人公が高級レストランで財布を忘れたことに気づく。

必要な音:
- 心臓のドクン音
- グラスの氷がカランと鳴る音
- 一瞬空気が止まるような環境音
- コミカルなショック音
```

### ComfyUIでの生成イメージ

```text
Prompt 1:
A single dramatic heartbeat thump, tense, close perspective, 1 second, no music, no voice

Prompt 2:
Ice cubes clinking in a wine glass in a quiet luxury restaurant, close-up, 2 seconds, no music, no voice

Prompt 3:
A short comedic shock impact sound, cartoon style, exaggerated, 1 second, no voice, no music
```

### 編集フロー

```text
ComfyUIでSE生成
  ↓
良いSEを選ぶ
  ↓
Premiere Proに読み込む
  ↓
映像タイミングに合わせる
  ↓
音量・ピッチ・長さを調整
  ↓
完成
```

---

## 8. ComfyUIで使うメリット

| メリット | 内容 |
|---|---|
| ノードベースで見える | 生成処理の流れが視覚的にわかる |
| ワークフローを保存できる | 同じ設定で再生成しやすい |
| 画像・動画生成と統合しやすい | アニメ制作の素材生成と相性がよい |
| ローカル実行しやすい | API課金なしで検証できる |
| 試行錯誤しやすい | プロンプトや長さを変えて複数生成できる |

---

## 9. 注意点

### 9.1 ComfyUIの更新が必要

Stable Audio 3.0 を使うには、ComfyUIを最新版に更新する必要があります。

Gitで導入している場合は、通常はComfyUIフォルダで次のように更新します。

```bash
git pull
```

ただし、拡張機能やカスタムノードを多く入れている場合は、更新前にバックアップを取るのが安全です。

---

### 9.2 モデルファイルの配置が必要

Stable Audio 3.0 のモデルは、Hugging Faceなどから取得し、ComfyUIが読み込める場所に配置する必要があります。

正確な配置場所は、ComfyUIのStable Audio対応ワークフローや使用するノードの説明に従ってください。

---

### 9.3 生成結果は編集前提で考える

生成AIのSEは、一発で完全に狙った音になるとは限りません。

実制作では、次のような調整が必要になることがあります。

- 長さ調整
- 音量調整
- ピッチ変更
- フェードイン / フェードアウト
- リバーブ追加
- ノイズ除去
- 複数音の重ね合わせ

---

## 10. Stable Audio 3.0 Small SFXを使う場合の考え方

アニメSE目的では、最初から長い音を作るより、**短い音を複数作って編集で組み合わせる**方が扱いやすいです。

### 推奨方針

| 方針 | 理由 |
|---|---|
| 1〜3秒程度のSEから作る | 映像に合わせやすい |
| no music, no voice を入れる | 余計な音楽や声を避けやすい |
| close perspective など距離感を書く | 音の印象を制御しやすい |
| cartoon style / realistic など方向性を書く | アニメ調かリアル調かを指定できる |
| 複数回生成して選ぶ | 生成AIはばらつきがあるため |

### プロンプト例

```text
A short cartoon slapstick impact sound, sharp and exaggerated, 1 second, no music, no voice
```

```text
A small object drops onto a wooden restaurant table, close perspective, realistic, 1 second, no music, no voice
```

```text
A quiet luxury restaurant ambience, distant conversations, soft plates and glasses, 8 seconds, no music, no voice
```

---

## 11. ざっくり導入手順

環境によって細部は変わりますが、全体像は次の通りです。

```text
1. ComfyUIを最新版に更新する
  ↓
2. Stable Audio 3.0用のモデルを取得する
  ↓
3. ComfyUIの指定フォルダへモデルを配置する
  ↓
4. 公式または配布されているワークフローを読み込む
  ↓
5. プロンプトと長さを指定して生成する
  ↓
6. 生成した音声を保存する
  ↓
7. 動画編集ソフトに読み込む
```

---

## 12. ユーザーさんの用途でのおすすめ

手描きアニメのSE生成が目的なら、まずは次の組み合わせがおすすめです。

```text
ComfyUI
  +
Stable Audio 3.0 Small SFX
  +
Premiere Pro / Audacity
```

### 理由

- Stable Audio 3.0 Small SFX はSE向け
- ComfyUIでワークフロー化しやすい
- ローカル実行できるため試行錯誤しやすい
- 生成した音をPremiere Proで映像に合わせて編集できる

---

## 13. まとめ

- Stable Audio 3.0 は ComfyUI で利用できる
- SE目的なら Stable Audio 3.0 Small SFX が第一候補
- 短いBGMやループなら Small Music
- 長めの音楽なら Medium
- ComfyUIはノードベースでワークフローを保存できるため、制作向き
- 生成した音はそのまま使うより、Premiere ProやAudacityで編集する前提がよい

---

## 参考リンク

### ComfyUI

- ComfyUI 公式サイト  
  https://www.comfy.org/

- ComfyUI GitHub  
  https://github.com/comfyanonymous/ComfyUI

- Stable Audio 3.0 Day-0 Support in ComfyUI  
  https://blog.comfy.org/p/stable-audio-3-day-0-support

- ComfyUI Stable Audio 3.0 ワークフロー例  
  https://www.comfy.org/workflows/9c3c4722a8e1-9c3c4722a8e1/

### Stability AI / Stable Audio

- Stable Audio 3.0 公式発表  
  https://stability.ai/news-updates/meet-stable-audio-3-the-model-family-built-for-artistic-experimentation-with-open-weight-models

- Stable Audio 公式ページ  
  https://stability.ai/stable-audio

- Stable Audio 3.0 Medium Hugging Face  
  https://huggingface.co/stabilityai/stable-audio-3-medium

- Stable Audio 3.0 Small SFX Hugging Face  
  https://huggingface.co/stabilityai/stable-audio-3-small-sfx

- Stable Audio 3 Hugging Face Space  
  https://huggingface.co/spaces/stabilityai/stable-audio-3

### 参考記事

- ComfyUIでStable Audio 3.0を試した記事  
  https://note.com/kongo_jun/n/n2a8c7d79e6f6
