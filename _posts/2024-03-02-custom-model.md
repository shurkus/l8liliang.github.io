---
layout: article
tags: AI
title: fastai notebook - write a custom model
mathjax: true
key: Linux
---

[soruce](https://docs.fast.ai/tutorial.vision.html)
{:.info} 


## write a custom model
```
from fastai.vision.all import *
path = untar_data(URLs.PETS)
files = get_image_files(path/"images")

class SiameseImage(fastuple):
    def show(self, ctx=None, **kwargs): 
        img1,img2,same_breed = self
        if not isinstance(img1, Tensor):
            if img2.size != img1.size: img2 = img2.resize(img1.size)
            t1,t2 = tensor(img1),tensor(img2)
            t1,t2 = t1.permute(2,0,1),t2.permute(2,0,1)
        else: t1,t2 = img1,img2
        line = t1.new_zeros(t1.shape[0], t1.shape[1], 10)
        return show_image(torch.cat([t1,line,t2], dim=2), 
                          title=same_breed, ctx=ctx)
    
def label_func(fname):
    return re.match(r'^(.*)_\d+.jpg$', fname.name).groups()[0]

class SiameseTransform(Transform):
    def __init__(self, files, label_func, splits):
        self.labels = files.map(label_func).unique()
        self.lbl2files = {l: L(f for f in files if label_func(f) == l) for l in self.labels}
        self.label_func = label_func
        self.valid = {f: self._draw(f) for f in files[splits[1]]}
        
    def encodes(self, f):
        f2,t = self.valid.get(f, self._draw(f))
        img1,img2 = PILImage.create(f),PILImage.create(f2)
        return SiameseImage(img1, img2, t)
    
    def _draw(self, f):
        same = random.random() < 0.5
        cls = self.label_func(f)
        if not same: cls = random.choice(L(l for l in self.labels if l != cls)) 
        return random.choice(self.lbl2files[cls]),same
    
splits = RandomSplitter()(files)
tfm = SiameseTransform(files, label_func, splits)
tls = TfmdLists(files, tfm, splits=splits)
dls = tls.dataloaders(after_item=[Resize(224), ToTensor], 
    after_batch=[IntToFloatTensor, Normalize.from_stats(*imagenet_stats)])


# We will use a pretrained architecture and pass our two images through it. 
# Then we can concatenate the results and send them to a custom head that will return two predictions. 
# In terms of modules, this looks like this:
class SiameseModel(Module):
    def __init__(self, encoder, head):
        self.encoder,self.head = encoder,head
    
    def forward(self, x1, x2):
        ftrs = torch.cat([self.encoder(x1), self.encoder(x2)], dim=1)
        return self.head(ftrs)

# To create our encoder, we just need to take a pretrained model and cut it, as we explained before. 
# The function create_body does that for us; we just have to pass it the place where we want to cut. 
# As we saw earlier, per the dictionary of metadata for pretrained models, the cut value for a resnet is -2:
encoder = create_body(resnet34, cut=-2)

# Then we can create our head. A look at the encoder tells us the last layer has 512 features, so this head will need to receive 512*2. 
# Why 2? We have to multiply by 2 because we have two images. So we create the head as follows:
encoder[-1]
head = create_head(512*2, 2, ps=0.5)

# With our encoder and head, we can now build our model:
model = SiameseModel(encoder, head)

# Before using Learner, we have two more things to define. First, we must define the loss function we want to use. 
# It’s regular cross-entropy, but since our targets are Booleans, we need to convert them to integers or PyTorch will throw an error:
def loss_func(out, targ):
    return nn.CrossEntropyLoss()(out, targ.long())

# A splitter is a function that tells the fastai library how to split the model into parameter groups. 
# These are used behind the scenes to train only the head of a model when we do transfer learning.
# Here we want two parameter groups: one for the encoder and one for the head. 
# We can thus define the following splitter (params is just a function that returns all parameters of a given module):
def siamese_splitter(model):
    return [params(model.encoder), params(model.head)]

# Since we are not using a convenience function from fastai for transfer learning (like vision_learner), we have to call learn.freeze manually. 
# This will make sure only the last parameter group (in this case, the head) is trained:
learn = Learner(dls, model, loss_func=loss_func, 
                splitter=siamese_splitter, metrics=accuracy)
learn.freeze()

learn.fit_one_cycle(4, 3e-3)

# Before unfreezing and fine-tuning the whole model a bit more with discriminative learning rates (that is: a lower learning rate for the body and a higher one for the head):
learn.unfreeze()
learn.fit_one_cycle(4, slice(1e-6,1e-4))
```
