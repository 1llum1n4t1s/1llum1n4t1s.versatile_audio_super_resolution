
# AudioSR: 汎用オーディオ超解像

> **本リポジトリは [haoheliu/versatile_audio_super_resolution](https://github.com/haoheliu/versatile_audio_super_resolution) のフォークです。**
> PyTorch 2.6 / CUDA 12.4 対応、コード品質改善、AMD Radeon GPU (ROCm) 対応などを追加しています。

[![arXiv](https://img.shields.io/badge/arXiv-2309.07314-brightgreen.svg?style=flat-square)](https://arxiv.org/abs/2309.07314)  [![githubio](https://img.shields.io/badge/GitHub.io-Audio_Samples-blue?logo=Github&style=flat-square)](https://audioldm.github.io/audiosr) [![Replicate](https://replicate.com/nateraw/audio-super-resolution/badge)](https://replicate.com/nateraw/audio-super-resolution)

音声を入力すると、AudioSR が高忠実度に変換します。

あらゆる種類の音声（音楽、音声、環境音など）、あらゆるサンプリングレートに対応しています。

![概要図](https://github.com/haoheliu/versatile_audio_super_resolution/blob/main/visualization.png?raw=true)

## 変更履歴
- 2025-06-28: [LSD 計算の落とし穴デモ](example/lsd_calculation_pitfall/README.md)を追加。公平な Log Spectral Distance 評価におけるエネルギースケーリングの重要性を示します。
- 2024-12-31: AudioSR のトレーニングコードを[こちら](https://drive.google.com/file/d/1BaZuHbk1AfURX7SvkaD5_ZWLwun-wdpW/view?usp=drive_link)で公開（参考用。コードは整理されていません）。
- 2024-12-16: [AudioSR を正しく動作させるための重要事項](example/how_to_make_audiosr_work.md)を追加。
![デモ失敗例](example/figs/demo-failure.png)
- 2023-09-24: Replicate デモ追加（@nateraw）、Windows でのエラー修正、librosa 警告修正等（@ORI-Muchim）。
- 2023-09-16: DC シフト問題の修正。デュレーションパディングバグの修正。デフォルト DDIM ステップ数を 50 に更新。

## Gradio デモ

ローカルで Gradio デモを実行する手順:

1. 依存関係をインストール: `pip install -r requirements.txt`
2. アプリを起動: `python app.py`
3. 表示された URL をブラウザで開く

## インストール

```shell
# 推奨: conda 環境を作成
conda create -n audiosr python=3.9; conda activate audiosr

# NVIDIA GPU の場合
pip install -r requirements.txt

# AMD Radeon GPU (ROCm) の場合
pip install -r requirements-rocm.txt

# または pip から直接インストール
pip3 install audiosr==0.0.7
```

## コマンドライン使用方法

ファイルリストを一括処理（結果はデフォルトで `./output` に保存）:

```shell
audiosr -il batch.lst
```

単一ファイルを処理:
```shell
audiosr -i example/music.wav
```

全オプション:

```shell
> audiosr -h

> usage: audiosr [-h] -i INPUT_AUDIO_FILE [-il INPUT_FILE_LIST] [-s SAVE_PATH] [--model_name {basic,speech}] [-d DEVICE] [--ddim_steps DDIM_STEPS] [-gs GUIDANCE_SCALE] [--seed SEED]

オプション引数:
  -h, --help            ヘルプメッセージを表示
  -i INPUT_AUDIO_FILE, --input_audio_file INPUT_AUDIO_FILE
                        超解像を行う入力音声ファイル
  -il INPUT_FILE_LIST, --input_file_list INPUT_FILE_LIST
                        超解像を行う音声ファイルのリストファイル
  -s SAVE_PATH, --save_path SAVE_PATH
                        出力の保存先パス
  --model_name {basic,speech}
                        使用するチェックポイント
  -d DEVICE, --device DEVICE
                        計算デバイス。未指定の場合は環境に応じて自動選択
  --ddim_steps DDIM_STEPS
                        DDIM のサンプリングステップ数
  -gs GUIDANCE_SCALE, --guidance_scale GUIDANCE_SCALE
                        ガイダンススケール（大 => 高品質・テキスト忠実度向上、小 => 多様性向上）
  --seed SEED           生成結果を変更するシード値（任意の整数）
  --suffix SUFFIX       出力ファイルのサフィックス
```

## 引用
本リポジトリが有用であれば、以下の引用をご検討ください:
```bibtex
@inproceedings{liu2024audiosr,
  title={{AudioSR}: Versatile audio super-resolution at scale},
  author={Liu, Haohe and Chen, Ke and Tian, Qiao and Wang, Wenwu and Plumbley, Mark D},
  booktitle={IEEE International Conference on Acoustics, Speech and Signal Processing},
  pages={1076--1080},
  year={2024},
  organization={IEEE}
}
```

# カットオフパターンが AudioSR の性能に与える影響

**AudioSR** は強力なオーディオ超解像ツールですが、入力データの特性、特にカットオフパターンによって性能が大きく左右されます。

## AudioSR が失敗するケース
1. **不慣れなカットオフパターンを持つ入力音声**
   入力音声のカットオフパターンがトレーニング時のものと**大きく異なる**場合、AudioSR は効果的に機能しない可能性があります。

2. **深刻な歪みを持つ入力音声**
   過度なノイズやリバーブなどの強い歪みは、AudioSR の性能を低下させます。

## なぜカットオフパターンがこれほど影響するのか？
トレーニング時、データは**ローパスフィルタリング**によってシミュレートされました。MP3 圧縮など、高周波損失の他の原因に対してはトレーニングされていないため、不慣れなカットオフパターンに遭遇すると AudioSR は苦戦します。

例えば、MP3 圧縮は以下のようなカットオフパターンを生成します:

![MP3 カットオフ例](example/figs/mp3.png)

### 重要な理由
カットオフ範囲付近に**スペクトログラムの穴**があり、トレーニング時のパターンとは大きく異なります。このようなデータに AudioSR を適用すると、出力は以下のようになります:

![MP3 での AudioSR 出力](example/figs/mp3_after.png)

不慣れなカットオフパターンにより、高周波数が適切に復元されません。

### 解決策: ローパスフィルタリング
この問題を軽減するには、AudioSR に入力する前に音声に**ローパスフィルタリング**を適用します。ローパスフィルタリング後の音声は標準的なカットオフパターンに近づきます:

![ローパスフィルタ適用後](example/figs/lowpass.jpg)

AudioSR で処理すると、高周波の復元が改善された期待通りの出力が得られます:

![ローパス後の AudioSR 出力](example/figs/lowpass_after.png)

---

前処理で制限事項に対処することで、AudioSR の性能を最大限に引き出すことができます。
