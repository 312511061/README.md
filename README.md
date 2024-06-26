DL_HW1

下面是使用hugging face的code
-
import pandas as pd
from datasets import Dataset, load_metric
import torchaudio
from transformers import AutoTokenizer, WhisperProcessor, WhisperForConditionalGeneration, TrainingArguments, Trainer
import numpy as np
from sklearn.model_selection import train_test_split
import torch

--
read dataset
--
df = pd.read_csv('nycu-iass-dl2024-taiwanese-asr/train/train-toneless.csv')
from datasets import Dataset, Audio
dataset = Dataset.from_pandas(df)
from datasets import DatasetDict

--
spilt
--
df_train, df_test = train_test_split(df, test_size=0.2)
dataset_train = Dataset.from_pandas(df_train)
dataset_test = Dataset.from_pandas(df_test)
dataset = DatasetDict({
    'train': dataset_train,
    'test': dataset_test
})

dataset = dataset.map(lambda x: {'audio': f"./nycu-iass-dl2024-taiwanese-asr/train/train/{x['id']}.wav"}, remove_columns=['id'])
from transformers import WhisperFeatureExtractor
from transformers import WhisperProcessor
from transformers import WhisperTokenizer

feature_extractor = WhisperFeatureExtractor.from_pretrained("openai/whisper-small")
tokenizer = WhisperTokenizer.from_pretrained("openai/whisper-small", language='en', task="transcribe")
processor = WhisperProcessor.from_pretrained("openai/whisper-small",language='en', task="transcribe")

--
sampling to 16KHz
--
dataset = dataset.cast_column("audio", Audio(sampling_rate=16000))

def prepare_dataset(batch):
    audio = batch["audio"]

    batch["input_features"] = feature_extractor(audio["array"], sampling_rate=audio["sampling_rate"]).input_features[0]
    
    batch["labels"] = tokenizer(batch["text"]).input_ids

    return batch

dataset = dataset.map(prepare_dataset,remove_columns=dataset.column_names["train"])

import torch
from dataclasses import dataclass
from typing import Any, Dict, List, Union
@dataclass
class DataCollatorSpeechSeq2SeqWithPadding:
    processor: Any

    def __call__(self, features: List[Dict[str, Union[List[int], torch.Tensor]]]) -> Dict[str, torch.Tensor]:
        # split inputs and labels since they have to be of different lengths and need different padding methods
        # first treat the audio inputs by simply returning torch tensors
        input_features = [{"input_features": feature["input_features"]} for feature in features]
        batch = self.processor.feature_extractor.pad(input_features, return_tensors="pt")

        # get the tokenized label sequences
        label_features = [{"input_ids": feature["labels"]} for feature in features]
        # pad the labels to max length
        labels_batch = self.processor.tokenizer.pad(label_features, return_tensors="pt")

        # replace padding with -100 to ignore loss correctly
        labels = labels_batch["input_ids"].masked_fill(labels_batch.attention_mask.ne(1), -100)

        # if bos token is appended in previous tokenization step,
        # cut bos token here as it's append later anyways
        if (labels[:, 0] == self.processor.tokenizer.bos_token_id).all().cpu().item():
            labels = labels[:, 1:]

        batch["labels"] = labels

        return batch
    
data_collator = DataCollatorSpeechSeq2SeqWithPadding(processor=processor)

import evaluate

metric = evaluate.load("wer")

def compute_metrics(pred):
    pred_ids = pred.predictions
    label_ids = pred.label_ids

    # replace -100 with the pad_token_id
    label_ids[label_ids == -100] = tokenizer.pad_token_id

    # we do not want to group tokens when computing the metrics
    pred_str = tokenizer.batch_decode(pred_ids, skip_special_tokens=True)
    label_str = tokenizer.batch_decode(label_ids, skip_special_tokens=True)

    wer = 100 * metric.compute(predictions=pred_str, references=label_str)

    return {"wer": wer}

    from transformers import Seq2SeqTrainingArguments
    
from transformers import WhisperForConditionalGeneration

# model setting
model = WhisperForConditionalGeneration.from_pretrained("openai/whisper-small")

model.config.forced_decoder_ids = None
model.config.suppress_tokens = []

#  per_device_eval_batch_size=1 否則會出錯(whisper更新強制檢測預輸入語言 台語輸出容易偵測到多種語言導致錯誤)
training_args = Seq2SeqTrainingArguments(
    #add
    output_dir="./whisper-small-EN",  
    per_device_train_batch_size=16,
    gradient_accumulation_steps=1,  
    learning_rate=1e-5,
    warmup_steps=500,
    max_steps=5000,
    gradient_checkpointing=True,
    fp16=True,
    evaluation_strategy="steps",
    per_device_eval_batch_size=1,#otherwise, can't run
    predict_with_generate=True,
    generation_max_length=225,
    save_steps=1000,
    eval_steps=1000,
    logging_steps=25,
    report_to=["tensorboard"],
    load_best_model_at_end=True,
    metric_for_best_model="wer",
    greater_is_better=False,
    push_to_hub=False,
)

from transformers import Seq2SeqTrainer

trainer = Seq2SeqTrainer(
    args=training_args,
    model=model,
    train_dataset=dataset["train"],  
    eval_dataset=dataset["test"],  
    data_collator=data_collator,
    compute_metrics=compute_metrics,
    tokenizer=processor.feature_extractor,
)
processor.save_pretrained(training_args.output_dir)
# train
trainer.train(resume_from_checkpoint=True)
#trainer.train()


-
使用NVIDIA GTX1080TI訓練
-
原本是打算都使用WSL2在本機端進行訓練 但是會出現預期外的CUDA錯誤(只有ESPnet會出錯) 
-
最後task1 2只好改在colab進行嘗試 但是算力吃緊下僅初步觀察結果(task1 WER約60% task2約50%)
-
由於task3可以使用hugging face 在我的anaconda底下執行 
-
因此主要是使用task3 whisper-small 進行比較多的測試
-
後面主要針對whisper修改 
-
卡比較久的地方應該是whisper近期更新 會預設選擇一種語言 因此閩南語可能會使得模型偵測到多種語言產生錯誤
-
解決辦法是把per_device_eval_batch_size=1
-
選擇whisper的語言時首先嘗試使用中文(認為中文可能跟閩南語的適配度高)
-
然後選擇英文與西班牙文(在whisper doc中 西文對大多數語言的匹配率效果甚好)
-
最後發現英文的結果可能再好一點
-
每個語言進行初次訓練(設置check point) 以英文的效果最好 接下去訓練到max_step = 5000
-

