---
toc: false
layout: post
description: What are they and how do they work together? In this post we will go over what DataBlocks and DataLoader(s) are at a high level, and explain how they work together in the context of building a Machine Learning model. 
categories: [machine-learning, fastai]
keywords: machine learning, ml, fast.ai, fastai, dataloaders, datablocks, pytorch, data block, data loader
title: Understanding DataBlocks and DataLoaders in fast.ai
image: images/db-dl-fastai.png
---

## Introduction

Coming from SciKit and TensorFlow, when I first started working with PyTorch and fast.ai I quickly realized they have a very opinionated (but convenient!) way to deal with data, through the `DataBlock`s and `DataLoaders` APIs. In this post we will quickly go over what they are (you can check the [official documentation](https://docs.fast.ai/data.load.html) if you want to dive a little deeper), and understand how they work together. If you are not 100% clear about the difference between a Datablock and a DataLoader, this blog post hopefully will shed some light.

## DataBlock

![]({{ site.baseurl }}/images/datablock-structure.png "DataBlock overview")
A Data block is nothing more than a pipeline for data assembly. When you initially create a `DataBlock`, you won't need to specify any data. What you will need to specify, however, is a set of rules for **how** to treat your data when it does flow in. It doesn't care about what you'll do with it, it just cares about how you want it gathered, classified and split.

In order to create a Data block you need to specify
1. what types of data to expect for your input (aka features) and target variables (aka labels)
1. how to get the data
1. how to differentiate features from the target variables, 
1. how to split the data for training (train & validation set)

Let's see how to do that. Below is an example DataBlock that we would create if we were looking to create a Convolutional Neural Network (CNN) to recognize different species of cats (i.e. a classification model).

```python
cats = DataBlock(
    blocks=(ImageBlock, CategoryBlock), 
    get_items=get_image_files, 
    splitter=RandomSplitter(valid_pct=0.2, seed=42),
    get_y=parent_label,
    item_tfms=Resize(128))
```

The four main steps mentioned above are exactly the first four (required) arguments of a DataBlock:
1. `blocks`: is where you define the types of data your model will work with. Usually you will specify at least two blocks: one that represents your independent (input) variable, and one that represents your dependent (target) variable. You can also specify multiple input/output variables.
1. `get_items`: a function that will actually go and pick up the data when necessary (more on this later)
1. `splitter`: how to split up the data in a training and validation set. The seed is optional and only added for replicability
1. `get_y`: how to extract the target (dependent) variable from the data. In the case of our cat classifier, this will be by looking at the parent folder, and fast.ai provides a built in function called `parent_label.`.
1. `item_tfms` is an optional argument that we can include to specify any additional processing that needs to be carried out when we flow our data through. In this case, we will resize all images to 128x128. We can specify other transforms, such as `item_tfms=Resize(128, ResizeMethod.Squish))` which will resize and squish our images to fit, or `item_tfms=Resize(128, ResizeMethod.Pad, pad_mode='zeros')` to resize and pad any leftover space with black. This method is incredibly powerful as it also supports data augmentation. This is beyond the scope of this blog post, but just know that `item_tfms` allows you to pre-process your data before it hits your model.

In that snippet of code, what we've done is specify a template through which to create `DataLoaders`. What are they you might ask? Well, read on!

>Tip: For more information on DataBlocks, I highly recommend Aman's blog post covering the [DataBlocks API](https://medium.com/@amaarora/fastai-v2-datablocks-api-code-overview-a-gentle-introduction-60338a6c9aa). This is also where I got the image for the DataBlocks overview.

## DataLoader and DataLoaders

Now that we've defined a `DataBlock`, and we've specified exactly how our data needs to be structured, categorized and processed, we can start actually feeding in the data for our model to train on. We _load_ this data in with...you guessed it... a Data loader. This is where `DataLoaders` come in. A `DataLoaders` is an iterator class that our DataBlock will call to load data according to the rules that we've specified in specific chunks (called batch size).

A `DataLoader` in fast.ai is a superset of the PyTorch DataLoader, with more helpful callbacks and flexibility. Whereas the Data block knows how to structure the data, the Loader knows how to work with it in the context of training a machine learning model -- i.e. how much to feed to the model at once (batch size), how many processes to spawn to load the data, how much memory to allocate and many more.

A `DataLoaders` (note the plural), is a thin class that automatically generates multiple `DataLoader` (singular) objects based on the rules specified in our `DataBlock`.

## Conclusion and TLDR

We've seen what they are, now let's re-iterate how they work together and what their differences are:
- A **DataBlock** is the data pipeline. A template that we create that has NO data, but has all the context on how to work with it. For example, how to split up the data, the data types of our features and targets/labels, how to extract the labels from the underlying data (e.g. folder name).
- A **DataLoader** doesn't care about preparing data, it expects the data ready to go and only cares about how to load the data (e.g. whether in parallel or in a single process) as well as feeding the data to the model in batches (i.e. batch size)
- A **DataLoaders** is a thin wrapper for more than one `DataLoader`.

In the context of training a model, you will first create a `DataBlock`, specify all data processing pipelines, and then load your data through your `DataBlock` via the `.dataloaders` property, like so:

```python
path = Path('cats') #e.g. a dir structure such as 'cats/sphynx/001.jpg'
dls = cats.dataloaders(path) 

# our data is ready to be fed into a model
learn = cnn_leaner(dls, resent18, metrics=error_rate)
learn.fine_tune(5)
```

Hope this post helps clarify what they are and how these two structures work together. As my knowledge of fast.ai grows, I'll be sure to update this post 
