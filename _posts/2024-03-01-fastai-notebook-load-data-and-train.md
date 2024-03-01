---
layout: article
tags: AI
title: fastai notebook - loading data and training
mathjax: true
key: Linux
---

[soruce](https://docs.fast.ai/tutorial.vision.html)
{:.info} 


## Loading the data with a factory method
```
from fastai.vision.all import *
path = untar_data(URLs.IMAGENETTE_160)
dls = ImageDataLoaders.from_folder(path, valid='val', 
    item_tfms=RandomResizedCrop(128, min_scale=0.35), batch_tfms=Normalize.from_stats(*imagenet_stats))
dls.show_batch()

```

## Loading the data with the data block API
```
fnames = get_image_files(path)

lbl_dict = dict(
    n01440764='tench',
    n02102040='English springer',
    n02979186='cassette player',
    n03000684='chain saw',
    n03028079='church',
    n03394916='French horn',
    n03417042='garbage truck',
    n03425413='gas pump',
    n03445777='golf ball',
    n03888257='parachute'
)

def label_func(fname):
    return lbl_dict[parent_label(fname)]
dblock = DataBlock(blocks    = (ImageBlock, CategoryBlock),
                   get_items = get_image_files,
                   get_y     = label_func,
                   splitter  = GrandparentSplitter(),
                   item_tfms = RandomResizedCrop(128, min_scale=0.35), 
                   batch_tfms=Normalize.from_stats(*imagenet_stats))
dls = dblock.dataloaders(path)
dls.show_batch()

# Another way to compose several functions for get_y is to put them in a Pipeline:
imagenette = DataBlock(blocks = (ImageBlock, CategoryBlock),
                       get_items = get_image_files,
                       get_y = Pipeline([parent_label, lbl_dict.__getitem__]),
                       splitter = GrandparentSplitter(valid_name='val'),
                       item_tfms = RandomResizedCrop(128, min_scale=0.35),
                       batch_tfms = Normalize.from_stats(*imagenet_stats))
```

## Loading the data with the mid-level API
```
source = untar_data(URLs.IMAGENETTE_160)
fnames = get_image_files(source)

# To open an image, we use the PILImage.create transform. It will open the image and make it of the fastai type PILImage:
PILImage.create(fnames[0])

tfm = Pipeline([parent_label, lbl_dict.__getitem__, Categorize(vocab = lbl_dict.values())])
tfm(fnames[0])
splits = GrandparentSplitter(valid_name='val')(fnames)
dsets = Datasets(fnames, [[PILImage.create], [parent_label, lbl_dict.__getitem__, Categorize]], splits=splits)

item_tfms = [ToTensor, RandomResizedCrop(128, min_scale=0.35)]
batch_tfms = [IntToFloatTensor, Normalize.from_stats(*imagenet_stats)]
dls = dsets.dataloaders(after_item=item_tfms, after_batch=batch_tfms, bs=64, num_workers=8)
dls.show_batch()

```

## Building a Learner
```
learn = vision_learner(dls, resnet34, metrics=accuracy, pretrained=False)
learn.fit_one_cycle(5, 5e-3)

learn = Learner(dls, xresnet34(n_out=10), metrics=accuracy)
learn.lr_find()
learn.fit_one_cycle(5, 1e-3)
learn.show_results()

```

