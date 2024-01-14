## はじめに

オープンソースのLLM(大規模言語モデル)を個人で利用する場合、GPUはとても高価で手が出せない場合が多いです。CPUを利用すれば実行時間がGPU利用時に比べてだいぶ長くなりますが、安価に利用できるようになります。

ここでは、東京工業大学と産総研が開発したLLMであるSwallowの利用方法と、ファインチューニング方法として効果的なLoRA（ローラ、ロラ、Low-Rank Adaptation）を用いたファインチューニングについてメモします。

## この記事にピッタリの方

1. OSSのLLMをプライベートで動作させたい
2. LoRAでファインチューニングしたい
3. GPUがないため、CPUを利用したい

## この記事で前提とすること

1. LLMについて基本的な知識とpythonやLinuxの基本知識がある

## おしながき

1. 試行に使った環境(マシンスペック) 
2. Swallowを動作させる(推論)
3. SwallowをLoRAでファインチューニングする
4. ファインチューニングした結果を利用する
5. まとめ

## 試行に使った環境(マシンスペック) 

OS：Ubuntu 22.04.3 LTS 
CPU数:16個(AMD Ryzen 5)
メモリ:約62GB
HDD：400GB

メモリは60GB以上は必須です。
HDDは200GB程度でも動作するかもですが、モデルのファイルサイズが大きくなりがちなため、沢山用意しておいたほうが良いです。

なお、GPUはありません。

## Swallowを動作させる(推論動作)

Swallowとは東京工業大学と産総研がタッグを組んで開発した、現在最大規模の日本語に対応した大規模言語モデルで、オープンで商用利用が可能であるため、ビジネスに安心して用いることができるものです。詳細は、[東京工業大学のページ](https://www.titech.ac.jp/news/2023/068089)に譲ります。

ここではそれを手元のパソコンで動作させることを試みます。

### 動作に必要なライブラリのインストール

この環境では以下の作業が必要でした。

```shell-session
sudo apt install pip
pip install transformers sentencepiece accelerate
pip install protobuf
```

### 動作のサンプルプログラムのダウンロード

```shell-session
git clone https://github.com/miyakz1192/alpaca-lora.git
cd alpaca-lora
git checkout oncpu
```

では早速、swallowをダウンロードして実際に動かしてみます。試しに、以下のようなプロンプトを入力してみます。

> 以下に、あるタスクを説明する指示があり、それに付随する入力が更なる文脈を提供しています。リクエストを適切に完了するための回答を記述してください。
> 
> ### 指示:
> 以下のトピックに関する詳細な情報を提供してください
> 
> 
> ### 入力:
> この中でラッパーなのはどれ？エミネム、マイケル・ジャクソン、リアーナ、50セント

以下のコマンドを実行します。このコマンドを実行すれば、自動的にswallowのモデルがネット(hugging face)からダウンロードされます。

```shell-session
cd sampleuse
./sample_swallow.py  --verbose  --input=input.txt --instruction=instruction.txt
```

以下のような結果となります(結果は毎回変わる可能性があります)。

```shell-session
miyakz@swallow:~/github_repos/alpaca-lora/sampleuse$ ./sample_swallow.py  --verbose  --input=input.txt --instruction=instruction.txt
LoRAのデータ指定=False
LoRAのデータディレクトリ=None
指示プロンプトファイル=instruction.txt
プロンプトファイル=input.txt
モデルの個別指定=None
最終的に選択したモデル=tokyotech-llm/Swallow-7b-instruct-hf
ベースモデルのロード
Loading checkpoint shards: 100%|██████████████████| 3/3 [00:02<00:00,  1.11it/s]
以下に、あるタスクを説明する指示があり、それに付随する入力が更なる文脈を提供しています。リクエストを適切に完了するための回答を記述してください。

### 指示:
以下のトピックに関する詳細な情報を提供してください


### 入力:
この中でラッパーなのはどれ？エミネム、マイケル・ジャクソン、リアーナ、50セント


### 応答:50セント
miyakz@swallow:~/github_repos/alpaca-lora/sampleuse$
```

なお、今回使ったモデルは、約70億パラメータ(最小)の"tokyotech-llm/Swallow-7b-instruct-hf"になります。必要に応じて選択可能です。選択可能なモデルは、[Swallowのhugging faceのサイト](https://huggingface.co/tokyotech-llm)を参照ください。

このサンプルコマンドの使用方法は以下の通りです。

```shell-session
miyakz@swallow:~/github_repos/alpaca-lora/sampleuse$ ./sample_swallow.py  --help
usage: sample_swallow.py [-h] [--lora_data LORA_DATA]
                         [--instruction INSTRUCTION] [--input INPUT]
                         [--model MODEL] [--verbose]
                         [--max_new_tokens MAX_NEW_TOKENS]

Swallow利用のサンプル

options:
  -h, --help            show this help message and exit
  --lora_data LORA_DATA
                        LoRAデータのディレクトリ名(絶対パスで指定する)
  --instruction INSTRUCTION
                        指示プロンプトを記載したファイル
  --input INPUT         入力プロンプトを記載したファイル
  --model MODEL         モデル名を指定します
  --verbose             詳細出力を有効
  --max_new_tokens MAX_NEW_TOKENS
                        出力のトークン数(デフォルト128。min:128, max:1024, step:64)
miyakz@swallow:~/github_repos/alpaca-lora/sampleuse$ 
```

モデル名を指定しないと、tokyotech-llm/Swallow-7b-instruct-hfが選択されます。
"--lora_data"は、ファインチューニングした結果生成されたファイルのディレクトリを指定します。LoRAによるファインチューニングは後述します。


## Swallowをファインチューニングする

それでは、Swallowをファインチューニングしてみます。
LoRAとはモデル全体を訓練するのではなく、重みの一部分を外付けで訓練して、モデル本体に移入するようなイメージです。詳細は[日経クロステック](https://xtech.nikkei.com/atcl/nxt/keyword/18/00002/120800245/)のサイトがわかりやすいです。

また、訓練データが必要です。データはこちらからダウンロードします。
databricks-dolly-15k-jaはDatabricks社の社員たちに作られたデータだそうで、オープンソースのものになります。

```shell-session
git clone https://github.com/kunishou/databricks-dolly-15k-ja.git
```

databricks-dolly-15k-jaディレクトリに移動し、databricks-dolly-15k-ja.jsonというファイルを先ほどのalpaca-loraディレクトリ下に保存しておきます。例えば、dataset/というディレクトリを掘り、そこに入れておきます。

次に、alpaca-loraディレクトリに行き、以下のスクリプトを実行します。

```shell-session
cd alpaca-lora
./go_finetune2.sh
```

実際の中身は以下を実行しているだけです。

```bash
set -x
python3 finetune.py --base_model="tokyotech-llm/Swallow-7b-instruct-hf"  --data_path="./dataset/databricks-dolly-15k-ja.json" --num_epochs=1 --output_dir=output
```

CPUのみの環境だとカナリの時間を要します。都合上、上記の通り、epoch=1で訓練しました。こちらの環境だと、約5日間かかりました。

## ファインチューニングした結果を利用する

以下のようにして、outputに吐き出されたデータを指定します。

```shell-session
miyakz@swallow:~/github_repos/alpaca-lora/sampleuse$ ./sample_swallow.py  --input=input.txt --instruction=instruction.txt --lora_data=/home/miyakz/alpaca-lora/output
Loading checkpoint shards: 100%|██████████████████| 3/3 [00:02<00:00,  1.11it/s]
/home/miyakz/.local/lib/python3.10/site-packages/bitsandbytes/cextension.py:34: UserWarning: The installed version of bitsandbytes was compiled without GPU support. 8-bit optimizers, 8-bit multiplication, and GPU quantization are unavailable.
  warn("The installed version of bitsandbytes was compiled without GPU support. "
/home/miyakz/.local/lib/python3.10/site-packages/bitsandbytes/libbitsandbytes_cpu.so: undefined symbol: cadam32bit_grad_fp32
以下に、あるタスクを説明する指示があり、それに付随する入力が更なる文脈を提供しています。リクエストを適切に完了するための回答を記述してください。

### 指示:
以下のトピックに関する詳細な情報を提供してください


### 入力:
この中でラッパーなのはどれ？エミネム、マイケル・ジャクソン、リアーナ、50セント


### 応答:マイケル・ジャクソン、50セント
miyakz@swallow:~/github_repos/alpaca-lora/sampleuse$ 
```

この入力質問の正しい答えは、"エミネム、50セント"ですので、間違っています(マイケル・ジャクソンが出力されてしまっている)

今度は、LoRAデータを指定せずに実行してみます。

```shell-session
miyakz@swallow:~/github_repos/alpaca-lora/sampleuse$ ./sample_swallow.py  --input=input.txt --instruction=instruction.txt
Loading checkpoint shards: 100%|██████████████████| 3/3 [00:02<00:00,  1.11it/s]
以下に、あるタスクを説明する指示があり、それに付随する入力が更なる文脈を提供しています。リクエストを適切に完了するための回答を記述してください。

### 指示:
以下のトピックに関する詳細な情報を提供してください


### 入力:
この中でラッパーなのはどれ？エミネム、マイケル・ジャクソン、リアーナ、50セント


### 応答:ラップはエミネムと50セントのもので、ヒップホップはマイケルジャクソンとリアーナのものです。
miyakz@swallow:~/github_repos/alpaca-lora/sampleuse$
```

正しいです。

結果としては、LoRAで訓練したデータを指定したほうが、誤回答が混じってしまう(悪化してしまう)という結果が得られました。

理由としては、今回の訓練のepochが1回のみだったことが考えられます。このことが、精度が良くなかった結果につながったのだと考えています。次回以降、epoch回数を増やしてみたいと思います。

ただ、LoRAデータの利用により結果が変化したため、LoRAによる訓練がベースモデルに反映された事はわかりました。

### まとめ

オープンソースのLLMであるSwallowの推論動作のサンプルと、LoRAによるファインチューニングおよびその結果の利用方法について述べました。

なお、この記事はオープンソースのLLMの利用方法のサンプルを示すものであり、いかなる結果について何かの責任を持ち、保証する訳ではないので、ご注意ください。ダウンロード先の知的財産、ライセンス等はご利用前にご確認をおねがいします。

