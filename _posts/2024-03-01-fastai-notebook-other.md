---
layout: article
tags: AI
title: fastai notebook - other
mathjax: true
key: Linux
---

[soruce](https://docs.fast.ai/tutorial.vision.html)
{:.info} 


## fine-tune
```
Fine-tuning: A transfer learning technique where the parameters of a pretrained model are updated by training for additional epochs using a different task to that used for pretraining.

When you use the fine_tune method, fastai will use these tricks for you. There are a few parameters you can set (which we’ll discuss later), but in the default form shown here, it does two steps:

Use one epoch to fit just those parts of the model necessary to get the new random head to work correctly with your dataset.
Use the number of epochs requested when calling the method to fit the entire model, updating the weights of the later layers (especially the head) faster than the earlier layers (which, as we’ll see, generally don’t require many changes from the pretrained weights).
```

## chapter2 

### image
```
ims = ['http://3.bp.blogspot.com/-S1scRCkI3vY/UHzV2kucsPI/AAAAAAAAA-k/YQ5UzHEm9Ss/s1600/Grizzly%2BBear%2BWildlife.jpg']
dest = 'images/grizzly.jpg'
download_url(ims[0], dest)
im = Image.open(dest)
im.to_thumb(128,128)

bear_types = 'grizzly','black','teddy'
path = Path('bears')

if not path.exists():
    path.mkdir()
    for o in bear_types:
        dest = (path/o)
        dest.mkdir(exist_ok=True)
        results = search_images_bing(key, f'{o} bear')
        download_images(dest, urls=results.attrgot('contentUrl'))

fns = get_image_files(path)

failed = verify_images(fns)

failed.map(Path.unlink);
```

### dataloaders
```
# DataLoaders is a thin class that just stores whatever DataLoader objects you pass to it, and makes them available as train and valid.

# For when you don’t, fastai has an extremely flexible system called the data block API. 
# With this API you can fully customize every stage of the creation of your DataLoaders

bears = DataBlock(
    blocks=(ImageBlock, CategoryBlock), 
    get_items=get_image_files, 
    splitter=RandomSplitter(valid_pct=0.2, seed=42),
    get_y=parent_label,
    item_tfms=Resize(128))

bears = bears.new(item_tfms=Resize(128, ResizeMethod.Squish))
dls = bears.dataloaders(path)
dls.valid.show_batch(max_n=4, nrows=1)

bears = bears.new(item_tfms=RandomResizedCrop(128, min_scale=0.3))
dls = bears.dataloaders(path)
dls.train.show_batch(max_n=4, nrows=1, unique=True)

bears = bears.new(item_tfms=Resize(128), batch_tfms=aug_transforms(mult=2))
dls = bears.dataloaders(path)
dls.train.show_batch(max_n=8, nrows=2, unique=True)

bears = bears.new(
    item_tfms=RandomResizedCrop(224, min_scale=0.5),
    batch_tfms=aug_transforms())
dls = bears.dataloaders(path)

learn = vision_learner(dls, resnet18, metrics=error_rate)
learn.fine_tune(4)

```

### Interpretation
```
interp = ClassificationInterpretation.from_learner(learn)
interp.plot_confusion_matrix()

interp.plot_top_losses(5, nrows=1)

cleaner = ImageClassifierCleaner(learn)
cleaner

# to delete (`unlink`) all images selected for deletion, we would run:
for idx in cleaner.delete(): cleaner.fns[idx].unlink()
# To move images for which we've selected a different category, we would run:
for idx,cat in cleaner.change(): shutil.move(str(cleaner.fns[idx]), path/cat)
```

### export
```
learn.export()

path = Path()
path.ls(file_exts='.pkl')

learn_inf = load_learner(path/'export.pkl')

learn_inf.predict('images/grizzly.jpg')

learn_inf.dls.vocab
```

## chapter4 minst basic - see notebook
```
path = untar_data(URLs.MNIST_SAMPLE)
Path.BASE_PATH = path

path.ls()

(path/'train').ls()

threes = (path/'train'/'3').ls().sorted()
sevens = (path/'train'/'7').ls().sorted()

im3_path = threes[1]
im3 = Image.open(im3_path)
im3

array(im3)[4:10,4:10]
tensor(im3)[4:10,4:10]

im3_t = tensor(im3)
df = pd.DataFrame(im3_t[4:15,4:22])
df.style.set_properties(**{'font-size':'6pt'}).background_gradient('Greys')

seven_tensors = [tensor(Image.open(o)) for o in sevens]
three_tensors = [tensor(Image.open(o)) for o in threes]
len(three_tensors),len(seven_tensors)

show_image(three_tensors[1]);

# stack https://www.jianshu.com/p/bc30409f98f7
stacked_sevens = torch.stack(seven_tensors).float()/255
stacked_threes = torch.stack(three_tensors).float()/255
stacked_threes.shape

stacked_threes.ndim

mean3 = stacked_threes.mean(0)
mean7 = stacked_sevens.mean(0)

# Take the mean of the absolute value of differences (absolute value is the function that replaces negative values with positive values). 
# This is called the mean absolute difference or L1 norm

# Take the mean of the square of differences (which makes everything positive) and then take the square root (which undoes the squaring). 
# This is called the root mean squared error (RMSE) or L2 norm
F.l1_loss(a_3.float(),mean7), F.mse_loss(a_3,mean7).sqrt()

valid_3_tens = torch.stack([tensor(Image.open(o)) 
                            for o in (path/'valid'/'3').ls()])
valid_3_tens = valid_3_tens.float()/255
valid_7_tens = torch.stack([tensor(Image.open(o)) 
                            for o in (path/'valid'/'7').ls()])
valid_7_tens = valid_7_tens.float()/255
valid_3_tens.shape,valid_7_tens.shape

def mnist_distance(a,b): return (a-b).abs().mean((-1,-2))

valid_3_dist = mnist_distance(valid_3_tens, mean3)

def is_3(x): return mnist_distance(x,mean3) < mnist_distance(x,mean7)

accuracy_3s =      is_3(valid_3_tens).float() .mean()
accuracy_7s = (1 - is_3(valid_7_tens).float()).mean()

```

### Stochastic Gradient Descent (SGD)
```
def f(x): return x**2
plot_function(f, 'x', 'x**2')
plt.scatter(-1.5, f(-1.5), color='red');

xt = tensor(3.).requires_grad_()
yt = f(xt)
yt

yt.backward()
xt.grad

xt = tensor([3.,4.,10.]).requires_grad_()

def f(x): return (x**2).sum()

yt = f(xt)
yt
yt.backward()
xt.grad

# Stepping With a Learning Rate
w -= gradient(w) * lr
```

## chapter5
```
from fastai.vision.all import *
path = untar_data(URLs.PETS)
Path.BASE_PATH = path

(path/"images").ls()
fname = (path/"images").ls()[0]

re.findall(r'(.+)_\d+.jpg$', fname.name)

pets = DataBlock(blocks = (ImageBlock, CategoryBlock),
                 get_items=get_image_files, 
                 splitter=RandomSplitter(seed=42),
                 get_y=using_attr(RegexLabeller(r'(.+)_\d+.jpg$'), 'name'),
                 item_tfms=Resize(460),
                 batch_tfms=aug_transforms(size=224, min_scale=0.75))
dls = pets.dataloaders(path/"images")

dblock1 = DataBlock(blocks=(ImageBlock(), CategoryBlock()),
                   get_y=parent_label,
                   item_tfms=Resize(460))

# Place an image in the 'images/grizzly.jpg' subfolder where this notebook is located before running this
dls1 = dblock1.dataloaders([(Path.cwd()/'images'/'grizzly.jpg')]*100, bs=8)
dls1.train.get_idxs = lambda: Inf.ones
x,y = dls1.valid.one_batch()
_,axs = subplots(1, 2)

x1 = TensorImage(x.clone())
x1 = x1.affine_coord(sz=224)
x1 = x1.rotate(draw=30, p=1.)
x1 = x1.zoom(draw=1.2, p=1.)
x1 = x1.warp(draw_x=-0.2, draw_y=0.2, p=1.)

tfms = setup_aug_tfms([Rotate(draw=30, p=1, size=224), Zoom(draw=1.2, p=1., size=224),
                       Warp(draw_x=-0.2, draw_y=0.2, p=1., size=224)])
x = Pipeline(tfms)(x)
#x.affine_coord(coord_tfm=coord_tfm, sz=size, mode=mode, pad_mode=pad_mode)
TensorImage(x[0]).show(ctx=axs[0])
TensorImage(x1[0]).show(ctx=axs[1]);
# x[0] is good then x1[0]
```

### use summary to debug datablock 
```
#hide_output
pets1 = DataBlock(blocks = (ImageBlock, CategoryBlock),
                 get_items=get_image_files, 
                 splitter=RandomSplitter(seed=42),
                 get_y=using_attr(RegexLabeller(r'(.+)_\d+.jpg$'), 'name'))
pets1.summary(path/"images")
```

### Cross-Entropy Loss
```
Cross-entropy loss is a loss function that is similar to the one we used in the previous chapter, but (as we’ll see) has two benefits:

It works even when our dependent variable has more than two categories.
It results in faster and more reliable training.


x,y = dls.one_batch()
preds,_ = learn.get_preds(dl=[(x,y)])
preds[0]

plot_function(torch.sigmoid, min=-4,max=4)

torch.random.manual_seed(42);
# 随机生成一个预测
acts = torch.randn((6,2))*2
tensor([[ 0.6734,  0.2576],
        [ 0.4689,  0.4607],
        [-2.2457, -0.3727],
        [ 4.4164, -1.2760],
        [ 0.9233,  0.5347],
        [ 1.0698,  1.6187]])

# sigmoid 把数据按比例压缩到min和max之间，默认0-1
acts.sigmoid()
tensor([[0.6623, 0.5641],
        [0.6151, 0.6132],
        [0.0957, 0.4079],
        [0.9881, 0.2182],
        [0.7157, 0.6306],
        [0.7446, 0.8346]])

# softmax把某个维度的数据压缩到0-1之间，并且保证和是1
sm_acts = torch.softmax(acts, dim=1)
tensor([[0.6025, 0.3975],
        [0.5021, 0.4979],
        [0.1332, 0.8668],
        [0.9966, 0.0034],
        [0.5959, 0.4041],
        [0.3661, 0.6339]])

# Softmax is the first part of the cross-entropy loss
# the second part is log likelihood.

# 一个神奇的数据提取方法
# 假如下面是label
targ = tensor([0,1,0,1,1,0])
idx = range(6)
sm_acts
tensor([[0.6025, 0.3975],
        [0.5021, 0.4979],
        [0.1332, 0.8668],
        [0.9966, 0.0034],
        [0.5959, 0.4041],
        [0.3661, 0.6339]])
sm_acts[idx, targ]
tensor([0.6025, 0.4979, 0.1332, 0.0034, 0.4041, 0.3661])

# PyTorch provides a function that does exactly the same thing as sm_acts[range(n), targ] 
(except it takes the negative, because when applying the log afterward, we will have negative numbers),
 called nll_loss (NLL stands for negative log likelihood):

-sm_acts[idx, targ]
tensor([-0.6025, -0.4979, -0.1332, -0.0034, -0.4041, -0.3661])

F.nll_loss(sm_acts, targ, reduction='none')
tensor([-0.6025, -0.4979, -0.1332, -0.0034, -0.4041, -0.3661])

# The nll in nll_loss stands for “negative log likelihood,” but it doesn’t actually take the log at all! 
# It assumes you have already taken the log. 
# PyTorch has a function called log_softmax that combines log and softmax in a fast and accurate way. 
# nll_loss is designed to be used after log_softmax.

# Taking the mean of the negative log of our probabilities (taking the mean of the loss column of our table) 
# gives us the negative log likelihood loss, which is another name for cross-entropy loss.
# nn.CrossEntropyLoss ctually does log_softmax and then nll_loss
loss_func = nn.CrossEntropyLoss()
loss_func(acts, targ)
tensor(1.8045)

# All PyTorch loss functions are provided in two forms, the class just shown above, and also a plain functional form, available in the `F` namespace:
F.cross_entropy(acts, targ)
tensor(1.8045)

# By default PyTorch loss functions take the mean of the loss of all items. You can use `reduction='none'` to disable that:
nn.CrossEntropyLoss(reduction='none')(acts, targ)
tensor([0.5067, 0.6973, 2.0160, 5.6958, 0.9062, 1.0048])

```

### Model Interpretation
```
#width 600
interp = ClassificationInterpretation.from_learner(learn)
interp.plot_confusion_matrix(figsize=(12,12), dpi=60)
interp.most_confused(min_val=5)
```

### The Learning Rate Finder
```
learn = vision_learner(dls, resnet34, metrics=error_rate)
learn.fine_tune(1, base_lr=0.1)

That doesn’t look good.

learn = vision_learner(dls, resnet34, metrics=error_rate)
lr_min,lr_steep = learn.lr_find(suggest_funcs=(minimum, steep))

print(f"Minimum/10: {lr_min:.2e}, steepest point: {lr_steep:.2e}")
Minimum/10: 1.00e-02, steepest point: 5.25e-03

learn = vision_learner(dls, resnet34, metrics=error_rate)
learn.fine_tune(2, base_lr=3e-3)

```

### Unfreezing and Transfer Learning
```
When we create a model from a pretrained network fastai automatically freezes all of the pretrained layers for us. When we call the fine_tune method fastai does two things:

Trains the randomly added layers for one epoch, with all other layers frozen
Unfreezes all of the layers, and trains them all for the number of epochs requested

fit_one_cycle is the suggested way to train models without using fine_tune. We’ll see why later in the book; 
in short, what fit_one_cycle does is to start training at a low learning rate, gradually increase it for the first section of training, 
and then gradually decrease it again for the last section of training.

learn = vision_learner(dls, resnet34, metrics=error_rate)
learn.fit_one_cycle(3, 3e-3)

learn.unfreeze()

learn.lr_find()

learn.fit_one_cycle(6, lr_max=1e-5)

# Discriminative Learning Rates
# fastai lets you pass a Python slice object anywhere that a learning rate is expected. 
# The first value passed will be the learning rate in the earliest layer of the neural network, 
# and the second value will be the learning rate in the final layer. The layers in between will 
# have learning rates that are multiplicatively equidistant throughout that range. 
# Let’s use this approach to replicate the previous training, but this time we’ll only set the lowest 
# layer of our net to a learning rate of 1e-6; the other layers will scale up to 1e-4. Let’s train for a while and see what happens:
learn = vision_learner(dls, resnet34, metrics=error_rate)
learn.fit_one_cycle(3, 3e-3)
learn.unfreeze()
learn.fit_one_cycle(12, lr_max=slice(1e-6,1e-4))

# fastai can show us a graph of the training and validation loss:
learn.recorder.plot_loss()

# use freeze_epochs
from fastai.callback.fp16 import *
learn = vision_learner(dls, resnet50, metrics=error_rate).to_fp16()
learn.fine_tune(6, freeze_epochs=3)
# You’ll see here we’ve gone back to using fine_tune, since it’s so handy! 
# We can pass freeze_epochs to tell fastai how many epochs to train for while frozen. It will automatically change learning rates appropriately for most datasets.
```

## chapter6
```
multi-label classification and regression. 
The first one is when you want to predict more than one label per image (or sometimes none at all), 
and the second is when your labels are one or several numbers—a quantity instead of a category.

```

### multi-label
```
from fastai.vision.all import *
path = untar_data(URLs.PASCAL_2007)

df = pd.read_csv(path/'train.csv')
df.head()

df.iloc[:,0]
df.iloc[0,:]
df['fname']
tmp_df = pd.DataFrame({'a':[1,2], 'b':[3,4]})
tmp_df
tmp_df['c'] = tmp_df['a']+tmp_df['b']
tmp_df

Dataset:: A collection that returns a tuple of your independent and dependent variable for a single item
DataLoader:: An iterator that provides a stream of mini-batches, where each mini-batch is a tuple of a batch of independent variables and a batch of dependent variables
Datasets:: An object that contains a training Dataset and a validation Dataset
DataLoaders:: An object that contains a training DataLoader and a validation DataLoader

dblock = DataBlock()
dsets = dblock.datasets(df)
len(dsets.train),len(dsets.valid)
x,y = dsets.train[0]
x,y
# 此处x y相同，因为没有传入get_x get_y

def get_x(r): return path/'train'/r['fname']
def get_y(r): return r['labels'].split(' ')
dblock = DataBlock(get_x = get_x, get_y = get_y)
dsets = dblock.datasets(df)
dsets.train[0]


dblock = DataBlock(blocks=(ImageBlock, MultiCategoryBlock),
                   get_x = get_x, get_y = get_y)
dsets = dblock.datasets(df)
dsets.train[0

idxs = torch.where(dsets.train[0][1]==1.)[0]
dsets.train.vocab[idxs]

def splitter(df):
    train = df.index[~df['is_valid']].tolist()
    valid = df.index[df['is_valid']].tolist()
    return train,valid

dblock = DataBlock(blocks=(ImageBlock, MultiCategoryBlock),
                   splitter=splitter,
                   get_x=get_x, 
                   get_y=get_y,
                   item_tfms = RandomResizedCrop(128, min_scale=0.35))
dls = dblock.dataloaders(df)

dls.show_batch(nrows=1, ncols=3)

learn = vision_learner(dls, resnet18)
x,y = to_cpu(dls.train.one_batch())
activs = learn.model(x)
activs.shape

# multi-label 问题不能使用nll_loss 和softmax 和 cross_entropy
# F.binary_cross_entropy and its module equivalent nn.BCELoss calculate cross-entropy on a one-hot-encoded target, but do not include the initial sigmoid. 
# Normally for one-hot-encoded targets you’ll want F.binary_cross_entropy_with_logits (or nn.BCEWithLogitsLoss), 
# which do both sigmoid and binary cross-entropy in a single function, as in the preceding example.

loss_func = nn.BCEWithLogitsLoss()
loss = loss_func(activs, y)
loss

def accuracy_multi(inp, targ, thresh=0.5, sigmoid=True):
    "Compute accuracy when `inp` and `targ` are the same size."
    if sigmoid: inp = inp.sigmoid()
    return ((inp>thresh)==targ.bool()).float().mean()

learn = vision_learner(dls, resnet50, metrics=partial(accuracy_multi, thresh=0.2))
learn.fine_tune(3, base_lr=3e-3, freeze_epochs=4)

# Picking a threshold is important. If you pick a threshold that’s too low, you’ll often be failing to select correctly labeled objects. 
# We can see this by changing our metric, and then calling validate, which returns the validation loss and metrics:
learn.metrics = partial(accuracy_multi, thresh=0.1)
learn.validate()

# If you pick a threshold that’s too high, you’ll only be selecting the objects for which your model is very confident:
learn.metrics = partial(accuracy_multi, thresh=0.99)
learn.validate()

preds,targs = learn.get_preds()
accuracy_multi(preds, targs, thresh=0.9, sigmoid=False)

# We can now use this approach to find the best threshold level:
xs = torch.linspace(0.05,0.95,29)
accs = [accuracy_multi(preds, targs, thresh=i, sigmoid=False) for i in xs]
plt.plot(xs,accs);

```

## chapter 8
```
检查电影属性和用户属性的匹配度。

from fastai.collab import *
from fastai.tabular.all import *
path = untar_data(URLs.ML_100k)

ratings = pd.read_csv(path/'u.data', delimiter='\t', header=None,
                      names=['user','movie','rating','timestamp'])
ratings.head()
	user	movie	rating	timestamp
0	196	242	3	881250949
1	186	302	3	891717742
2	22	377	1	878887116
3	244	51	2	880606923
4	166	346	1	886397596

检查电影属性和用户属性的匹配度。 越大越好
last_skywalker = np.array([0.98,0.9,-0.9])
user1 = np.array([0.9,0.8,-0.6])
(user1*last_skywalker).sum()

movies = pd.read_csv(path/'u.item',  delimiter='|', encoding='latin-1',
                     usecols=(0,1), names=('movie','title'), header=None)
movies.head()
	movie	title
0	1	Toy Story (1995)
1	2	GoldenEye (1995)
2	3	Four Rooms (1995)
3	4	Get Shorty (1995)
4	5	Copycat (1995)

ratings = ratings.merge(movies)
ratings.head()
	user	movie	rating	timestamp	title
0	196	242	3	881250949	Kolya (1996)
1	186	302	3	891717742	L.A. Confidential (1997)
2	22	377	1	878887116	Heavyweights (1994)
3	244	51	2	880606923	Legends of the Fall (1994)
4	166	346	1	886397596	Jackie Brown (1997)

dls = CollabDataLoaders.from_df(ratings, item_name='title', bs=64)
dls.show_batch()
        user	title	rating
0	325	Good Man in Africa, A (1994)	4
1	752	Air Force One (1997)	3
2	757	Cyrano de Bergerac (1990)	2
3	660	Jurassic Park (1993)	2
4	483	North by Northwest (1959)	3
5	625	Independence Day (ID4) (1996)	3
6	454	Robin Hood: Prince of Thieves (1991)	2
7	387	Waiting for Guffman (1996)	5
8	26	Broken Arrow (1996)	2
9	425	Wag the Dog (1997)	4


n_users  = len(dls.classes['user'])
n_movies = len(dls.classes['title'])
n_factors = 5

user_factors = torch.randn(n_users, n_factors)
movie_factors = torch.randn(n_movies, n_factors)

one_hot_3 = one_hot(3, n_users).float()
user_factors.t() @ one_hot_3
tensor([0.3188, 0.3033, 0.1449, 0.9004, 0.5670])
# It gives us the same vector as the one at index 3 in the matrix:
user_factors[3]
tensor([0.3188, 0.3033, 0.1449, 0.9004, 0.5670])

# Embedding: Multiplying by a one-hot-encoded matrix, using the computational shortcut that it can be implemented by simply indexing directly. 
# This is quite a fancy word for a very simple concept. The thing that you multiply the one-hot-encoded matrix by 
# (or, using the computational shortcut, index into directly) is called the embedding matrix.
```

### Collaborative Filtering from Scratch
```
class DotProductBias(Module):
    def __init__(self, n_users, n_movies, n_factors, y_range=(0,5.5)):
        self.user_factors = Embedding(n_users, n_factors)
        self.user_bias = Embedding(n_users, 1)
        self.movie_factors = Embedding(n_movies, n_factors)
        self.movie_bias = Embedding(n_movies, 1)
        self.y_range = y_range
        
    def forward(self, x):
        users = self.user_factors(x[:,0])
        movies = self.movie_factors(x[:,1])
        res = (users * movies).sum(dim=1, keepdim=True)
        res += self.user_bias(x[:,0]) + self.movie_bias(x[:,1])
        return sigmoid_range(res, *self.y_range)

model = DotProductBias(n_users, n_movies, 50)
learn = Learner(dls, model, loss_func=MSELossFlat())
learn.fit_one_cycle(5, 5e-3)

# To use weight decay in fastai, just pass wd in your call to fit or fit_one_cycle:
model = DotProductBias(n_users, n_movies, 50)
learn = Learner(dls, model, loss_func=MSELossFlat())
learn.fit_one_cycle(5, 5e-3, wd=0.1)
```

### Creating Our Own Embedding Module
```
def create_params(size):
    return nn.Parameter(torch.zeros(*size).normal_(0, 0.01))

class DotProductBias(Module):
    def __init__(self, n_users, n_movies, n_factors, y_range=(0,5.5)):
        self.user_factors = create_params([n_users, n_factors])
        self.user_bias = create_params([n_users])
        self.movie_factors = create_params([n_movies, n_factors])
        self.movie_bias = create_params([n_movies])
        self.y_range = y_range
        
    def forward(self, x):
        users = self.user_factors[x[:,0]]
        movies = self.movie_factors[x[:,1]]
        res = (users*movies).sum(dim=1)
        res += self.user_bias[x[:,0]] + self.movie_bias[x[:,1]]
        return sigmoid_range(res, *self.y_range)

model = DotProductBias(n_users, n_movies, 50)
learn = Learner(dls, model, loss_func=MSELossFlat())
learn.fit_one_cycle(5, 5e-3, wd=0.1)


```

#### Interpreting Embeddings and Biases
```
movie_bias = learn.model.movie_bias.squeeze()
idxs = movie_bias.argsort()[:5]
[dls.classes['title'][i] for i in idxs]
['Children of the Corn: The Gathering (1996)',
 'Lawnmower Man 2: Beyond Cyberspace (1996)',
 'Beautician and the Beast, The (1997)',
 'Crow: City of Angels, The (1996)',
 'Home Alone 3 (1997)']

idxs = movie_bias.argsort(descending=True)[:5]
[dls.classes['title'][i] for i in idxs]
['L.A. Confidential (1997)',
 'Titanic (1997)',
 'Silence of the Lambs, The (1991)',
 'Shawshank Redemption, The (1994)',
 'Star Wars (1977)']
So, for instance, even if you don't normally enjoy detective movies, you might enjoy *LA Confidential*!

# picture
g = ratings.groupby('title')['rating'].count()
top_movies = g.sort_values(ascending=False).index.values[:1000]
top_idxs = tensor([learn.dls.classes['title'].o2i[m] for m in top_movies])
movie_w = learn.model.movie_factors[top_idxs].cpu().detach()
movie_pca = movie_w.pca(3)
fac0,fac1,fac2 = movie_pca.t()
idxs = list(range(50))
X = fac0[idxs]
Y = fac2[idxs]
plt.figure(figsize=(12,12))
plt.scatter(X, Y)
for i, x, y in zip(top_movies[idxs], X, Y):
    plt.text(x,y,i, color=np.random.rand(3)*0.7, fontsize=11)
plt.show()

```

#### Using fastai.collab
```
learn = collab_learner(dls, n_factors=50, y_range=(0, 5.5))
learn.fit_one_cycle(5, 5e-3, wd=0.1)
# The names of the layers can be seen by printing the model:
learn.model
EmbeddingDotBias(
  (u_weight): Embedding(944, 50)
  (i_weight): Embedding(1635, 50)
  (u_bias): Embedding(944, 1)
  (i_bias): Embedding(1635, 1)
)

# We can use these to replicate any of the analyses we did in the previous section—for instance:

movie_bias = learn.model.i_bias.weight.squeeze()
idxs = movie_bias.argsort(descending=True)[:5]
[dls.classes['title'][i] for i in idxs]
['Titanic (1997)',
 "Schindler's List (1993)",
 'Shawshank Redemption, The (1994)',
 'L.A. Confidential (1997)',
 'Silence of the Lambs, The (1991)']
```

#### Embedding Distance
```
movie_factors = learn.model.i_weight.weight
idx = dls.classes['title'].o2i['Silence of the Lambs, The (1991)']
distances = nn.CosineSimilarity(dim=1)(movie_factors, movie_factors[idx][None])
idx = distances.argsort(descending=True)[1]
dls.classes['title'][idx]
'Dial M for Murder (1954)'
```

#### use more layers
```
# fastai has a function get_emb_sz that returns recommended sizes for embedding matrices for your data, based on a heuristic that fast.ai has found tends to work well in practice:

embs = get_emb_sz(dls)
embs
[(944, 74), (1635, 101)]

class CollabNN(Module):
    def __init__(self, user_sz, item_sz, y_range=(0,5.5), n_act=100):
        self.user_factors = Embedding(*user_sz)
        self.item_factors = Embedding(*item_sz)
        self.layers = nn.Sequential(
            nn.Linear(user_sz[1]+item_sz[1], n_act),
            nn.ReLU(),
            nn.Linear(n_act, 1))
        self.y_range = y_range
        
    def forward(self, x):
        embs = self.user_factors(x[:,0]),self.item_factors(x[:,1])
        x = self.layers(torch.cat(embs, dim=1))
        return sigmoid_range(x, *self.y_range)
model = CollabNN(*embs)
learn = Learner(dls, model, loss_func=MSELossFlat())
learn.fit_one_cycle(5, 5e-3, wd=0.01)

# fastai provides this model in `fastai.collab` if you pass `use_nn=True` in your call to `collab_learner` (including calling `get_emb_sz` for you), and it lets you easily create more layers. 
# For instance, here we're creating two hidden layers, of size 100 and 50, respectively:
learn = collab_learner(dls, use_nn=True, y_range=(0, 5.5), layers=[100,50])
learn.fit_one_cycle(5, 5e-3, wd=0.1)

```
