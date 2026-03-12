# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AudioSR (Versatile Audio Super-Resolution at Scale) - 低解像度の音声を48kHz高品質音声にアップスケールする拡散モデルベースのツール。音楽、音声、環境音など全種類の音声に対応。論文: arXiv:2309.07314

## Commands

```bash
# インストール (pip)
pip3 install audiosr==0.0.7
# または開発用
pip3 install -e .

# 単一ファイルの超解像
audiosr -i example/music.wav

# バッチ処理 (ファイルリスト)
audiosr -il batch.lst

# 長時間音声のチャンク処理
audiosr -i long_audio.wav --chunking --chunk_duration 15 --overlap_duration 2

# Gradio デモ
pip install -r requirements.txt
python app.py

# Cog (Replicate) での推論
python predict.py

# PyPI パッケージビルド
python3 setup.py sdist bdist_wheel
```

## Architecture

### パイプライン全体の流れ

入力音声 → ローパスフィルタでカットオフ検出 → メルスペクトログラム抽出 → VAE潜在空間エンコード → DDIM拡散サンプリング → HiFi-GANボコーダーでデコード → 48kHz WAV出力

### 主要モジュール構成

- **`audiosr/pipeline.py`** - メインのエントリポイント。`build_model()`, `super_resolution()`, `super_resolution_long_audio()` を提供。長時間音声はチャンク分割 + Hann窓クロスフェードで結合
- **`audiosr/utils.py`** - 音声I/O、メルスペクトログラム抽出(`wav_feature_extraction`)、正規化、ローパスフィルタ前処理、HuggingFaceからのチェックポイントダウンロード、モデル設定(`get_basic_config`)
- **`audiosr/lowpass.py`** - 多種フィルタ(butter/cheby1/ellip/bessel)によるローパス・バンドパスフィルタリング。STFT硬判定ローパスも含む
- **`audiosr/latent_diffusion/models/ddpm.py`** - `LatentDiffusion` クラス (DDPMベース)。`generate_batch()` で推論実行
- **`audiosr/latent_diffusion/models/ddim.py`** - DDIMサンプラー
- **`audiosr/latent_diffusion/modules/diffusionmodules/openaimodel.py`** - UNetモデル (条件付き拡散)
- **`audiosr/latent_encoder/autoencoder.py`** - VAE (AutoencoderKL) で音声をメルスペクトログラムから潜在空間にエンコード/デコード
- **`audiosr/hifigan/`** - HiFi-GAN ボコーダー (メルスペクトログラム → 波形変換)
- **`audiosr/clap/`** - CLAP (Contrastive Language-Audio Pretraining) エンコーダー
- **`audiosr/latent_diffusion/modules/audiomae/`** - AudioMAE エンコーダー

### エントリポイント

| ファイル | 用途 |
|---------|------|
| `audiosr/__main__.py` | CLI (`audiosr` コマンド) |
| `app.py` | Gradio Web UI (チャンク処理・ステレオ対応) |
| `predict.py` | Cog/Replicate 推論 (シンプル版) |
| `inference.py` | Cog推論 拡張版 (マルチバンドアンサンブル・ステレオ対応) |

### 重要な定数・制約

- 出力サンプルレート: 常に **48000 Hz**
- 潜在空間の時間解像度: **12.8 フレーム/秒** (`latent_t_per_second`)
- 入力は **5.12秒の倍数** にパディングされる
- 10.24秒を超える入力は品質劣化の可能性あり (警告表示)
- モデルチェックポイント: `basic` (汎用) / `speech` (音声特化) の2種。HuggingFace Hub からダウンロード
- メルスペクトログラム: 256 mel bins, 20-24000 Hz, hop_length=480, filter_length=2048
- デフォルト DDIM ステップ: 50, ガイダンススケール: 3.5

### 既知の制約

- 訓練データがローパスフィルタによるシミュレーションのみのため、**MP3圧縮のようなカットオフパターン**には弱い。入力前にローパスフィルタをかけると改善する
- `save_wave()` で `ffmpeg` を使って元の音声長にトリミングするため、**ffmpeg がシステムに必要**
