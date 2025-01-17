## LoRAの学習について

[LoRA: Low-Rank Adaptation of Large Language Models](https://arxiv.org/abs/2106.09685)（arxiv）、[LoRA](https://github.com/microsoft/LoRA)（github）をStable Diffusionに適用したものです。

[cloneofsimo氏のリポジトリ](https://github.com/cloneofsimo/lora)を大いに参考にさせていただきました。ありがとうございます。

8GB VRAMでもぎりぎり動作するようです。

## 学習したモデルに関する注意

cloneofsimo氏のリポジトリ、およびd8ahazard氏の[Dreambooth Extension for Stable-Diffusion-WebUI](https://github.com/d8ahazard/sd_dreambooth_extension)とは、現時点では互換性がありません。いくつかの機能拡張を行っているためです（後述）。

WebUI等で画像生成する場合には、学習したLoRAのモデルを学習元のStable Diffusionのモデルにこのリポジトリ内のスクリプトであらかじめマージしておくか、こちらの[WebUI用extention](https://github.com/kohya-ss/sd-webui-additional-networks)を使ってください。

## 学習方法

train_network.pyを用います。

DreamBoothの手法（identifier（sksなど）とclass、オプションで正則化画像を用いる）と、キャプションを用いるfine tuningの手法の両方で学習できます。

どちらの方法も既存のスクリプトとほぼ同じ方法で学習できます。異なる点については後述します。

### DreamBoothの手法を用いる場合

[DreamBoothのガイド](./train_db_README-ja.md) を参照してデータを用意してください。

学習するとき、train_db.pyの代わりにtrain_network.pyを指定してください。

ほぼすべてのオプション（Stable Diffusionのモデル保存関係を除く）が使えますが、stop_text_encoder_trainingはサポートしていません。

### キャプションを用いる場合

[fine-tuningのガイド](./fine_tune_README_ja.md) を参照し、各手順を実行してください。

学習するとき、fine_tune.pyの代わりにtrain_network.pyを指定してください。ほぼすべてのオプション（モデル保存関係を除く）がそのまま使えます。

なお「latentsの事前取得」は行わなくても動作します。VAEから学習時（またはキャッシュ時）にlatentを取得するため学習速度は遅くなりますが、代わりにcolor_augが使えるようになります。

### LoRAの学習のためのオプション

train_network.pyでは--network_moduleオプションに、学習対象のモジュール名を指定します。LoRAに対応するのはnetwork.loraとなりますので、それを指定してください。

なお学習率は通常のDreamBoothやfine tuningよりも高めの、1e-4程度を指定するとよいようです。

以下はコマンドラインの例です（DreamBooth手法）。

```
accelerate launch --num_cpu_threads_per_process 12 train_network.py 
    --pretrained_model_name_or_path=..\models\model.ckpt 
    --train_data_dir=..\data\db\char1 --output_dir=..\lora_train1 
    --reg_data_dir=..\data\db\reg1 --prior_loss_weight=1.0 
    --resolution=448,640 --train_batch_size=1 --learning_rate=1e-4 
    --max_train_steps=400 --use_8bit_adam --xformers --mixed_precision=fp16 
    --save_every_n_epochs=1 --save_model_as=safetensors --clip_skip=2 --seed=42 --color_aug 
    --network_module=networks.lora
```

--output_dirオプションで指定したディレクトリに、LoRAのモデルが保存されます。

その他、以下のオプションが指定できます。

* --network_dim
  * LoRAの次元数を指定します（``--networkdim=4``など）。省略時は4になります。数が多いほど表現力は増しますが、学習に必要なメモリ、時間は増えます。また闇雲に増やしても良くないようです。
* --network_weights
  * 学習前に学習済みのLoRAの重みを読み込み、そこから追加で学習します。
* --network_train_unet_only
  * U-Netに関連するLoRAモジュールのみ有効とします。fine tuning的な学習で指定するとよいかもしれません。
* --network_train_text_encoder_only
  * Text Encoderに関連するLoRAモジュールのみ有効とします。Textual Inversion的な効果が期待できるかもしれません。
* --unet_lr
  * U-Netに関連するLoRAモジュールに、通常の学習率（--learning_rateオプションで指定）とは異なる学習率を使う時に指定します。
* --text_encoder_lr
  * Text Encoderに関連するLoRAモジュールに、通常の学習率（--learning_rateオプションで指定）とは異なる学習率を使う時に指定します。Text Encoderのほうを若干低めの学習率（5e-5など）にしたほうが良い、という話もあるようです。

--network_train_unet_onlyと--network_train_text_encoder_onlyの両方とも未指定時（デフォルト）はText EncoderとU-Netの両方のLoRAモジュールを有効にします。

## マージスクリプトについて

merge_lora.pyでStable DiffusionのモデルにLoRAの学習結果をマージしたり、複数のLoRAモデルをマージしたりできます。

### Stable DiffusionのモデルにLoRAのモデルをマージする

マージ後のモデルは通常のStable Diffusionのckptと同様に扱えます。たとえば以下のようなコマンドラインになります。

```
python networks\merge_lora.py --sd_model ..\model\model.ckpt 
    --save_to ..\lora_train1\model-char1-merged.safetensors 
    --models ..\lora_train1\last.safetensors --ratios 0.8
```

Stable Diffusion v2.xのモデルで学習し、それにマージする場合は、--v2オプションを指定してください。

--sd_modelオプションにマージの元となるStable Diffusionのモデルファイルを指定します（.ckptまたは.safetensorsのみ対応で、Diffusersは今のところ対応していません）。

--save_toオプションにマージ後のモデルの保存先を指定します（.ckptまたは.safetensors、拡張子で自動判定）。

--modelsに学習したLoRAのモデルファイルを指定します。複数指定も可能で、その時は順にマージします。

--ratiosにそれぞれのモデルの適用率（どのくらい重みを元モデルに反映するか）を0~1.0の数値で指定します。例えば過学習に近いような場合は、適用率を下げるとマシになるかもしれません。モデルの数と同じだけ指定してください。

複数指定時は以下のようになります。

```
python networks\merge_lora.py --sd_model ..\model\model.ckpt 
    --save_to ..\lora_train1\model-char1-merged.safetensors 
    --models ..\lora_train1\last.safetensors ..\lora_train2\last.safetensors --ratios 0.8 0.5
```

### 複数のLoRAのモデルをマージする

複数のLoRAモデルをひとつずつSDモデルに適用する場合と、複数のLoRAモデルをマージしてからSDモデルにマージする場合とは、計算順序の関連で微妙に異なる結果になります。

たとえば以下のようなコマンドラインになります。

```
python networks\merge_lora.py 
    --save_to ..\lora_train1\model-char1-style1-merged.safetensors 
    --models ..\lora_train1\last.safetensors ..\lora_train2\last.safetensors --ratios 0.6 0.4
```

--sd_modelオプションは指定不要です。

--save_toオプションにマージ後のLoRAモデルの保存先を指定します（.ckptまたは.safetensors、拡張子で自動判定）。

--modelsに学習したLoRAのモデルファイルを指定します。三つ以上も指定可能です。

--ratiosにそれぞれのモデルの比率（どのくらい重みを元モデルに反映するか）を0~1.0の数値で指定します。二つのモデルを一対一でマージす場合は、「0.5 0.5」になります。「1.0 1.0」では合計の重みが大きくなりすぎて、恐らく結果はあまり望ましくないものになると思われます。

v1で学習したLoRAとv2で学習したLoRA、次元数の異なるLoRAはマージできません。U-NetだけのLoRAとU-Net+Text EncoderのLoRAはマージできるはずですが、結果は未知数です。


### その他のオプション

* precision
  * マージ計算時の精度をfloat、fp16、bf16から指定できます。省略時は精度を確保するためfloatになります。メモリ使用量を減らしたい場合はfp16/bf16を指定してください。
* save_precision
  * モデル保存時の精度をfloat、fp16、bf16から指定できます。省略時はprecisionと同じ精度になります。

## 当リポジトリ内の画像生成スクリプトで生成する

gen_img_diffusers.pyに、--network_module、--network_weightsの各オプションを追加してください。意味は学習時と同様です。

--network_mulオプションで0~1.0の数値を指定すると、LoRAの適用率を変えられます。

## 二つのモデルの差分からLoRAモデルを作成する

[こちらのディスカッション](https://github.com/cloneofsimo/lora/discussions/56)を参考に実装したものです。数式はそのまま使わせていただきました（よく理解していませんが近似には特異値分解を用いるようです）。

二つのモデル（たとえばfine tuningの元モデルとfine tuning後のモデル）の差分を、LoRAで近似します。

### スクリプトの実行方法

以下のように指定してください。
```
python networks\extract_lora_from_models.py --model_org base-model.ckpt
    --model_tuned fine-tuned-model.ckpt 
    --save_to lora-weights.safetensors --dim 4
```

--model_orgオプションに元のStable Diffusionモデルを指定します。作成したLoRAモデルを適用する場合は、このモデルを指定して適用することになります。.ckptまたは.safetensorsが指定できます。

--model_tunedオプションに差分を抽出する対象のStable Diffusionモデルを指定します。たとえばfine tuningやDreamBooth後のモデルを指定します。.ckptまたは.safetensorsが指定できます。

--save_toにLoRAモデルの保存先を指定します。--dimにLoRAの次元数を指定します。

生成されたLoRAモデルは、学習したLoRAモデルと同様に使用できます。

Text Encoderが二つのモデルで同じ場合にはLoRAはU-NetのみのLoRAとなります。

### その他のオプション

- --v2
  - v2.xのStable Diffusionモデルを使う場合に指定してください。
- --device
  - ``--device cuda``としてcudaを指定すると計算をGPU上で行います。処理が速くなります（CPUでもそこまで遅くないため、せいぜい倍～数倍程度のようです）。
- --save_precision
  - LoRAの保存形式を"float", "fp16", "bf16"から指定します。省略時はfloatになります。

## 追加情報

### cloneofsimo氏のリポジトリとの違い

12/25時点では、当リポジトリはLoRAの適用個所をText EncoderのMLP、U-NetのFFN、Transformerのin/out projectionに拡大し、表現力が増しています。ただその代わりメモリ使用量は増え、8GBぎりぎりになりました。

またモジュール入れ替え機構は全く異なります。

### 将来拡張について

LoRAだけでなく他の拡張にも対応可能ですので、それらも追加予定です。
