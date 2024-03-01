---
layout: article
tags: AI
title: fastai notebook - mid tier api
mathjax: true
key: Linux
---

[soruce](https://docs.fast.ai/tutorial.vision.html)
{:.info} 


## Transform
```
from fastai.text.all import *
from fastai.vision.all import *

source = untar_data(URLs.MNIST_TINY)/'train'
items = get_image_files(source)
fn = items[0]; fn

img = PILImage.create(fn); img
tconv = ToTensor()
img = tconv(img)
img.shape,type(img)

lbl = parent_label(fn); lbl

tcat = Categorize(vocab=['3','7'])
lbl = tcat(lbl); lbl

lbld = tcat.decode(lbl)
lbld
```

## pipeline
```
pipe = Pipeline([PILImage.create,tconv])
img = pipe(fn)
img.shape

pipe.show(img, figsize=(1,1), cmap='Greys');
```

## Loading the pets dataset using only Transform
```
source = untar_data(URLs.PETS)/"images"
items = get_image_files(source)

def resized_image(fn:Path, sz=128):
    x = Image.open(fn).convert('RGB').resize((sz,sz))
    # Convert image to tensor for modeling
    return tensor(array(x)).permute(2,0,1).float()/255.

class TitledImage(fastuple):
    def show(self, ctx=None, **kwargs): show_titled_image(self, ctx=ctx, **kwargs)

img = resized_image(items[0])
TitledImage(img,'test title').show()

class PetTfm(Transform):
    def __init__(self, vocab, o2i, lblr): self.vocab,self.o2i,self.lblr = vocab,o2i,lblr
    def encodes(self, o): return [resized_image(o), self.o2i[self.lblr(o)]]
    def decodes(self, x): return TitledImage(x[0],self.vocab[x[1]])

labeller = using_attr(RegexLabeller(pat = r'^(.*)_\d+.jpg$'), 'name')

vals = list(map(labeller, items))
vocab,o2i = uniqueify(vals, sort=True, bidir=True)
pets = PetTfm(vocab,o2i,labeller)

x,y = pets(items[0])
x.shape,y

dec = pets.decode([x,y])
dec.show()

# to prevent a Transform from dispatching over a tuple, we just have to make it an ItemTransform:
class PetTfm(ItemTransform):
    def __init__(self, vocab, o2i, lblr): self.vocab,self.o2i,self.lblr = vocab,o2i,lblr
    def encodes(self, o): return (resized_image(o), self.o2i[self.lblr(o)])
    def decodes(self, x): return TitledImage(x[0],self.vocab[x[1]])
dec = pets.decode(pets(items[0]))
dec.show()

# Setting up the internal state with a setups
class PetTfm(ItemTransform):
    def setups(self, items):
        self.labeller = using_attr(RegexLabeller(pat = r'^(.*)_\d+.jpg$'), 'name')
        vals = map(self.labeller, items)
        self.vocab,self.o2i = uniqueify(vals, sort=True, bidir=True)

    def encodes(self, o): return (resized_image(o), self.o2i[self.labeller(o)])
    def decodes(self, x): return TitledImage(x[0],self.vocab[x[1]])

# Combining our Transform with data augmentation in a Pipeline.
# We can take advantage of fastai’s data augmentation transforms if we give the right type to our elements. 
# Instead of returning a standard PIL.Image, if our transform returns the fastai type PILImage, we can then use any fastai’s transform with it. Let’s just return a PILImage for our first element:
class PetTfm(ItemTransform):
    def setups(self, items):
        self.labeller = using_attr(RegexLabeller(pat = r'^(.*)_\d+.jpg$'), 'name')
        vals = map(self.labeller, items)
        self.vocab,self.o2i = uniqueify(vals, sort=True, bidir=True)

    def encodes(self, o): return (PILImage.create(o), self.o2i[self.labeller(o)])
    def decodes(self, x): return TitledImage(x[0],self.vocab[x[1]])

tfms = Pipeline([PetTfm(), Resize(224), FlipItem(p=1), ToTensor()])
tfms.setup(items)
tfms.vocab
x,y = tfms(items[0])
x.shape,y
tfms.show(tfms(items[0]))

# To customize the order of a Transform, just set order = ... before the __init__ (it’s a class attribute). Let’s make PetTfm of order -5 to be sure it’s always run first:
class PetTfm(ItemTransform):
    order = -5
    def setups(self, items):
        self.labeller = using_attr(RegexLabeller(pat = r'^(.*)_\d+.jpg$'), 'name')
        vals = map(self.labeller, items)
        self.vocab,self.o2i = uniqueify(vals, sort=True, bidir=True)

    def encodes(self, o): return (PILImage.create(o), self.o2i[self.labeller(o)])
    def decodes(self, x): return TitledImage(x[0],self.vocab[x[1]])
tfms = Pipeline([Resize(224), PetTfm(), FlipItem(p=1), ToTensor()])
tfms
> Pipeline: PetTfm -> FlipItem -- {'p': 1} -> Resize -- {'size': (224, 224), 'method': 'crop', 'pad_mode': 'reflection', 'resamples': (<Resampling.BILINEAR: 2>, <Resampling.NEAREST: 0>), 'p': 1.0} -> ToTensor
```

## One pipeline makes a TfmdLists
```
tls = TfmdLists(items, [Resize(224), PetTfm(), FlipItem(p=0.5), ToTensor()])
x,y = tls[0]
x.shape,y
tls.vocab
tls.show((x,y))
show_at(tls, 0)

# Traning and validation set
splits = RandomSplitter(seed=42)(items)
splits
tls = TfmdLists(items, [Resize(224), PetTfm(), FlipItem(p=0.5), ToTensor()], splits=splits)
show_at(tls.train, 0)
show_at(tls.train, 0)

dls = tls.dataloaders(bs=64)
dls.show_batch()

# augmentation
dls = tls.dataloaders(bs=64, after_batch=[IntToFloatTensor(), *aug_transforms()])
dls.show_batch()

```

## Using Datasets
```
class ImageResizer(Transform):
    order=1
    "Resize image to `size` using `resample`"
    def __init__(self, size, resample=BILINEAR):
        if not is_listy(size): size=(size,size)
        self.size,self.resample = (size[1],size[0]),resample

    def encodes(self, o:PILImage): return o.resize(size=self.size, resample=self.resample)
    def encodes(self, o:PILMask):  return o.resize(size=self.size, resample=NEAREST)

tfms = [[PILImage.create, ImageResizer(128), ToTensor(), IntToFloatTensor()],
        [labeller, Categorize()]]
dsets = Datasets(items, tfms)
t = dsets[0]
type(t[0]),type(t[1])
x,y = dsets.decode(t)
x.shape,y
dsets.show(t);

dsets = Datasets(items, tfms, splits=splits)

tfms = [[PILImage.create], [labeller, Categorize()]]
dsets = Datasets(items, tfms, splits=splits)
dls = dsets.dataloaders(bs=64, after_item=[ImageResizer(128), ToTensor(), IntToFloatTensor()])
dls.show_batch()

# If we just wanted to build one DataLoader from our Datasets (or the previous TfmdLists), you can pass it directly to TfmdDL:
dsets = Datasets(items, tfms)
dl = TfmdDL(dsets, bs=64, after_item=[ImageResizer(128), ToTensor(), IntToFloatTensor()])
```

## Adding a test dataloader for inference
```
tfms = [[PILImage.create], [labeller, Categorize()]]
dsets = Datasets(items, tfms, splits=splits)
dls = dsets.dataloaders(bs=64, after_item=[ImageResizer(128), ToTensor(), IntToFloatTensor()])

…and imagine we have some new files to classify.

path = untar_data(URLs.PETS)
tst_files = get_image_files(path/"images")

We can create a dataloader that takes those files and applies the same transforms as the validation set with DataLoaders.test_dl:

tst_dl = dls.test_dl(tst_files)

tst_dl.show_batch(max_n=9)

Extra:
You can call learn.get_preds passing this newly created dataloaders to make predictions on our new images!
What is really cool is that after you finished training your model, you can save it with learn.export, this is also going to save all the transforms that need to be applied to your data. In inference time you just need to load your learner with load_learner and you can immediately create a dataloader with test_dl to use it to generate new predictions!

```

## Purely in PyTorch
```
from fastai.data.external import untar_data,URLs
from fastai.data.transforms import get_image_files

path = untar_data(URLs.PETS)
files = get_image_files(path/"images")
files[0]

import PIL

img = PIL.Image.open(files[0])
img

import torch
import numpy as np

def open_image(fname, size=224):
    img = PIL.Image.open(fname).convert('RGB')
    img = img.resize((size, size))
    t = torch.Tensor(np.array(img))
    return t.permute(2,0,1).float()/255.0

import re

def label_func(fname):
    return re.match(r'^(.*)_\d+.jpg$', fname.name).groups()[0]

label_func(files[0])

labels = list(set(files.map(label_func)))
len(labels)

lbl2files = {l: [f for f in files if label_func(f) == l] for l in labels}

import random

class SiameseDataset(torch.utils.data.Dataset):
    def __init__(self, files, is_valid=False):
        self.files,self.is_valid = files,is_valid
        if is_valid: self.files2 = [self._draw(f) for f in files]
        
    def __getitem__(self, i):
        file1 = self.files[i]
        (file2,same) = self.files2[i] if self.is_valid else self._draw(file1)
        img1,img2 = open_image(file1),open_image(file2)
        return (img1, img2, torch.Tensor([same]).squeeze())
    
    def __len__(self): return len(self.files)
    
    def _draw(self, f):
        same = random.random() < 0.5
        cls = label_func(f)
        if not same: cls = random.choice([l for l in labels if l != cls]) 
        return random.choice(lbl2files[cls]),same

idxs = np.random.permutation(range(len(files)))
cut = int(0.8 * len(files))
train_files = files[idxs[:cut]]
valid_files = files[idxs[cut:]]

train_ds = SiameseDataset(train_files)
valid_ds = SiameseDataset(valid_files, is_valid=True)

from fastai.data.core import DataLoaders

dls = DataLoaders.from_dsets(train_ds, valid_ds)

b = dls.one_batch()

# If you want to use the GPU, you can just write:
dls = dls.cuda()
```

## Using the mid-level API
```
class SiameseTransform(Transform):
    def __init__(self, files, is_valid=False):
        self.files,self.is_valid = files,is_valid
        if is_valid: self.files2 = [self._draw(f) for f in files]
        
    def encodes(self, i):
        file1 = self.files[i]
        (file2,same) = self.files2[i] if self.is_valid else self._draw(file1)
        img1,img2 = open_image(file1),open_image(file2)
        return (TensorImage(img1), TensorImage(img2), torch.Tensor([same]).squeeze())
    
    def _draw(self, f):
        same = random.random() < 0.5
        cls = label_func(f)
        if not same: cls = random.choice([l for l in labels if l != cls]) 
        return random.choice(lbl2files[cls]),same

train_tl= TfmdLists(range(len(train_files)), SiameseTransform(train_files))
valid_tl= TfmdLists(range(len(valid_files)), SiameseTransform(valid_files, is_valid=True))

dls = DataLoaders.from_dsets(train_tl, valid_tl, 
                             after_batch=[Normalize.from_stats(*imagenet_stats), *aug_transforms()])
dls = dls.cuda()


# make show work
class SiameseImage(fastuple):
    def show(self, ctx=None, **kwargs): 
        if len(self) > 2:
            img1,img2,similarity = self
        else:
            img1,img2 = self
            similarity = 'Undetermined'
        if not isinstance(img1, Tensor):
            if img2.size != img1.size: img2 = img2.resize(img1.size)
            t1,t2 = tensor(img1),tensor(img2)
            t1,t2 = t1.permute(2,0,1),t2.permute(2,0,1)
        else: t1,t2 = img1,img2
        line = t1.new_zeros(t1.shape[0], t1.shape[1], 10)
        return show_image(torch.cat([t1,line,t2], dim=2), title=similarity, ctx=ctx, **kwargs)

img = PILImage.create(files[0])
img1 = PILImage.create(files[1])
s = SiameseImage(img, img1, False)
s.show();

# Now let’s rewrite a bit our previous transform. Instead of taking integers, we can take files directly for instance. 
class SiameseTransform(Transform):
    def __init__(self, files, splits):
        self.splbl2files = [{l: [f for f in files[splits[i]] if label_func(f) == l] for l in labels}
                          for i in range(2)]
        self.valid = {f: self._draw(f,1) for f in files[splits[1]]}
    def encodes(self, f):
        f2,same = self.valid.get(f, self._draw(f,0))
        img1,img2 = PILImage.create(f),PILImage.create(f2)
        return SiameseImage(img1, img2, same)
    
    def _draw(self, f, split=0):
        same = random.random() < 0.5
        cls = label_func(f)
        if not same: cls = random.choice(L(l for l in labels if l != cls)) 
        return random.choice(self.splbl2files[split][cls]),same

# Then we create our splits using a RandomSplitter:
splits = RandomSplitter()(files)
tfm = SiameseTransform(files, splits)

valids = [v[0] for k,v in tfm.valid.items()]      
assert not [v for v in valids if v in files[splits[0]]]

tls = TfmdLists(files, tfm, splits=splits)
show_at(tls.valid, 0)
dls = tls.dataloaders(after_item=[Resize(224), ToTensor], 
                      after_batch=[IntToFloatTensor, Normalize.from_stats(*imagenet_stats)])
b = dls.one_batch()
type(b)
show_batch(x, y, samples, ctxs=None, **kwargs)

```

## Writing your custom data block
```
# Let’s create a type to represent our tuple of two images:
class ImageTuple(fastuple):
    @classmethod
    def create(cls, fns): return cls(tuple(PILImage.create(f) for f in fns))
    
    def show(self, ctx=None, **kwargs): 
        t1,t2 = self
        if not isinstance(t1, Tensor) or not isinstance(t2, Tensor) or t1.shape != t2.shape: return ctx
        line = t1.new_zeros(t1.shape[0], t1.shape[1], 10)
        return show_image(torch.cat([t1,line,t2], dim=2), ctx=ctx, **kwargs)

img = ImageTuple.create((files[0], files[1]))
tst = ToTensor()(img)
type(tst[0]),type(tst[1])

img1 = Resize(224)(img)
tst = ToTensor()(img1)
tst.show();

# We can now define a block associated to ImageTuple that we will use in the data block API. 
# A block is basically a set of default transforms, here we specify how to create the ImageTuple and the IntToFloatTensor transform necessary for image preprocessing:

def ImageTupleBlock(): return TransformBlock(type_tfms=ImageTuple.create, batch_tfms=IntToFloatTensor)

splits_files = [files[splits[i]] for i in range(2)]
splits_sets = mapped(set, splits_files)

def get_split(f):
    for i,s in enumerate(splits_sets):
        if f in s: return i
    raise ValueError(f'File {f} is not presented in any split.')

splbl2files = [{l: [f for f in s if label_func(f) == l] for l in labels} for s in splits_sets]

def splitter(items): 
    def get_split_files(i): return [j for j,(f1,f2,same) in enumerate(items) if get_split(f1)==i]
    return get_split_files(0),get_split_files(1)

def draw_other(f):
    same = random.random() < 0.5
    cls = label_func(f)
    split = get_split(f)
    if not same: cls = random.choice(L(l for l in labels if l != cls)) 
    return random.choice(splbl2files[split][cls]),same

def get_tuples(files): return [[f, *draw_other(f)] for f in files]

def get_x(t): return t[:2]
def get_y(t): return t[2]

siamese = DataBlock(
    blocks=(ImageTupleBlock, CategoryBlock),
    get_items=get_tuples,
    get_x=get_x, get_y=get_y,
    splitter=splitter,
    item_tfms=Resize(224),
    batch_tfms=[Normalize.from_stats(*imagenet_stats)]
)

dls = siamese.dataloaders(files)

b = dls.one_batch()
explode_types(b)


# The show_batch method here works out of the box, but to customize how things are organized, we can define a dispatched show_batch function. 
@typedispatch
def show_batch(x:ImageTuple, y, samples, ctxs=None, max_n=6, nrows=None, ncols=2, figsize=None, **kwargs):
    if figsize is None: figsize = (ncols*6, max_n//ncols * 3)
    if ctxs is None: ctxs = get_grid(min(len(samples), max_n), nrows=nrows, ncols=ncols, figsize=figsize)
    ctxs = show_batch[object](x, y, samples, ctxs=ctxs, max_n=max_n, **kwargs)
    return ctxs

dls.show_batch()
```

## Training a model
```
class SiameseModel(Module):
    def __init__(self, encoder, head):
        self.encoder,self.head = encoder,head
    
    def forward(self, x1, x2):
        ftrs = torch.cat([self.encoder(x1), self.encoder(x2)], dim=1)
        return self.head(ftrs)

# If we want to check where fastai usually cuts the model, we can have a look at the model_meta dictionary:
model_meta[resnet34]
> {'cut': -2,
 'split': <function fastai.vision.learner._resnet_split(m)>,
 'stats': ([0.485, 0.456, 0.406], [0.229, 0.224, 0.225])}

encoder = create_body(resnet34, cut=-2)

# Let’s have a look at the last block of this encoder:
encoder[-1]

head = create_head(512*2, 2, ps=0.5)
model = SiameseModel(encoder, head)

def siamese_splitter(model):
    return [params(model.encoder), params(model.head)]

def loss_func(out, targ):
    return CrossEntropyLossFlat()(out, targ.long())

class SiameseTransform(Transform):
    def __init__(self, files, splits):
        self.splbl2files = [{l: [f for f in files[splits[i]] if label_func(f) == l] for l in labels}
                          for i in range(2)]
        self.valid = {f: self._draw(f,1) for f in files[splits[1]]}
    def encodes(self, f):
        f2,same = self.valid.get(f, self._draw(f,0))
        img1,img2 = PILImage.create(f),PILImage.create(f2)
        return SiameseImage(img1, img2, int(same))
    
    def _draw(self, f, split=0):
        same = random.random() < 0.5
        cls = label_func(f)
        if not same: cls = random.choice(L(l for l in labels if l != cls)) 
        return random.choice(self.splbl2files[split][cls]),same


splits = RandomSplitter()(files)
tfm = SiameseTransform(files, splits)
tls = TfmdLists(files, tfm, splits=splits)
dls = tls.dataloaders(after_item=[Resize(224), ToTensor], 
                      after_batch=[IntToFloatTensor, Normalize.from_stats(*imagenet_stats)])

valids = [v[0] for k,v in tfm.valid.items()]      
assert not [v for v in valids if v in files[splits[0]]]

learn = Learner(dls, model, loss_func=CrossEntropyLossFlat(), splitter=siamese_splitter, metrics=accuracy)

learn.freeze()

learn.lr_find()

learn.fit_one_cycle(4, 3e-3)

learn.unfreeze()

learn.fit_one_cycle(4, slice(1e-6,1e-4))

```

## Making show_results work
```
@typedispatch
def show_results(x:SiameseImage, y, samples, outs, ctxs=None, max_n=6, nrows=None, ncols=2, figsize=None, **kwargs):
    if figsize is None: figsize = (ncols*6, max_n//ncols * 3)
    if ctxs is None: ctxs = get_grid(min(x[0].shape[0], max_n), nrows=None, ncols=ncols, figsize=figsize)
    for i,ctx in enumerate(ctxs): 
        title = f'Actual: {["Not similar","Similar"][x[2][i].item()]} \n Prediction: {["Not similar","Similar"][y[2][i].item()]}'
        SiameseImage(x[0][i], x[1][i], title).show(ctx=ctx)
```
