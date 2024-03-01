---
layout: article
tags: AI
title: fastai notebook - vision
mathjax: true
key: Linux
---

[soruce](https://docs.fast.ai/tutorial.vision.html)
{:.info} 


## Single-label classification

### Cats vs Dogs
```
from fastai.vision.all import *

path = untar_data(URLs.PETS)
path.ls()
files = get_image_files(path/"images")
len(files)

def label_func(f): return f[0].isupper()

dls = ImageDataLoaders.from_name_func(path, files, label_func, item_tfms=Resize(224))

dls.show_batch()

learn = vision_learner(dls, resnet34, metrics=error_rate)
learn.fine_tune(1)

learn.predict(files[0])

learn.show_results()
```

### Classifying breeds
```
files[0].name
> 'great_pyrenees_173.jpg'

pat = r'^(.*)_\d+.jpg'
dls = ImageDataLoaders.from_name_re(path, files, pat, item_tfms=Resize(224))
dls.show_batch()

dls = ImageDataLoaders.from_name_re(path, files, pat, item_tfms=Resize(460),
                                    batch_tfms=aug_transforms(size=224))
dls.show_batch()
learn = vision_learner(dls, resnet34, metrics=error_rate)
learn.lr_find()
learn.fine_tune(2, 3e-3)
```

#### interp = Interpretation.from_learner(learn)
```
interp = Interpretation.from_learner(learn)
interp.plot_top_losses(9, figsize=(15,10))
```

### Single-label classification - With the data block API
```
pets = DataBlock(blocks=(ImageBlock, CategoryBlock), 
                 get_items=get_image_files, 
                 splitter=RandomSplitter(),
                 get_y=using_attr(RegexLabeller(r'(.+)_\d+.jpg$'), 'name'),
                 item_tfms=Resize(460),
                 batch_tfms=aug_transforms(size=224))
dls = pets.dataloaders(untar_data(URLs.PETS)/"images")
```

## Multi-label classification
```
path = untar_data(URLs.PASCAL_2007)
path.ls()
df = pd.read_csv(path/'train.csv')
df.head()
           fname	labels	        is_valid
0	000005.jpg	chair	        True
1	000007.jpg	car	        True
2	000009.jpg	horse person	True
3	000012.jpg	car	        False
4	000016.jpg	bicycle	        True
```

### Multi-label classification - Using the high-level API
```
dls = ImageDataLoaders.from_df(df, path, folder='train', valid_col='is_valid', label_delim=' ',
                               item_tfms=Resize(460), batch_tfms=aug_transforms(size=224))

error_rate will not work for a multi-label problem, but we can use accuracy_thresh and F1ScoreMulti. 
We can also change the default name for a metric, for instance, we may want to see F1 scores with macro and samples averaging.

f1_macro = F1ScoreMulti(thresh=0.5, average='macro')
f1_macro.name = 'F1(macro)'
f1_samples = F1ScoreMulti(thresh=0.5, average='samples')
f1_samples.name = 'F1(samples)'
learn = vision_learner(dls, resnet50, metrics=[partial(accuracy_multi, thresh=0.5), f1_macro, f1_samples])

learn.lr_find()
learn.fine_tune(2, 3e-2)
learn.predict(path/'train/000005.jpg')

interp = Interpretation.from_learner(learn)
interp.plot_top_losses(9)
```

### Multi-label classification - With the data block API
```
pascal = DataBlock(blocks=(ImageBlock, MultiCategoryBlock),
                   splitter=ColSplitter('is_valid'),
                   get_x=ColReader('fname', pref=str(path/'train') + os.path.sep),
                   get_y=ColReader('labels', label_delim=' '),
                   item_tfms = Resize(460),
                   batch_tfms=aug_transforms(size=224))
dls = pascal.dataloaders(df)
```

## Segmentation
```
path = untar_data(URLs.CAMVID_TINY)
path.ls()
> (#3) [Path('/home/jhoward/.fastai/data/camvid_tiny/codes.txt'),Path('/home/jhoward/.fastai/data/camvid_tiny/images'),Path('/home/jhoward/.fastai/data/camvid_tiny/labels')]

The images folder contains the images, and the corresponding segmentation masks of labels are in the labels folder. 
The codes file contains the corresponding integer to class (the masks have an int value for each pixel).

codes = np.loadtxt(path/'codes.txt', dtype=str)
codes
> array(['Animal', 'Archway', 'Bicyclist', 'Bridge', 'Building', 'Car',
       'CartLuggagePram', 'Child', 'Column_Pole', 'Fence', 'LaneMkgsDriv',
       'LaneMkgsNonDriv', 'Misc_Text', 'MotorcycleScooter', 'OtherMoving',
       'ParkingBlock', 'Pedestrian', 'Road', 'RoadShoulder', 'Sidewalk',
       'SignSymbol', 'Sky', 'SUVPickupTruck', 'TrafficCone',
       'TrafficLight', 'Train', 'Tree', 'Truck_Bus', 'Tunnel',
       'VegetationMisc', 'Void', 'Wall'], dtype='<U17')

```

### Segmentation - Using the high-level API
```
fnames = get_image_files(path/"images")

def label_func(fn): return path/"labels"/f"{fn.stem}_P{fn.suffix}"

dls = SegmentationDataLoaders.from_label_func(
    path, bs=8, fnames = fnames, label_func = label_func, codes = codes
)
learn = unet_learner(dls, resnet34)
learn.fine_tune(6)

interp = SegmentationInterpretation.from_learner(learn)
interp.plot_top_losses(k=3)
```

### Segmentation - With the data block API
```
camvid = DataBlock(blocks=(ImageBlock, MaskBlock(codes)),
                   get_items = get_image_files,
                   get_y = label_func,
                   splitter=RandomSplitter(),
                   batch_tfms=aug_transforms(size=(120,160)))

dls = camvid.dataloaders(path/"images", path=path, bs=8)
```

## Points
```
path = untar_data(URLs.BIWI_HEAD_POSE)
path.ls()
> (#50) [Path('/home/sgugger/.fastai/data/biwi_head_pose/01.obj'),Path('/home/sgugger/.fastai/data/biwi_head_pose/18.obj'),Path('/home/sgugger/.fastai/data/biwi_head_pose/04'),Path('/home/sgugger/.fastai/data/biwi_head_pose/10.obj'),Path('/home/sgugger/.fastai/data/biwi_head_pose/24'),Path('/home/sgugger/.fastai/data/biwi_head_pose/14.obj'),Path('/home/sgugger/.fastai/data/biwi_head_pose/20.obj'),Path('/home/sgugger/.fastai/data/biwi_head_pose/11.obj'),Path('/home/sgugger/.fastai/data/biwi_head_pose/02.obj'),Path('/home/sgugger/.fastai/data/biwi_head_pose/07')...]

(path/'01').ls()
> (#1000) [Path('01/frame_00087_pose.txt'),Path('01/frame_00079_pose.txt'),Path('01/frame_00114_pose.txt'),Path('01/frame_00084_rgb.jpg'),Path('01/frame_00433_pose.txt'),Path('01/frame_00323_rgb.jpg'),Path('01/frame_00428_rgb.jpg'),Path('01/frame_00373_pose.txt'),Path('01/frame_00188_rgb.jpg'),Path('01/frame_00354_rgb.jpg')...]

img_files = get_image_files(path)
def img2pose(x): return Path(f'{str(x)[:-7]}pose.txt')
img2pose(img_files[0])
> Path('04/frame_00084_pose.txt')

im = PILImage.create(img_files[0])
im.shape

im.to_thumb(160)

cal = np.genfromtxt(path/'01'/'rgb.cal', skip_footer=6)
def get_ctr(f):
    ctr = np.genfromtxt(img2pose(f), skip_header=3)
    c1 = ctr[0] * cal[0][0]/ctr[2] + cal[0][2]
    c2 = ctr[1] * cal[1][1]/ctr[2] + cal[1][2]
    return tensor([c1,c2])

get_ctr(img_files[0])

biwi = DataBlock(
    blocks=(ImageBlock, PointBlock),
    get_items=get_image_files,
    get_y=get_ctr,
    splitter=FuncSplitter(lambda o: o.parent.name=='13'),
    batch_tfms=[*aug_transforms(size=(240,320)), 
                Normalize.from_stats(*imagenet_stats)]
)

dls = biwi.dataloaders(path)
dls.show_batch(max_n=9, figsize=(8,6))
learn = vision_learner(dls, resnet18, y_range=(-1,1))
learn.lr_find()
learn.fine_tune(1, 5e-3)
math.sqrt(0.0001)
learn.show_results()
```
