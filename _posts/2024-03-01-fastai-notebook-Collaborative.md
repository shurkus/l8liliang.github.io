---
layout: article
tags: AI
title: fastai notebook - Collaborative
mathjax: true
key: Linux
---

[soruce](https://docs.fast.ai/tutorial.vision.html)
{:.info} 


## Text transfer learning
```
from fastai.tabular.all import *
from fastai.collab import *

path = untar_data(URLs.ML_100k)

# The main table is in u.data. Since it’s not a proper csv, we have to specify a few things while opening it: the tab delimiter, the columns we want to keep and their names.
ratings = pd.read_csv(path/'u.data', delimiter='\t', header=None,
                      usecols=(0,1,2), names=['user','movie','rating'])
ratings.head()
	user	movie	rating
0	196	242	3
1	186	302	3
2	22	377	1
3	244	51	2
4	166	346	1


# Movie ids are not ideal to look at things, so we load the corresponding movie id to the title that is in the table u.item:
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
        user	movie	rating	title
0	196	242	3	Kolya (1996)
1	63	242	3	Kolya (1996)
2	226	242	5	Kolya (1996)
3	154	242	3	Kolya (1996)
4	306	242	5	Kolya (1996)


# We can then build a DataLoaders object from this table. By default, it takes the first column for user, 
the second column for the item (here our movies) and the third column for the ratings. We need to change the value of item_name in our case, to use the titles instead of the ids:
dls = CollabDataLoaders.from_df(ratings, item_name='title', bs=64)

dls.show_batch()
        user	title	rating
0	181	Substitute, The (1996)	1
1	189	Ulee's Gold (1997)	3
2	6	L.A. Confidential (1997)	4
3	849	Net, The (1995)	5
4	435	Blade Runner (1982)	4
5	718	My Best Friend's Wedding (1997)	4

# fastai can create and train a collaborative filtering model by using collab_learner:
learn = collab_learner(dls, n_factors=50, y_range=(0, 5.5))
learn.fit_one_cycle(5, 5e-3, wd=0.1)
```

### Interpretation
```
g = ratings.groupby('title')['rating'].count()
top_movies = g.sort_values(ascending=False).index.values[:1000]
top_movies[:10]
> array(['Star Wars (1977)', 'Contact (1997)', 'Fargo (1996)',
       'Return of the Jedi (1983)', 'Liar Liar (1997)',
       'English Patient, The (1996)', 'Scream (1996)', 'Toy Story (1995)',
       'Air Force One (1997)', 'Independence Day (ID4) (1996)'],
      dtype=object)
```

#### bias
```
# Our model has learned one bias per movie, a unique number independent of users that can be interpreted as the intrinsic “value” of the movie.
# We can grab the bias of each movie in our top_movies list with the following command:
movie_bias = learn.model.bias(top_movies, is_item=True)
movie_bias.shape

mean_ratings = ratings.groupby('title')['rating'].mean()
movie_ratings = [(b, i, mean_ratings.loc[i]) for i,b in zip(top_movies,movie_bias)]

item0 = lambda o:o[0]
sorted(movie_ratings, key=item0)[:15]

```

#### weight
```
movie_w = learn.model.weight(top_movies, is_item=True)
movie_w.shape

```
