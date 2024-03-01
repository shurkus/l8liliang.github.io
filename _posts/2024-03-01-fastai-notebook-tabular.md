---
layout: article
tags: AI
title: fastai notebook - tabular
mathjax: true
key: Linux
---

[soruce](https://docs.fast.ai/tutorial.vision.html)
{:.info} 


## Tabular training
```
from fastai.tabular.all import *
path = untar_data(URLs.ADULT_SAMPLE)
path.ls()

df = pd.read_csv(path/'adult.csv')
df.head()

dls = TabularDataLoaders.from_csv(path/'adult.csv', path=path, y_names="salary",
    cat_names = ['workclass', 'education', 'marital-status', 'occupation', 'relationship', 'race'],
    cont_names = ['age', 'fnlwgt', 'education-num'],
    procs = [Categorify, FillMissing, Normalize])


# use TabularPandas
splits = RandomSplitter(valid_pct=0.2)(range_of(df))

to = TabularPandas(df, procs=[Categorify, FillMissing,Normalize],
                   cat_names = ['workclass', 'education', 'marital-status', 'occupation', 'relationship', 'race'],
                   cont_names = ['age', 'fnlwgt', 'education-num'],
                   y_names='salary',
                   splits=splits)
to.xs.iloc[:2]
       marital-status	occupation	relationship	race	education-num_na	age	fnlwgt	education-num
15780	2	16	1	5	2	5	1	0.984037	2.210372	-0.033692
17442	5	12	5	8	2	5	1	-1.509555	-0.319624	-0.425324

dls = to.dataloaders(bs=64)
dls.show_batch()

# We can define a model using the tabular_learner method. When we define our model, fastai will try to infer the loss function based on our y_names earlier.
# Sometimes with tabular data, your y’s may be encoded (such as 0 and 1). In such a case you should explicitly pass 
# y_block = CategoryBlock in your constructor so fastai won’t presume you are doing regression.
learn = tabular_learner(dls, metrics=accuracy)

learn.fit_one_cycle(1)
learn.show_results()

row, clas, probs = learn.predict(df.iloc[0])
row.show()
   marital-status	occupation	relationship	race	education-num_na	age	fnlwgt	education-num	salary
0	Private	Assoc-acdm	Married-civ-spouse	#na#	Wife	White	False	49.0	101319.99788	12.0	>=50k

# To get prediction on a new dataframe, you can use the test_dl method of the DataLoaders. That dataframe does not need to have the dependent variable in its column.
test_df = df.copy()
test_df.drop(['salary'], axis=1, inplace=True)
dl = learn.dls.test_dl(test_df)

learn.get_preds(dl=dl)

```

### other 
```
to.xs[:3]

X_train, y_train = to.train.xs, to.train.ys.values.ravel()
X_test, y_test = to.valid.xs, to.valid.ys.values.ravel()
```

## chapter 9

### process data
```
The goal of the contest is to predict the sale price of a particular piece of heavy equipment at auction based on its usage, equipment type, and configuration. 
The data is sourced from auction result postings and includes information on usage and equipment configurations.

from fastbook import *
from pandas.api.types import is_string_dtype, is_numeric_dtype, is_categorical_dtype
from fastai.tabular.all import *
from sklearn.ensemble import RandomForestRegressor
from sklearn.tree import DecisionTreeRegressor
from dtreeviz.trees import *
from IPython.display import Image, display_svg, SVG

pd.options.display.max_rows = 20
pd.options.display.max_columns = 8

# The easiest way to download Kaggle datasets is to use the Kaggle API. You can install this using `pip` by running this in a notebook cell:
!pip install kaggle
creds = '{"username":"liliang666","key":"5b9e0248109348c312e44886c4066405"}'

cred_path = Path('~/.kaggle/kaggle.json').expanduser()
if not cred_path.exists():
    cred_path.parent.mkdir(exist_ok=True)
    cred_path.write_text(creds)
    cred_path.chmod(0o600)

comp = 'bluebook-for-bulldozers'
path = URLs.path(comp)
path

Path.BASE_PATH = path

from kaggle import api

if not path.exists():
    path.mkdir(parents=true)
    api.competition_download_cli(comp, path=path)
    shutil.unpack_archive(str(path/f'{comp}.zip'), str(path))

Kaggle provides information about some of the fields of our dataset. The Data explains that the key fields in train.csv are:

SalesID:: The unique identifier of the sale.
MachineID:: The unique identifier of a machine. A machine can be sold multiple times.
saleprice:: What the machine sold for at auction (only provided in train.csv).
saledate:: The date of the sale.

df = pd.read_csv(path/'TrainAndValid.csv', low_memory=False)
df.columns

df['ProductSize'].unique()
array([nan, 'Medium', 'Small', 'Large / Medium', 'Mini', 'Large', 'Compact'], dtype=object)


# We can tell Pandas about a suitable ordering of these levels like so:
sizes = 'Large','Large / Medium','Medium','Small','Mini','Compact'
df['ProductSize'] = df['ProductSize'].astype('category')
df['ProductSize'].cat.set_categories(sizes, ordered=True)

dep_var = 'SalePrice'
df[dep_var] = np.log(df[dep_var])

# 处理时间
df = add_datepart(df, 'saledate')
df_test = pd.read_csv(path/'Test.csv', low_memory=False)
df_test = add_datepart(df_test, 'saledate')
' '.join(o for o in df.columns if o.startswith('sale'))
'saleYear saleMonth saleWeek saleDay saleDayofweek saleDayofyear saleIs_month_end saleIs_month_start saleIs_quarter_end saleIs_quarter_start saleIs_year_end saleIs_year_start saleElapsed'

# A second piece of preparatory processing is to be sure we can handle strings and missing data.
procs = [Categorify, FillMissing]
# TabularPandas will also handle splitting the dataset into training and validation sets for us. 
# However we need to be very careful about our validation set. We want to design it so that it is like the test set Kaggle will use to judge the contest.
cond = (df.saleYear<2011) | (df.saleMonth<10)
train_idx = np.where( cond)[0]
valid_idx = np.where(~cond)[0]
splits = (list(train_idx),list(valid_idx))

# TabularPandas needs to be told which columns are continuous and which are categorical. We can handle that automatically using the helper function cont_cat_split:
cont,cat = cont_cat_split(df, 1, dep_var=dep_var)
to = TabularPandas(df, procs, cat, cont, y_names=dep_var, splits=splits)
len(to.train),len(to.valid)
to.show(3)

to1 = TabularPandas(df, procs, ['state', 'ProductGroup', 'Drive_System', 'Enclosure'], [], y_names=dep_var, splits=splits)
to1.show(3)

# However, the underlying items are all numeric:
to.items.head(3)
to1.items[['state', 'ProductGroup', 'Drive_System', 'Enclosure']].head(3)

# Since it takes a minute or so to process the data to get to this point, 
# we should save it—that way in the future we can continue our work from here without rerunning the previous steps. 
# fastai provides a save method that uses Python’s pickle system to save nearly any Python object:
save_pickle(path/'to.pkl',to)

```

### Creating the Decision Tree
```
to = (path/'to.pkl').load()
xs,y = to.train.xs,to.train.y
valid_xs,valid_y = to.valid.xs,to.valid.y
m = DecisionTreeRegressor(max_leaf_nodes=4)
m.fit(xs, y);

# 把太老的时间改成1950
xs.loc[xs['YearMade']<1900, 'YearMade'] = 1950
valid_xs.loc[valid_xs['YearMade']<1900, 'YearMade'] = 1950

# We’ll create a little function to check the root mean squared error of our model (m_rmse), since that’s how the competition was judged:
m = DecisionTreeRegressor()
m.fit(xs, y);
def r_mse(pred,y): return round(math.sqrt(((pred-y)**2).mean()), 6)
def m_rmse(m, xs, y): return r_mse(m.predict(xs), y)
m_rmse(m, valid_xs, valid_y)

# result is bad, maybe overfitting
# check leaf count
m.get_n_leaves(), len(xs)

# Let’s change the stopping rule to tell sklearn to ensure every leaf node contains at least 25 auction records:
m = DecisionTreeRegressor(min_samples_leaf=25)
m.fit(to.train.xs, to.train.y)
m_rmse(m, xs, y), m_rmse(m, valid_xs, valid_y)


```

### Random Forest
```
In the following function definition n_estimators defines the number of trees we want, max_samples defines how many rows to sample for training each tree, 
and max_features defines how many columns to sample at each split point (where 0.5 means “take half the total number of columns”). 
We can also specify when to stop splitting the tree nodes, effectively limiting the depth of the tree, by including the same min_samples_leaf parameter we used in the last section. 
Finally, we pass n_jobs=-1 to tell sklearn to use all our CPUs to build the trees in parallel. 
By creating a little function for this, we can more quickly try different variations in the rest of this chapter:

def rf(xs, y, n_estimators=40, max_samples=200_000,
       max_features=0.5, min_samples_leaf=5, **kwargs):
    return RandomForestRegressor(n_jobs=-1, n_estimators=n_estimators,
        max_samples=max_samples, max_features=max_features,
        min_samples_leaf=min_samples_leaf, oob_score=True).fit(xs, y)

m = rf(xs, y);
m_rmse(m, xs, y), m_rmse(m, valid_xs, valid_y)

```

### Feature Importance
```
def rf_feat_importance(m, df):
    return pd.DataFrame({'cols':df.columns, 'imp':m.feature_importances_}
                       ).sort_values('imp', ascending=False)
fi = rf_feat_importance(m, xs)
fi[:10]

        cols	        imp
59	YearMade	0.180070
7	ProductSize	0.113915
31	Coupler_System	0.104699
8	fiProductClassDesc	0.064118
33	Hydraulics_Flow	0.059110
56	ModelID	0.059087
51	saleElapsed	0.051231
4	fiSecondaryDesc	0.041778
32	Grouser_Tracks	0.037560
2	fiModelDesc	0.030933

def plot_fi(fi):
    return fi.plot('cols', 'imp', 'barh', figsize=(12,7), legend=False)

plot_fi(fi[:30]);

```

### Neural Network
```
df_nn = pd.read_csv(path/'TrainAndValid.csv', low_memory=False)
df_nn['ProductSize'] = df_nn['ProductSize'].astype('category')
df_nn['ProductSize'].cat.set_categories(sizes, ordered=True, inplace=True)
df_nn[dep_var] = np.log(df_nn[dep_var])
df_nn = add_datepart(df_nn, 'saledate')

df_nn_final = df_nn[list(xs_final_time.columns) + [dep_var]]

cont_nn,cat_nn = cont_cat_split(df_nn_final, max_card=9000, dep_var=dep_var)

cat_nn.remove('fiModelDescriptor')

procs_nn = [Categorify, FillMissing, Normalize]
to_nn = TabularPandas(df_nn_final, procs_nn, cat_nn, cont_nn,
                      splits=splits, y_names=dep_var)

dls = to_nn.dataloaders(1024)

learn = tabular_learner(dls, y_range=(8,12), layers=[500,250],
                        n_out=1, loss_func=F.mse_loss)
learn.lr_find()
learn.fit_one_cycle(5, 1e-2)
preds,targs = learn.get_preds()
r_mse(preds,targs)

learn.save('nn')
```
