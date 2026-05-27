# Text-to-Audio（TTA）生成AIの種類まとめ

## 1. このドキュメントの目的

このドキュメントは、効果音（SE）・環境音・BGMなどを生成する **Text-to-Audio（TTA）生成AI** について、今回の議論内容を整理したものです。

特に、手描きアニメや短編動画制作で使うことを想定し、次の観点で整理します。

- TTAとは何か
- TTSとの違い
- 代表的なモデル・サービス
- ローカル実行かクラウド利用か
- パラメータ数
- 個人制作での使いやすさ

---

## 2. TTAとは何か

**TTA（Text-to-Audio）** は、テキストで書いた指示から音声素材を生成する技術です。

たとえば、次のようなプロンプトを入力します。

```text
A rusty wooden door slowly creaking open in an abandoned house, dark horror atmosphere, close perspective, 2 seconds, no music, no voice
```

すると、AIが「古い木のドアがギィ……と開く音」のような効果音を生成します。

---

## 3. TTSとの違い

| 種類 | 入力 | 出力 | 主な用途 |
|---|---|---|---|
| TTS | テキスト | 人の声の読み上げ | ナレーション、AI音声会話、読み上げ |
| TTA | テキスト | 効果音・環境音・音楽など | SE、BGM、 Foley、環境音、演出音 |

### イメージ

```text
TTS:
「こんにちは」
  ↓
人間の声で「こんにちは」と読み上げる

TTA:
「雨の夜、古い街灯の下で足音が響く」
  ↓
雨音、足音、環境音を生成する
```

---

## 4. LLMとTTAの関係

通常のLLMは、音そのものを直接生成するのではなく、**音の設計やプロンプト作成**に向いています。

### LLMが得意なこと

- シーンに必要なSEを洗い出す
- SEのプロンプトを作る
- 音の長さ、距離感、質感、感情を言語化する
- Premiere ProやDaVinci Resolveに配置するための音響設計を考える

### TTAモデルが担当すること

- 実際の効果音を生成する
- 環境音を生成する
- 短い音楽・ループ素材を生成する

### 制作フロー例

```text
シーン内容をLLMに説明する
  ↓
必要なSEリストを作る
  ↓
各SEのプロンプトを作る
  ↓
TTAモデルで音を生成する
  ↓
動画編集ソフトで配置・調整する
```

---

## 5. 代表的なTTAモデル・サービス比較

| モデル / サービス | ローカル / クラウド | 主な用途 | パラメータ数 | 特徴 | 個人制作でのおすすめ度 |
|---|---|---|---:|---|---:|
| ElevenLabs Sound Effects | クラウド / API | SE、環境音、動画用効果音 | 非公開 | Web/APIで使いやすい。SE生成に実用寄り | ★★★★★ |
| Stable Audio 3.0 Small SFX | ローカル可 | 効果音、短い環境音 | 約0.6B規模 | SE向けのオープンウェイトモデル | ★★★★☆ |
| Stable Audio 3.0 Small Music | ローカル可 | 短い音楽、ループ素材 | 約0.6B規模 | 音楽・ループ向け | ★★★★☆ |
| Stable Audio 3.0 Medium | ローカル可 | 長めの音楽、構造のある音 | 約1.4B規模 | より長く構造的な音楽生成向け | ★★★★☆ |
| Stable Audio 3.0 Large | Stability AI API中心 | 高品質な音楽・音声生成 | 約2.7B規模 | API提供中心。大きめのモデル | ★★★★☆ |
| Stable Audio Open 1.0 | ローカル可 | SE、環境音、短い音素材、音楽素材 | 約1.21B | 最大47秒、44.1kHzステレオ生成 | ★★★★☆ |
| AudioLDM 2 | ローカル可 | SE、環境音、音楽、音声 | Large系で約1.5B前後の記載あり | 研究・検証向き。Diffusers対応 | ★★★☆☆ |
| AudioLDM 初代 | ローカル可 | SE、環境音、音楽 | Small: 181M / Large: 739M | 比較的軽めだが、やや古い | ★★☆☆☆ |

※ パラメータ数は公開情報ベースです。サービス型モデルは非公開の場合があります。

---

## 6. クラウド型とローカル型の違い

| 観点 | クラウド型 | ローカル型 |
|---|---|---|
| 代表例 | ElevenLabs Sound Effects | Stable Audio 3.0、Stable Audio Open、AudioLDM |
| 初期導入 | 簡単 | 環境構築が必要 |
| 料金 | 月額・クレジット課金が多い | モデル自体は無料利用可能なものが多い |
| 実行コスト | サービス料金 | 自分のPCの電気代・GPU・ストレージ |
| 品質 | 安定しやすい | モデルや環境に依存 |
| 商用利用 | プランや規約を確認 | ライセンスを確認 |
| カスタマイズ性 | 低〜中 | 高い |
| 個人制作の手軽さ | 高い | 中〜高 |

---

## 7. 目的別おすすめ

| 目的 | おすすめ |
|---|---|
| まずSE生成を試したい | ElevenLabs Sound Effects |
| 手軽に高品質なSEを作りたい | ElevenLabs Sound Effects |
| ローカルで無料に近い形で試したい | Stable Audio 3.0 Small SFX |
| ComfyUIでノードベースに扱いたい | Stable Audio 3.0 Small SFX |
| 短いBGMやループも作りたい | Stable Audio 3.0 Small Music |
| 長めのBGMを作りたい | Stable Audio 3.0 Medium |
| 研究・技術検証したい | AudioLDM 2 / Stable Audio Open |

---

## 8. アニメ制作での使い方

### 例：高級レストランのコメディシーン

```text
シーン:
主人公が高級レストランで財布を忘れたことに気づく。

必要なSE:
1. 心臓がドクンと鳴る音
2. グラスの氷がカランと揺れる音
3. 周囲の会話が一瞬遠のく環境音
4. コミカルな「ガーン」という衝撃音
5. 冷や汗が垂れるような小さな効果音
```

### 生成プロンプト例

```text
A short comedic shock sound effect, cartoon style, sharp impact, exaggerated, 1 second, no music, no voice
```

```text
A single heartbeat thump, tense and dramatic, close perspective, 1 second, no music, no voice
```

```text
A small ice cube clinking inside a glass in a quiet restaurant, close-up sound, 2 seconds, no music, no voice
```

---

## 9. 実用上の注意点

TTAモデルは便利ですが、狙った音が一発で出るとは限りません。

実制作では、次のような流れが現実的です。

```text
AIで複数パターン生成
  ↓
良い素材を選ぶ
  ↓
Audacity / Premiere Pro / DaVinci Resolve などで編集
  ↓
音量、長さ、ピッチ、リバーブを調整
  ↓
映像に配置
```

特にアニメ向けのSEでは、生成した音をそのまま使うより、**編集ソフトで少し誇張する**方が使いやすいです。

---

## 10. まとめ

- TTAは、テキストから効果音・環境音・音楽を生成する技術
- TTSは人の声の読み上げ、TTAはSEやBGM生成
- LLMは音そのものではなく、SE設計やプロンプト作成に向いている
- 実用重視なら ElevenLabs Sound Effects が使いやすい
- ローカルで試すなら Stable Audio 3.0 Small SFX が有力
- ComfyUIと組み合わせるなら Stable Audio 3.0 系が相性が良い
- 生成後は動画編集ソフトで調整する前提にすると実用性が高い

---

## 参考リンク

### ElevenLabs

- ElevenLabs Sound Effects 公式ページ  
  https://elevenlabs.io/sound-effects

- ElevenLabs Sound Effects API Documentation  
  https://elevenlabs.io/docs/overview/capabilities/sound-effects

- ElevenLabs Pricing  
  https://elevenlabs.io/pricing

- ElevenLabs Sound Effects Cost Help  
  https://help.elevenlabs.io/hc/en-us/articles/25735337678481-How-much-does-it-cost-to-generate-sound-effects

### Stability AI / Stable Audio

- Stable Audio 3.0 公式発表  
  https://stability.ai/news-updates/meet-stable-audio-3-the-model-family-built-for-artistic-experimentation-with-open-weight-models

- Stable Audio 公式ページ  
  https://stability.ai/stable-audio

- Stable Audio Open 1.0 Hugging Face  
  https://huggingface.co/stabilityai/stable-audio-open-1.0

- Stable Audio 3.0 Medium Hugging Face  
  https://huggingface.co/stabilityai/stable-audio-3-medium

- Stable Audio 3.0 Small SFX Hugging Face  
  https://huggingface.co/stabilityai/stable-audio-3-small-sfx

- Stable Audio 3 Hugging Face Space  
  https://huggingface.co/spaces/stabilityai/stable-audio-3

### AudioLDM

- AudioLDM 2 公式ページ  
  https://audioldm.github.io/audioldm2/

- AudioLDM 2 Hugging Face  
  https://huggingface.co/cvssp/audioldm2

- AudioLDM 2 Diffusers Documentation  
  https://huggingface.co/docs/diffusers/en/api/pipelines/audioldm2

- AudioLDM 2 GitHub  
  https://github.com/haoheliu/audioldm2
