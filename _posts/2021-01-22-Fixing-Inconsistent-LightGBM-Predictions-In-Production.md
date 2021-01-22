---
toc: true
badges: false
layout: post
description: How hard can it be - just use pickle! Here's what I learned deploying an LightGBM model to production in order to ensure consistent predictions.
categories: [machine-learning]
keywords: lightgbm, pickle, lgb, booster, save_model, save lightgbm model, inconsistent predictions in production, machine learning, regression, joblib, dump, load, sklearn, sci-kit, bst
title: How to Ensure Consistent LightGBM Predictions in Production
image: images/pkl-meme.jpg
---

Recently I've been working on productionizing a regression model to accurately estimate used car values. This was following a request from a friend who owns a car dealership and wants to know two things: 1) how much a customer's car will be worth in 3-4 years when they trade it in for a new model, and 2) what's the trade-in value now of a new customer's used car. I was looking for a new project so I decided to help him out.

The model uses a LightGBM booster with ~6-10k estimators (depending on the number of features used). It's been quite the adventure, and I will write a blog post on the end-to-end process sometime in the future. In short, the process consists of:

1. **Scraping data** with Scrapy on multiple car sites via an Amazon EC2 instance and merging the data with other proprietary data.
1. **Aggregating and cleaning** the data and outputting a single, uniform dataset for model training. This is by far the most delicate and important step!
1. **Training the model**, benchmarking it against other methods including RF, XGBoost and Deep Learning using embeddings.
1. **Deploying the model to production**, making it accessible via an API endpoint.
1. **Creating a web app** that acts as a pretty customer frontend that queries the API for predictions.

This is all well and good, but having never worked with LightGBM before, some problems arose around Step 4, when I first started exporting the model and trying to replicate predictions in the production environment.

![]({{ site.baseurl }}/images/pkl-meme.jpg)

## Exporting a LightGBM Model

Now right off the bat, let's just say that LightGBM is awesome-- it's an efficient gradient boosting framework that uses tree-based learning. It's very efficient, uses lower memory than other tree/boosting methods and supports dealing with categorical label-encoded variables. However, I had a couple frustrations when porting my model from my prototyping environment to production.

Let's first recap the various ways in which you can export a model. We'll assume we have a standard booster that we need to save. Specific parameters are beyond the scope of this post.

```python
import lightgbm as lgb
# Our training model...
bst = lgb.LGBMRegressor().fit(X, y)
```

### Serializing with `pickle` or `joblib`

Two common ways to export any Sci-Kit model is [`pickle`](https://docs.python.org/3/library/pickle.html) or [`joblib`](https://joblib.readthedocs.io/en/latest/index.html#module-joblib). They are quite similar, as `joblib` uses `pickle` as a protocol under the hood. Pickling is essentially a process of serializing a python object structure by converting the underlying object hierarchy into a byte stream. What's not so great about pickling is that the resulting bytestream is hard to inspect unless unpickled (or generated using the oldest Protocol, v0). It also represents a potential security risk as a pickle could contain malicious code, and an untrusted pickle file opened without precautions could lead to naughty code being arbitrarily executed. `joblib` extends `pickle` by supporting compression helping serialize objects a bit more efficiently.

```python
# Using pickle 
import pickle
pickle.dump(bst, open('model.pkl', 'wb'))

# Using joblib
import joblib
joblib.dump(bst, 'model.pkl')
```

To de-serialize (import) your model later, you would use:

```python
# Using pickle 
import pickle
bst = pickle.load(open('model.pkl', 'rb'))

# Using joblib
import joblib
bst = joblib.load('model.pkl')
```

Once imported, it is _theoretically_ the same as the original model. This is not always the case (read on!).

### Exporting using LightGBM's `save_model`

LightGBM also offers its own export functionality which can be called directly from the booster itself. The method is called `.save_model`. The method outputs a clear, human-readable text file.

```python
# Saving the model using LightGBM's save_model method
bst.booster_.save_model('model.txt')
```

To de-serialize (import) your model later, you would use:

```python
# Importing the model using LightGBM's save_model method
bst = lgb.Booster(model_file='model.txt')
```

Again, once imported, it is _theoretically_ the same as the original model. However there's some important considerations that I found out the hard way.

## Inconsistent Predictions in Production

So you're having a blast prototyping away on your Jupyter Notebook, getting unbelievable predictive accuracy, eye-wateringly low validation loss. Your team, manager and CEO are popping the champagne and patting you on the back saying "you did it, you rockstar!".

Feeling invincible, you export the model and deploy it to production, start feeding it live data and your initial glee quickly turns to gut-wrenching dread as the predictions are all over the place. What went wrong?!

The first immediate hypothesis is that there's something wrong with the file. You try exporting the model again, but the predictions are different, and just as wild. You start to question your sanity. You start sweating, a cold panic sweat. Your brain is taunting you: "_All of that champage...for what?_"

Then you remember you were reading this blog post, and continue on reading.

### Common Reasons for Inconsistent LightGBM Predictions in Production

#### Environment Consistency

Goes without saying, that first and foremost you should ensure **environment consistency**. Make sure that your Python environment is identical to the one that you used in your model creation step. This means Python version, dependencies' versions, pip requirements, etc.

If your production environment needs to maintain a different dependency set for some reason, an alternative would be to **re-train your model in your production environment** so you're sure the same exact packages generating and saving your model are the same ones opening your model and making predictions.

Another good way to ensure consistency is to use `virtualenv`, generate a `pip` requirements list using `pip freeze > requirements.txt`. In production, you would copy over the requirements file and install your dependencies by using `pip install -r requirements.txt`. Done!

#### Feature (Column) Ordering

The second reason is **feature ordering**. In my experience, after having ruled out the potential issues above, the real reason for wildly wrong results was column ordering! Simple as that. It was difficult to debug when using pickled files as the byte stream is not human readable. However, if you save your model using the text-based `Booster.save_model` format, you can inspect the `model.txt`.

Here's an excerpt:

```txt
tree
version=v3
num_class=1
num_tree_per_iteration=1
label_index=0
max_feature_idx=12
objective=regression
feature_names=year color_cat km material_interiors_cat cv model_cat years_old transmission_cat fuel_cat seats cc doors brand_cat
feature_infos=[2006:2021] -1:3:8:11:1:5:14:13:4:15:2:10:6:7:12:0 [500:2920000] -1:0:2:6:3:1:4 [40:796] -1:71:410:281:200:138:318:302:187:199:57:408 ...
tree_sizes=3607 3727 3577 ...

... the file continues for 200,000 more lines
```

In the 8th line you see there's a `feature_names` attribute where the columns (aka features) are listed. **This ordering is very important! Make sure that any DataFrame or NumPy Array used for predictions follows this same ordering.**

#### How to Ensure Consistent Column Ordering

A manual way to ensure consistent column ordering this is to extract the feature names from the `model.txt` file at the line called `feature_names` and splitting them based on the space character, for example: `cols = 'feature1 feature2 feature3'.split(' ')`. This is what you would do at 3AM when you are debugging with tired mind (in my defense, it was quite late). Don't do this.

The better way is to read the feature names directly using LightGBM's built-in `Booster.feature_name` method.  You can then **reindex** your production DataFrame / or array based on the order of those columns. In Pandas you would do this in the following way:

```python
# In our production environment...
import lightgbm as lgb

# Load the booster
bst = lgb.Booster(model_file='model.txt')

# Get the model's features in the correct order
cols = bst.feature_name()     # -> ['feat1', 'feat2', 'feat3', ...]

# Use col to reindex the prediction DataFrame
df = df.reindex(columns=cols) # -> df now has the same col ordering as the model

# Get predictions that are consistent! 
predictions = bst.predict(df)
```

## Conclusions

I'll be sure to update this post as I uncover other quirks with productionizing this LightGBM model. So far I'm appreciating the compact size (compared to a Random Forest for example) and quick prediction speed!

That's all for now. Good luck with your model deployment and hope you find this useful!
