## Install

Checkout the project and install dependencies:
```
git clone --recursive https://github.com/tugstugi/mongolian-bert.git
pip3 install -r requirements.txt
```


## Data preparation

Download the Mongolian Wikipedia and the 700 million word Mongolian news data set and pre process them into a directory `mn_corpus/`:
(you can also download it from [here](https://www.dropbox.com/s/l4wldeuyzi0x26k/mn_corpus.tar.gz?dl=1))
```
# Mongolian Wikipedia
python3 dl_and_preprop_mn_wiki.py
# 700 million words Mongolian news data set
python3 dl_and_preprop_mn_news.py
```

## Train SentencePiece vocabulary

Now, train the SentencePiece model i.e. with the vocabulary size 32000 :
```
cat mn_corpus/*.txt > all.txt
python3 train_sentencepiece.py --input all.txt --vocab-size 32000 --prefix mn_cased
```
If the training was successful, the following files should be created: `mn_cased.model` and `mn_cased.vocab`.
You can also test whether the SentencePiece model is working as intended:
```
>>> import sentencepiece as spm
>>> s = spm.SentencePieceProcessor()
>>> s.Load('mn_cased.model')
>>> s.EncodeAsPieces('Мөнгөө тушаачихсаныхаа дараа мэдэгдээрэй')
['▁Мөнгөө', '▁тушаа', 'чихсан', 'ыхаа', '▁дараа', '▁мэдэгд', 'ээрэй']
```
Move `mn_cased.model` and `mn_cased.vocab` into the folder `model-32k`


## Active Training Models

* [model-32k](model-32k) for 1.000.000 steps with batch size 256
  * checkpoints and tensorboard logs [here](https://console.cloud.google.com/storage/browser/mongolian-bert/model-32k)
 
Planned:
* model-32k-512 for training from scratch with max_seq_length=512 with batch size 48

## Training

Some interesting info from the BERT documentation:
* `max_predictions_per_seq=max_seq_length*masked_lm_prob`: it means for each max seq length, TFRecords must be created again.
* pretrain 90,000 steps with a sequence length of 128 and then for 10,000 additional steps with a sequence length of 512.


### Troubleshooting
* If the training script get a permission denies, execute on the VM `gcloud auth application-default login`
* TPU service account added into the storage permission?
* TPU name same as the model name?


### Training for vocab_size=32000
* model directory is [model-32k](model-32k)
* bucket name: `gs://mongolian-bert/model-32k` and [bucket URL](https://console.cloud.google.com/storage/browser/mongolian-bert/model-32k)

Generate TFRecords for `max_seq_length=128` and copy them into bucket:
```
python3 create_pretraining_data_for_model.py --max_seq_length=128 --max_predictions_per_seq=20 model-32k
gsutil cp maxseq128*.tfrecord gs://mongolian-bert/model-32k/
```

Train for `max_seq_length=128`:
```
export MODEL_DIR=model-32k
export TPU_ADDRESS=$MODEL_DIR
export INPUT_FILES=gs://mongolian-bert/$MODEL_DIR/maxseq128-mn_news_700m_1.tfrecord,gs://mongolian-bert/$MODEL_DIR/maxseq128-mn_news_700m_10.tfrecord,gs://mongolian-bert/$MODEL_DIR/maxseq128-mn_news_700m_11.tfrecord,gs://mongolian-bert/$MODEL_DIR/maxseq128-mn_news_700m_12.tfrecord,gs://mongolian-bert/$MODEL_DIR/maxseq128-mn_news_700m_13.tfrecord,gs://mongolian-bert/$MODEL_DIR/maxseq128-mn_news_700m_14.tfrecord,gs://mongolian-bert/$MODEL_DIR/maxseq128-mn_news_700m_15.tfrecord,gs://mongolian-bert/$MODEL_DIR/maxseq128-mn_news_700m_16.tfrecord,gs://mongolian-bert/$MODEL_DIR/maxseq128-mn_news_700m_17.tfrecord,gs://mongolian-bert/$MODEL_DIR/maxseq128-mn_news_700m_18.tfrecord,gs://mongolian-bert/$MODEL_DIR/maxseq128-mn_news_700m_19.tfrecord,gs://mongolian-bert/$MODEL_DIR/maxseq128-mn_news_700m_2.tfrecord,gs://mongolian-bert/$MODEL_DIR/maxseq128-mn_news_700m_3.tfrecord,gs://mongolian-bert/$MODEL_DIR/maxseq128-mn_news_700m_4.tfrecord,gs://mongolian-bert/$MODEL_DIR/maxseq128-mn_news_700m_5.tfrecord,gs://mongolian-bert/$MODEL_DIR/maxseq128-mn_news_700m_6.tfrecord,gs://mongolian-bert/$MODEL_DIR/maxseq128-mn_news_700m_7.tfrecord,gs://mongolian-bert/$MODEL_DIR/maxseq128-mn_news_700m_8.tfrecord,gs://mongolian-bert/$MODEL_DIR/maxseq128-mn_news_700m_9.tfrecord,gs://mongolian-bert/$MODEL_DIR/maxseq128-mn_wiki.tfrecord
python3 bert/run_pretraining.py \
  --input_file=$INPUT_FILES \
  --output_dir=gs://mongolian-bert/$MODEL_DIR/model \
  --use_tpu=True \
  --tpu_name=$TPU_ADDRESS \
  --num_tpu_cores=8 \
  --do_train=True \
  --do_eval=True \
  --bert_config_file=$MODEL_DIR/bert_config.json \
  --train_batch_size=256 \
  --max_seq_length=128 \
  --max_predictions_per_seq=20 \
  --num_train_steps=1000000 \
  --num_warmup_steps=10000 \
  --learning_rate=1e-4
```

Generate TFRecords for `max_seq_length=512` and copy them into bucket (see `max_predictions_per_seq`):
```
python3 create_pretraining_data_for_model.py --max_seq_length=512 --max_predictions_per_seq=77 model-32k
gsutil cp maxseq128*.tfrecord gs://mongolian-bert/model-32k/
```
batch size 48?


## TODO:
* SentencePiece unigram or BPE?
* vocabulary size bigger than 32000? Mongolian language has 85K root words.
  * english cased: 28996
  * multi lingual cased: 119547 `[PAD][UNK][CLS][SEP][MASK]<S><T>`
  * japanese uncased: 32000 `[PAD][CLS][SEP][MASK]`
* ...
