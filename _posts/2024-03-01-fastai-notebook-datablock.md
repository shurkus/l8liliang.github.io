---
layout: article
tags: AI
title: fastai notebook - datablock
mathjax: true
key: Linux
---

[soruce](https://docs.fast.ai/tutorial.vision.html)
{:.info} 


## Building a DataBlock from scratch
```
from fastai.data.all import *
from fastai.vision.all import *

path = untar_data(URLs.PETS)

fnames = get_image_files(path/"images")

dblock = DataBlock()

dsets = dblock.datasets(fnames)
dsets.train[0]
> 
(Path('/home/jhoward/.fastai/data/oxford-iiit-pet/images/Birman_82.jpg'),
 Path('/home/jhoward/.fastai/data/oxford-iiit-pet/images/Birman_82.jpg'))
# By default, the data block API assumes we have an input and a target, which is why we see our filename repeated twice.

# The first thing we can do is use a get_items function to actually assemble our items inside the data block:
dblock = DataBlock(get_items = get_image_files)
dsets = dblock.datasets(path/"images")
dsets.train[0]

# specify label func
def label_func(fname):
    return "cat" if fname.name[0].isupper() else "dog"

dblock = DataBlock(get_items = get_image_files,
                   get_y     = label_func)
dsets = dblock.datasets(path/"images")
dsets.train[0]

#  specify types 
dblock = DataBlock(blocks    = (ImageBlock, CategoryBlock),
                   get_items = get_image_files,
                   get_y     = label_func)

dsets = dblock.datasets(path/"images")
dsets.train[0]

dsets.vocab
> ['cat', 'dog']

dblock = DataBlock(blocks    = (ImageBlock, CategoryBlock),
                   get_items = get_image_files,
                   get_y     = label_func,
                   splitter  = RandomSplitter(),
                   item_tfms = Resize(224))
dls = dblock.dataloaders(path/"images")
dls.show_batch()
```

### Image classification - MNIST (single label)
```
mnist = DataBlock(blocks=(ImageBlock(cls=PILImageBW), CategoryBlock), 
                  get_items=get_image_files, 
                  splitter=GrandparentSplitter(),
                  get_y=parent_label)

dls = mnist.dataloaders(untar_data(URLs.MNIST_TINY))
dls.show_batch(max_n=9, figsize=(4,4))
mnist.summary(untar_data(URLs.MNIST_TINY))
```

### Image classification - Pets (single label)
```
# resize and augmentation
pets = DataBlock(blocks=(ImageBlock, CategoryBlock), 
                 get_items=get_image_files, 
                 splitter=RandomSplitter(),
                 get_y=Pipeline([attrgetter("name"), RegexLabeller(pat = r'^(.*)_\d+.jpg$')]),
                 item_tfms=Resize(128),
                 batch_tfms=aug_transforms())
dls = pets.dataloaders(untar_data(URLs.PETS)/"images")
dls.show_batch(max_n=9)
```

### Image classification - Pascal (multi-label)
```
pascal_source = untar_data(URLs.PASCAL_2007)
df = pd.read_csv(pascal_source/"train.csv")

        fname	labels	is_valid
0	000005.jpg	chair	True
1	000007.jpg	car	True
2	000009.jpg	horse person	True
3	000012.jpg	car	False
4	000016.jpg	bicycle	True

# get_x=ColReader(0, pref=pascal_source/"train"),
# 意思是读取第0列的值，并且添加前缀生成文件名
pascal = DataBlock(blocks=(ImageBlock, MultiCategoryBlock),
                   splitter=ColSplitter(),
                   get_x=ColReader(0, pref=pascal_source/"train"),
                   get_y=ColReader(1, label_delim=' '),
                   item_tfms=Resize(224),
                   batch_tfms=aug_transforms())
dls = pascal.dataloaders(df)
dls.show_batch()

# 
pascal = DataBlock(blocks=(ImageBlock, MultiCategoryBlock),
                   splitter=ColSplitter(),
                   get_x=lambda x:pascal_source/"train"/f'{x[0]}',
                   get_y=lambda x:x[1].split(' '),
                   item_tfms=Resize(224),
                   batch_tfms=aug_transforms())

dls = pascal.dataloaders(df)
dls.show_batch()

# use lambda
pascal = DataBlock(blocks=(ImageBlock, MultiCategoryBlock),
                   splitter=ColSplitter(),
                   get_x=lambda o:f'{pascal_source}/train/'+o.fname,
                   get_y=lambda o:o.labels.split(),
                   item_tfms=Resize(224),
                   batch_tfms=aug_transforms())

dls = pascal.dataloaders(df)
dls.show_batch()

# use columns as attributes
pascal = DataBlock(blocks=(ImageBlock, MultiCategoryBlock),
                   splitter=ColSplitter(),
                   get_x=lambda o:f'{pascal_source}/train/'+o.fname,
                   get_y=lambda o:o.labels.split(),
                   item_tfms=Resize(224),
                   batch_tfms=aug_transforms())

dls = pascal.dataloaders(df)
dls.show_batch()

# use from_columns
# The most efficient way (to avoid iterating over the rows of the dataframe, which can take a long time) is to use the from_columns method. 
# It will use get_items to convert the columns into numpy arrays. The drawback is that since we lose the dataframe after extracting the relevant columns, 
# we can’t use a ColSplitter anymore. Here we used an IndexSplitter after manually extracting the index of the validation set from the dataframe:
def _pascal_items(x): return (
    f'{pascal_source}/train/'+x.fname, x.labels.str.split())
valid_idx = df[df['is_valid']].index.values

pascal = DataBlock.from_columns(blocks=(ImageBlock, MultiCategoryBlock),
                   get_items=_pascal_items,
                   splitter=IndexSplitter(valid_idx),
                   item_tfms=Resize(224),
                   batch_tfms=aug_transforms())

```

### Image localization - Segmentation
```
path = untar_data(URLs.CAMVID_TINY)
camvid = DataBlock(blocks=(ImageBlock, MaskBlock(codes = np.loadtxt(path/'codes.txt', dtype=str))),
    get_items=get_image_files,
    splitter=RandomSplitter(),
    get_y=lambda o: path/'labels'/f'{o.stem}_P{o.suffix}',
    batch_tfms=aug_transforms())
dls = camvid.dataloaders(path/"images")
dls.show_batch()
```

### Image localization - Points
```
biwi_source = untar_data(URLs.BIWI_SAMPLE)
fn2ctr = load_pickle(biwi_source/'centers.pkl')
biwi = DataBlock(blocks=(ImageBlock, PointBlock),
                 get_items=get_image_files,
                 splitter=RandomSplitter(),
                 get_y=lambda o:fn2ctr[o.name].flip(0),
                 batch_tfms=aug_transforms())
dls = biwi.dataloaders(biwi_source)
dls.show_batch(max_n=9)
```

### Image localization - Bounding boxes
```
coco_source = untar_data(URLs.COCO_TINY)
images, lbl_bbox = get_annotations(coco_source/'train.json')
img2bbox = dict(zip(images, lbl_bbox))
coco = DataBlock(blocks=(ImageBlock, BBoxBlock, BBoxLblBlock),
                 get_items=get_image_files,
                 splitter=RandomSplitter(),
                 get_y=[lambda o: img2bbox[o.name][0], lambda o: img2bbox[o.name][1]], 
                 item_tfms=Resize(128),
                 batch_tfms=aug_transforms(),
                 n_inp=1)
dls = coco.dataloaders(coco_source)
dls.show_batch(max_n=9)
```

### Text
```
from fastai.text.all import *
path = untar_data(URLs.IMDB_SAMPLE)
df = pd.read_csv(path/'texts.csv')
df.head()

imdb_lm = DataBlock(blocks=TextBlock.from_df('text', is_lm=True),
                    get_x=ColReader('text'),
                    splitter=ColSplitter())
dls = imdb_lm.dataloaders(df, bs=64, seq_len=72)
dls.show_batch(max_n=6)

# You need to make sure to use the same seq_len in TextBlock and the Learner you will define later on.
imdb_clas = DataBlock(blocks=(TextBlock.from_df('text', seq_len=72, vocab=dls.vocab), CategoryBlock),
                      get_x=ColReader('text'),
                      get_y=ColReader('label'),
                      splitter=ColSplitter())
dls = imdb_clas.dataloaders(df, bs=64)
dls.show_batch()
```

### Tabular data
```
from fastai.tabular.core import *
adult_source = untar_data(URLs.ADULT_SAMPLE)
df = pd.read_csv(adult_source/'adult.csv')
df.head()

cat_names = ['workclass', 'education', 'marital-status', 'occupation', 'relationship', 'race']
cont_names = ['age', 'fnlwgt', 'education-num']

procs = [Categorify, FillMissing, Normalize]

splits = RandomSplitter()(range_of(df))

to = TabularPandas(df, procs, cat_names, cont_names, y_names="salary", splits=splits, y_block=CategoryBlock)

dls = to.dataloaders()
dls.show_batch()
```
