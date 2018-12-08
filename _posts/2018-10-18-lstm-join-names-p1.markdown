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

```python
import pandas as pd
import numpy as np
import random
```

## Loading and pre-processing data. 
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
def get_vectors(args):
    sequences,next_chars,chars = args
    char_indices = dict((c, i) for i, c in enumerate(chars))
    indices_char = dict((i, c) for i, c in enumerate(chars))
    X = np.zeros((len(sequences), SEQLEN, len(chars)), dtype=np.bool)
    y = np.zeros((len(sequences), len(chars)), dtype=np.bool)
    for i, sentence in enumerate(sequences):
        for t, char in enumerate(sentence):
            X[i, t, char_indices[char]] = 1
        y[i, char_indices[next_chars[i]]] = 1
    print ("Shape of X (sequences):",X.shape)
    print ("Shape of y (next_chars):",y.shape)
    return X,y,char_indices, indices_char

```

## Creating the LSTM Model
- We're creating a simple LSTM model that takes in a sequence of size `SEQLEN`, each element containing `len(chars)` elements
- The output of the LSTM goes into a Dense layer that predicts the next character with a softmaxed one-hot encoding

```python
from keras.models import Sequential
from keras.layers import Dense, Activation
from keras.layers import LSTM
from keras.optimizers import RMSprop

def get_model(num_chars):
    model = Sequential()
    model.add(LSTM(16, input_shape=(SEQLEN, num_chars)))
    model.add(Dense(len(chars)))
    model.add(Activation('softmax'))

    optimizer = RMSprop(lr=0.01)
    model.compile(loss='categorical_crossentropy', optimizer=optimizer)
    return model
```

## Sampling with our model
- Picking the element with the greatest probability will always return the same character for a given sequence
- I'd like to induce some variance by sampling from a probability array instead.

```python
def sample(preds, temperature=1.0):
    preds = np.asarray(preds).astype('float64')
    preds = np.log(preds) / temperature
    exp_preds = np.exp(preds)
    preds = exp_preds / np.sum(exp_preds)
    probas = np.random.multinomial(1, preds, 1)
    return np.argmax(probas)
```
## Training the LSTM on our dataset of Names

- Finally, we use the functions defined above to fetch data and train the model

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
        next_char = indices_char[sample(preds)]
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
