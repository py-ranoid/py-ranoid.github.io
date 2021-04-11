---
title: "Using LSTMs to join words (Portmanteaus): Part 1"
layout: post
date: 2018-12-08 18:55
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
Often, I find myself making [portmateaus](https://dictionary.cambridge.org/dictionary/english/portmanteau-word) from verbs, names, adjectives and pretty much any word I think too much about. Sometimes to shrink phrases, and sometimes to name a product or app; occasionally, to ship couples. And as someone who loves tinkering with AI, I wondered if it was possible to write an algorithm to do it… and here we are. The first part, this blog is about training a model that can generate artificial names with a character-level LSTM.

If you’re new to LSTMs, RNNs, or sequential models, here are a few resources that can help you learn and get started:
[bit.ly/SeqModelsResources](http://bit.ly/SeqModelsResources).

# Part 1 : Training a Name-Generating LSTM

<div class="side-by-side">
    <div class="toleft">
        <img src = "http://karpathy.github.io/assets/rnn/charseq.jpeg">
    </div>

    <div class="toright">
        <ul>
        <li> First, we need to train an LSTM on a large dataset of names, so it can generate artificial names by predicting the nth character given (n-1) characters of a name. </li>
        <li> In the image on the left, we have a character-level RNN with accepts and predicts 1 of 4 characters ('h','e','l' and 'o').  </li>
        <li> Hence it has 4-dimensional input and output layers, and a hidden layer of 3 units (neurons). </li>
        <li> The output layer contains confidences the RNN assigns for the next character (vocabulary is "h,e,l,o") </li>
        <li> We want the green numbers to be high and red numbers to be low (in the output layer). </li>
        </ul>
        <br>
        Image Credits : <a href = "http://karpathy.github.io/2015/05/21/rnn-effectiveness/">Andrej Karpathy</a>
    </div>
</div>

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

```python
SEQLEN = 3 # No. of chars our LSTM uses to predict the next character
STEP = 1 # No. of letters to skip between two samples
```
Hence, the name `PETER` is used to generate the following samples :

| X1  |  X2  | X3  | Y  |
|---:|---:|---:|---:|
| -  | - | P  |  **E**  |
| -  | P | E  |  **T**  |
| P  | E | T  |  **E**  |
| E  | T | E  |  **R**  |
| T  | E | R  |  **-**  |

We need to do this for all names in our dataset.
- Load names from `NationalNames.csv`
- Eliminate names shorter than 4 chars and having frequency less than 3
- Join (seperating) names with `\n`

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

- Split text into `sequences` of 3 characters (X) and adding next character to `next_chars` (y)

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


<img src = "../assets/images/keras_lstm.png">

## Sampling with our model
- Picking the element with the greatest probability will always return the same character for a given sequence
- I'd like to induce some variance by sampling from a probability array instead.

To explain this better, here's an excerpt  from Andrej Karpathy's blog about CharRNNs :
> Temperature. We can also play with the temperature of the Softmax during sampling. Decreasing the **temperature from 1 to some lower number (e.g. 0.5) makes the RNN more confident, but also more conservative** in its samples. Conversely, **higher temperatures will give more diversity but at cost of more mistakes** (e.g. spelling mistakes, etc).

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

Great! We have a model that can generate fake names. Now all you need is a fake address and an empty passport. *Jk*.


In the [next blog](http://vishalgupta.me/lstm-join-names-p2), I'll explain how you can use this model to join two words by finding the best bridge.


---
