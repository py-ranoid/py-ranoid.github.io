---
title: "Using LSTMs to join words (Portmanteaus): Part 2"
layout: post
date: 2018-12-08 23:22
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
Now that we have an LSTM to generate names, we can use it to bridge two words. Since the model can predict likeliness of 27 characters succeeding a given sequence of letters, we can use it find the bridge between two words. We define a bridge as : 
- Let `m` = Length of left word, `L`
- Let `n` = Length of right word, `R`
- Then `Bridge(L,R) = (i,j)`, `i <= m` & `j <= n` 
- Where `i` is the end index of the left word in the portmanteau and `j` is the start index of the right word in the portmanteau.
- Hence... `Join(L,R) = L[:i] + R[j:]`
- For all combinations of `(i,j)` compute a score by summing the probabilities obtained from the char LSTM.
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

Alternatively, you can also download the weight files 
```bash
wget https://github.com/py-ranoid/WhatDoWeCallIt/raw/master/model_keras.h5 -nv
wget wget https://github.com/py-ranoid/WhatDoWeCallIt/raw/master/model_keras.json -nv
```    

## Helper functions
- `sample_preds` : Probabilities of 27 characters (A-Z + \n) to follow given sequence
- `ohmygauss`:Function to return decreasing gaussian sequence (right half of bell curve)

```python
from scipy.stats import norm
def sample_preds(preds, temperature=1.0):
    preds = np.asarray(preds).astype('float64')
    preds = np.log(preds) / temperature
    exp_preds = np.exp(preds)
    preds = exp_preds / np.sum(exp_preds)
    return preds


def ohmygauss(length, sigma=1.8):
    rv = norm(loc=0, scale=sigma)
    x = np.arange(length)
    return rv.pdf(x)

```

## Getting bridge scores
1. Iterate over all sequences of 3 (`SEQLEN`) characters in left word. (`MINLEFT -> n`)
    1. Iterate over all sequences of 3 (`COMPARE`) characters in right word. (`0 -> MINRIGHT`)
        1. Get probability that given character in right word will follow sequence from word
        2. Repeat for `COMPARE` sequences.</br>
        For example : to bridge **britain** and **exit** at `_br+exit`, <br>
         Score : `prob(e|"_br")*w1 + prob(x|"bre")*w2 + prob(i|"rex")*w3`
        3. Multiply Gaussian factors to score to prioritize words that are bridges towards the beggining of the right word

```python
MINLEFT = 3
MINRIGHT = 3
COMPARE = 3
LEFT_BIAS = [0.06, 0.05, 0.04]

def proc(left, right, verbose=False):
    best_matches = {}
    best_i = None
    best_i_score = -1
    for i in range(0, len(left) - MINLEFT + 1):
        # Searching all sequences of size COMPARE in the right word
        # to find best match
        best_j = None
        best_j_score = -1
        best_matches[i] = {}
        right_bound = len(right) - MINRIGHT + 1
        gaus_factors = ohmygauss(right_bound)
        for j in range(0, right_bound):
            right_chars = right[j:j + COMPARE]
            s = 0
            for x in range(COMPARE):
                
                # Character on right which is being sampled
                c_index = char_indices[right_chars[x]]
                if verbose:
                    print ("Sampling " + left[i + x:i + SEQLEN] +
                           right[j:j + x] + "-->" + right_chars[x])

                # Generating sequence and getting probability
                Xoh = one_hot(left[i + x:i + SEQLEN] + right[j:j + x],char_indices)
                preds = model.predict(Xoh, verbose=0)[0]
                pred_probs = sample_preds(preds, 0.7)

                # Getting corresponding character in left word
                left_char = np.zeros((1, len(char_indices)))
                try:
                    left_char[0, char_indices[left[i + SEQLEN + x]]] = 1
                except IndexError:
                    pass
                # Adding some bias to left_char and adding it to predicted probs
                biased_probs = LEFT_BIAS[x] * left_char + \
                    (1 - LEFT_BIAS[x]) * pred_probs

                # Adding probability of bridging at c_index to s
                s += biased_probs[0, c_index]

            # Prioritizing words that start with the first few letters of the right word
            s = s * gaus_factors[j]

            if verbose:
                print (i, j, s,)
            best_matches[i][j] = s
            if s > best_j_score:
                best_j = j
                best_j_score = s
#         best_matches[i] = {'index': best_j, 'score': best_j_score}
        if best_j_score > best_i_score and i < len(left) - MINLEFT:
            best_i_score = best_j_score
            best_i = i

    return best_matches, best_i
```

## Picking the best portmanteaus
- Maximize smoothness of the bridge (derived from `proc` using the LSTM  model)
- Minimize length of portmanteau
- Maximize fraction of each word in portmanteau

```python
SEQLEN = 3
MAXLEN = 10
PHONEME_WT = 4

def join(left, right, verbose=False,dict_result=False,n=3):
    left = '\n' + left
    right = right + '\n'
    matches, i = proc(left, right, verbose)
    probs = {}
    for i_temp in matches:
        for j_temp in matches[i_temp]:
            word = (left[:i_temp + SEQLEN] + right[j_temp:]).replace('\n', '').title()
            num_letters = len(word)
            if verbose :
                print (word, num_letters,(1 / float(num_letters)) * 0.5)
            probs[word] = probs.get(word,0)+round(matches[i_temp][j_temp],4) + (1 / float(num_letters) * PHONEME_WT)
            probs[word] *= (min((i_temp+1)/min(len(left),8),1.0) + min((len(right) - j_temp - 1)/min(len(right),8),1.0))
    if dict_result:
        return probs
    else:
        ser = pd.Series(probs).sort_values()[::-1][:n]
        ports = ser.index.tolist()
        port_vals = [i+'('+str(round(ser[i],3))+')' for i in ports]
        print (left,'+',right,' = ',port_vals)
```

## Generating common portmanteaus
```python
word_pairs =  [('britain','exit'),('biology', 'electronic'), ('affluence', 'influenza'), ('brad', 'angelina'),
               ('brother', 'romance'), ('breakfast', 'lunch'), ('chill', 'relax'), ('emotion', 'icon'),('feminist', 'nazi')]

for p in word_pairs:
  join(p[0],p[1])
```
    britain + exit
    =  ['Britainexit(0.71)', 'Brexit(0.705)', 'Briexit(0.69)']

    biology + electronic
    =  ['Biolectronic(1.23)', 'Biolonic(0.821)', 'Bionic(0.677)']

    affluence + influenza
    =  ['Affluenza(2.722)', 'Affluenfluenza(1.261)', 'Affluencenza(1.093)']

    brad + angelina
    =  ['Brangelina(1.626)', 'Braangelina(0.637)', 'Bradangelina(0.635)']

    brother + romance
    =  ['Brotheromance(1.493)', 'Bromance(0.963)', 'Brothermance(0.625)']

    breakfast + lunch
    =  ['Breaunch(0.657)', 'Breakfasunch(0.59)', 'Breakfalunch(0.588)']

    chill + relax
    =  ['Chillax(1.224)', 'Chilax(1.048)', 'Chillelax(0.699)']

    emotion + icon
    =  ['Emoticon(1.331)', 'Emotion(0.69)', 'Emicon(0.667)']

    feminist + nazi
    =  ['Feminazi(1.418)', 'Femazi(0.738)', 'Feministazi(0.678)']

## Generating Pokemon names!
```python
pokemon_pairs = [('char','lizard'), ('venus', 'dinosaur'), ('blast', 'tortoise'), ('pikapika', 'chu')]        
for p in pokemon_pairs:
  join(p[0],p[1])
```

    char + lizard
    =  ['Chard(0.928)', 'Charizard(0.764)', 'Charlizard(0.698)']

    venus + dinosaur
    =  ['Venusaur(1.051)', 'Venosaur(0.945)', 'Venusdinosaur(0.661)']

    blast + tortoise
    =  ['Blastortoise(1.46)', 'Blastoise(1.121)', 'Blasttortoise(0.627)']

    pikapika + chu
    =  ['Pikachu(0.728)', 'Pikapikachu(0.714)', 'Pichu(0.711)']

## Try it yourself!
Wanna give it a spin ?
Run it on *Colab* or download the notebook and run it locally. <br/>**[bit.ly/ColabCharLSTM](bit.ly/ColabCharLSTM)**

--- 
