---
title: " Generating Couple Names (Portmanteau) with LSTMs : Part 1"
layout: post
date: 2008-09-23 08:36
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
# Part 1 : Training a Name-Generating LSTM
First, we need to train an LSTM on a large dataset of names, so it can generate artificial names by predicting the nth character given (n-1) characters of a name.

## Imports 

Importing `pandas`, `numpy`, `random` and `sys`
```python
import pandas as pd
import numpy as np
import random
import sys
```

## Downloading the Dataset
Baby Names from Social Security Card Applications - National Level Data
```bash
!wget https://raw.githubusercontent.com/jcbain/celeb_baby_names/master/data/NationalNames.csv
```

## Loading and pre-processing data. 

Declaring `SEQLEN` and `STEP` :
`SEQLEN` is the number of characters our LSTM uses to predict the next character</br>
`STEP` number of letters to skip between two samples

Hence, the name `PETER` is used to generate the following samples :

| X1  |  X2  | X3  | Y  |
|--:|---|---|---|
| -  | - | P  | **E**  |
| -  | P | E  | **T**  |
| P | E | T  | **E**  |
| E  | T | E  | **R**  |
| T  | E | R  | **-**  |

```python
SEQLEN = 3
STEP = 1
```

- Loading names from `NationalNames.csv`
- Eliminating names shorter than 4 chars and having frequency less than 3
- Joining (seperating) names with `\n`

```python
def get_data():
    df = pd.read_csv('data/Names/NationalNames.csv')
    names = list(df[(df['Count'] > 3) & (df['Name'].str.len() > 4)]['Name'].unique())
    text = '\n' + '\n\n'.join(names).lower() + '\n'
    chars = sorted(list(set(text)))

    print ("Loaded",len(names),"names with",len(chars),"characters.")
    # Loaded 87659 names with 27 characters.
    return text,chars
```

- Splitting text into `sequences` of 3 characters (X) and adding next character to `next_chars` (y)

```python
def get_seq(args):
    text = args[0]
    sequences = []
    next_chars = []
    for i in range(0, len(text) - SEQLEN, STEP):
        sequences.append(text[i: i + SEQLEN])
        next_chars.append(text[i + SEQLEN])
    print('No. of sequences:', len(sequences))
    print('No. of chars:', len(next_chars))
    return sequences,next_chars,args[1]

```

- One-Hot Encoding characters in `sequences` and `next_chars`

```python
# This function encodes a given word into a numpy array by 1-hot encoding the characters
def one_hot(word,char_indices):
    x_pred = np.zeros((1, SEQLEN, 27))

    for t, char in enumerate(word):
        x_pred[0, t, char_indices[char]] = 1.
    return x_pred
  
# Encoding all sequences
def get_vectors(args):
    sequences,next_chars,chars = args
    char_indices = dict((c, i) for i, c in enumerate(chars))
    indices_char = dict((i, c) for i, c in enumerate(chars))
    X = np.zeros((len(sequences), SEQLEN, len(chars)), dtype=np.bool)
    y = np.zeros((len(sequences), len(chars)), dtype=np.bool)
    for i, sentence in enumerate(sequences):
        X[i] = one_hot(sentence,char_indices)
        y[i, char_indices[next_chars[i]]] = 1
    print ("Shape of X (sequences):",X.shape)
    print ("Shape of y (next_chars):",y.shape)
    # Shape of X (sequences): (764939, 3, 27)
    # Shape of y (next_chars): (764939, 27)
    return X,y,char_indices, indices_char
```

## Creating the LSTM Model
- We're creating a simple LSTM model that takes in a sequence of size `SEQLEN`, each element of `len(chars)` numbers (1 or 0)
- The output of the LSTM goes into a Dense layer that predicts the next character with a softmaxed one-hot encoding

```python
from keras.models import Sequential
from keras.layers import Dense, Activation
from keras.layers import LSTM
from keras.optimizers import RMSprop

def get_model(num_chars):
    model = Sequential()
    model.add(LSTM(16, input_shape=(SEQLEN, num_chars)))
    model.add(Dense(num_chars))
    model.add(Activation('softmax'))

    optimizer = RMSprop(lr=0.01)
    model.compile(loss='categorical_crossentropy', optimizer=optimizer)
    return model
```

## Sampling with our model
- Picking the element with the greatest probability will always return the same character for a given sequence
- I'd like to induce some variance by sampling from a probability array instead.

To explain this better, here's an excerpt  from Andrej Karpathy's blog aobut CharRNNs : 
> Temperature. We can also play with the temperature of the Softmax during sampling. Decreasing the **temperature from 1 to some lower number (e.g. 0.5) makes the RNN more confident, but also more conservative** in its samples. Conversely, **higher temperatures will give more diversity but at cost of more mistakes** (e.g. spelling mistakes, etc). In particular, setting temperature very near zero will give the most likely thing that Paul Graham might say:

> *“is that they were all the same thing that was a startup is that they were all the same thing that was a startup is that they were all the same thing that was a startup is that they were all the same”*


```python
def sample(preds, temperature=1.0):
    # helper function to sample an index from a probability array
    preds = np.asarray(preds).astype('float64')
    preds = np.log(preds) / temperature
    exp_preds = np.exp(preds)
    preds = exp_preds / np.sum(exp_preds)
    probas = np.random.multinomial(1, preds, 1)
    return np.argmax(probas)
```
## Training the LSTM on our dataset of Names

- Finally, we use the functions defined above to fetch data and train the model
- I trained for 30 epochs since the loss almost seemed to stagnate after that
- This depends on the dataset that's being used and complexity of the sequences.

```python
X,y,char_indices, indices_char = get_vectors(get_seq(get_data()))
model = get_model(len(char_indices.keys()))
model.fit(X, y,
          batch_size=128,
          epochs=30)

# Saving the model
model_json = model.to_json()
with open("model_keras.json", "w") as json_file:
    json_file.write(model_json)
model.save_weights("model_keras.h5")
```

## Testing our model

```python
def gen_name(seed):
    generated = seed
    for i in range(10):
        x_pred = np.zeros((1, SEQLEN, 27))
        for t, char in enumerate(seed):
            x_pred[0, t, char_indices[char]] = 1.
        preds = model.predict(x_pred, verbose=0)[0]
        next_char = indices_char[sample(preds,0.5)]
        if next_char == '\n':break
        generated += next_char
        seed = seed[1:] + next_char

    return generated

```
- Generating names from 3-letter seeds

```python
for i in ['mar','ram','seb']:
    print ('Seed: "'+i+'"\tNames :',[gen_name(i) for _ in range(5)])
```

    Seed: "mar"	Names : ['marisa', 'maria', 'marthi', 'marvamarra', 'maria']
    Seed: "ram"	Names : ['ramir', 'ramundro', 'ramariis', 'raminyodonami', 'ramariegena']
    Seed: "seb"	Names : ['sebeexenn', 'sebrinx', 'seby', 'sebrey', 'seberle']  

---

---
