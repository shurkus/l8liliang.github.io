---
layout: article
tags: AI
title: fastai notebook - transformer
mathjax: true
key: Linux
---

[soruce](https://docs.fast.ai/tutorial.vision.html)
{:.info} 


## Text transfer learning
```
!pip install -Uq transformers
from transformers import GPT2LMHeadModel, GPT2TokenizerFast

from fastai.text.all import *

pretrained_weights = 'gpt2'
tokenizer = GPT2TokenizerFast.from_pretrained(pretrained_weights)
model = GPT2LMHeadModel.from_pretrained(pretrained_weights)

ids = tokenizer.encode('This is an example of text, and')
ids
> [1212, 318, 281, 1672, 286, 2420, 11, 290]


tokenizer.decode(ids)
> 'This is an example of text, and'

import torch
t = torch.LongTensor(ids)[None]
preds = model.generate(t)

preds.shape,preds[0]

tokenizer.decode(preds[0].numpy())
> "This is an example of text, and it's not a good one.\n\nThe first thing"

# Bridging the gap with fastai
path = untar_data(URLs.WIKITEXT_TINY)
path.ls()
df_train = pd.read_csv(path/'train.csv', header=None)
df_valid = pd.read_csv(path/'test.csv', header=None)
df_train.head()

all_texts = np.concatenate([df_train[0].values, df_valid[0].values])

class TransformersTokenizer(Transform):
    def __init__(self, tokenizer): self.tokenizer = tokenizer
    def encodes(self, x): 
        toks = self.tokenizer.tokenize(x)
        return tensor(self.tokenizer.convert_tokens_to_ids(toks))
    def decodes(self, x): return TitledStr(self.tokenizer.decode(x.cpu().numpy()))

splits = [range_of(df_train), list(range(len(df_train), len(all_texts)))]
tls = TfmdLists(all_texts, TransformersTokenizer(tokenizer), splits=splits, dl_type=LMDataLoader)

tls.train[0],tls.valid[0]

tls.tfms(tls.train.items[0]).shape, tls.tfms(tls.valid.items[0]).shape
show_at(tls.train, 0)
show_at(tls.valid, 0)

bs,sl = 4,256
dls = tls.dataloaders(bs=bs, seq_len=sl)
dls.show_batch(max_n=2)

class DropOutput(Callback):
    def after_pred(self): self.learn.pred = self.pred[0]

learn = Learner(dls, model, loss_func=CrossEntropyLossFlat(), cbs=[DropOutput], metrics=Perplexity()).to_fp16()
learn.validate()
learn.lr_find()
learn.fit_one_cycle(1, 1e-4)
df_valid.head(1)
prompt = "\n = Unicorn = \n \n A unicorn is a magical creature with a rainbow tail and a horn"
prompt_ids = tokenizer.encode(prompt)
inp = tensor(prompt_ids)[None].cuda()
inp.shape
preds = learn.model.generate(inp, max_length=40, num_beams=5, temperature=1.5)
tokenizer.decode(preds[0].cpu().numpy())
```
