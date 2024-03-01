---
layout: article
tags: AI
title: fastai notebook - text transfer learning
mathjax: true
key: Linux
---

[soruce](https://docs.fast.ai/tutorial.vision.html)
{:.info} 


## Text transfer learning
```
from fastai.text.all import *
```
### Train a text classifier from a pretrained model

#### Using the high-level API
```
path = untar_data(URLs.IMDB)
dls = TextDataLoaders.from_folder(untar_data(URLs.IMDB), valid='test')
dls.show_batch()

        text	category
0	xxbos xxmaj match 1 : xxmaj tag xxmaj team xxmaj table xxmaj match xxmaj bubba xxmaj ray and xxmaj spike xxmaj dudley vs xxmaj eddie xxmaj guerrero and xxmaj chris xxmaj benoit xxmaj bubba xxmaj ray and xxmaj spike xxmaj dudley started things off with a xxmaj tag xxmaj team xxmaj table xxmaj match against xxmaj eddie xxmaj guerrero and xxmaj chris xxmaj benoit . xxmaj according to the rules of the match , both opponents have to go through tables in order to get the win . xxmaj benoit and xxmaj guerrero heated up early on by taking turns hammering first xxmaj spike and then xxmaj bubba xxmaj ray . a xxmaj german xxunk by xxmaj benoit to xxmaj bubba took the wind out of the xxmaj dudley brother . xxmaj spike tried to help his brother , but the referee restrained him while xxmaj benoit and xxmaj guerrero	pos
1	xxbos xxmaj warning : xxmaj does contain spoilers . \n\n xxmaj open xxmaj your xxmaj eyes \n\n xxmaj if you have not seen this film and plan on doing so , just stop reading here and take my word for it . xxmaj you have to see this film . i have seen it four times so far and i still have n't made up my mind as to what exactly happened in the film . xxmaj that is all i am going to say because if you have not seen this film , then stop reading right now . \n\n xxmaj if you are still reading then i am going to pose some questions to you and maybe if anyone has any answers you can email me and let me know what you think . \n\n i remember my xxmaj grade 11 xxmaj english teacher quite well . xxmaj	pos

6	xxbos xxmaj i 've rented and watched this movie for the 1st time on xxup dvd without reading any reviews about it . xxmaj so , after 15 minutes of watching xxmaj i 've noticed that something is wrong with this movie ; it 's xxup terrible ! i mean , in the trailers it looked scary and serious ! \n\n i think that xxmaj eli xxmaj roth ( mr . xxmaj director ) thought that if all the characters in this film were stupid , the movie would be funny … ( so stupid , it 's funny … ? xxup wrong ! ) xxmaj he should watch and learn from better horror - comedies such xxunk xxmaj night " , " the xxmaj lost xxmaj boys " and " the xxmaj return xxmaj of the xxmaj living xxmaj dead " ! xxmaj those are funny ! \n\n "	neg

learn = text_classifier_learner(dls, AWD_LSTM, drop_mult=0.5, metrics=accuracy)

We use the AWD LSTM architecture, drop_mult is a parameter that controls the magnitude of all dropouts 
in that model, and we use accuracy to track down how well we are doing. We can then fine-tune our pretrained model:

learn.fine_tune(4, 1e-2)

learn.show_results()
learn.predict("I really liked that movie!")
> ('pos', tensor(1), tensor([0.0092, 0.9908]))
```

#### Using the data block API
```
imdb = DataBlock(blocks=(TextBlock.from_folder(path), CategoryBlock),
                 get_items=get_text_files,
                 get_y=parent_label,
                 splitter=GrandparentSplitter(valid_name='test'))
dls = imdb.dataloaders(path)
```

### The ULMFiT approach

#### Fine-tuning a language model on IMDb
```
dls_lm = TextDataLoaders.from_folder(path, is_lm=True, valid_pct=0.1)
learn = language_model_learner(dls_lm, AWD_LSTM, metrics=[accuracy, Perplexity()], path=path, wd=0.1).to_fp16()

# By default, a pretrained Learner is in a frozen state, meaning that only the head of the model will train while the body stays frozen. 
# We show you what is behind the fine_tune method here and use a fit_one_cycle method to fit the model:

learn.fit_one_cycle(1, 1e-2)

# You can easily save the state of your model like so:
learn.save('1epoch')

# It will create a file in learn.path/models/ named “1epoch.pth”. If you want to load your model on another machine 
# after creating your Learner the same way, or resume training later, you can load the content of this file with:
learn = learn.load('1epoch')

# We can them fine-tune the model after unfreezing:
learn.unfreeze()
learn.fit_one_cycle(10, 1e-3)

# Once this is done, we save all of our model except the final layer that converts activations to probabilities of picking each token in our vocabulary. 
# The model not including the final layer is called the encoder. We can save it with save_encoder:
learn.save_encoder('finetuned')

TEXT = "I liked this movie because"
N_WORDS = 40
N_SENTENCES = 2
preds = [learn.predict(TEXT, N_WORDS, temperature=0.75) 
         for _ in range(N_SENTENCES)]
print("\n".join(preds))
> i liked this movie because of its story and characters . The story line was very strong , very good for a sci - fi film . The main character , Alucard , was very well developed and brought the whole story
> i liked this movie because i like the idea of the premise of the movie , the ( very ) convenient virus ( which , when you have to kill a few people , the " evil " machine has to be used to protect
```

#### Training a text classifier
```
dls_clas = TextDataLoaders.from_folder(untar_data(URLs.IMDB), valid='test', text_vocab=dls_lm.vocab)

# The main difference is that we have to use the exact same vocabulary as when we were fine-tuning our language model, 
# or the weights learned won’t make any sense. We pass that vocabulary with text_vocab.

# Then we can define our text classifier like before:
learn = text_classifier_learner(dls_clas, AWD_LSTM, drop_mult=0.5, metrics=accuracy)

# The difference is that before training it, we load the previous encoder:
learn = learn.load_encoder('finetuned')

learn.fit_one_cycle(1, 2e-2)

# The last step is to train with discriminative learning rates and gradual unfreezing. 
# In computer vision, we often unfreeze the model all at once, but for NLP classifiers, we find that unfreezing a few layers at a time makes a real difference.

learn.fit_one_cycle(1, 2e-2)

learn.freeze_to(-2)
learn.fit_one_cycle(1, slice(1e-2/(2.6**4),1e-2))

learn.freeze_to(-3)
learn.fit_one_cycle(1, slice(5e-3/(2.6**4),5e-3))

learn.unfreeze()
learn.fit_one_cycle(2, slice(1e-3/(2.6**4),1e-3))
```
