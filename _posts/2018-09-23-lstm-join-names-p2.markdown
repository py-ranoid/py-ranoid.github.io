---
title: " Generating Couple Names (Portmanteau) with LSTMs : Part 2"
layout: post
date: 2018-10-19 08:16
tag: 
- ml
image: ../assets/images/link.png
headerImage: true
projects: false
hidden: false
description: "Using sequential deep learning models to join two names, ie, Brad + Angelina = Brangelina"
category: blog
author: Vishal Gupta
externalLink: false
---
# Part 2 : Joining Names using our LSTM
Now that we have an LSTM to generate names, we can use it to bridge two words. 

## Imports

```python
import pandas as pd
import numpy as np
import random
```

## Loading our model 

```python
from keras.models import model_from_json

json_file = open('model_keras.json', 'r')
loaded_model_json = json_file.read()
json_file.close()
model = model_from_json(loaded_model_json)
model.load_weights("model_keras.h5")
print("Loaded model from disk")
```

## Converting word to one-hot encoded vector
```python
def one_hot(word):
    x_pred = np.zeros((1, SEQLEN, len(chars)))

    for t, char in enumerate(word):
        x_pred[0, t, char_indices[char]] = 1.
    return x_pred
```

--- 
