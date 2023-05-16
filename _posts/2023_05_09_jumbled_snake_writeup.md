---
title: "Crypto: Can you unravel the jumbled snake?"
author: "Martin Svalstuen Brunæs"
date: "2023-05-15"
tags: ["CTF", "Crypto", "Python", "Substitution cipher"]
layout: post
categories: post
---

## San Diego CTF - Jumbled Snake Write Up
I attended the San Diego Capture The Flag (SDCTF) 2023, a jeopardy style CTF from May 6-8. In this blog post we will take a look at one of the crypto challenges in this CTF, namely, "jumbled snake".

### Challenge Text
![Screenshot from 2023-05-09 19-24-45](https://github.com/memsecno/memsec.no/assets/13424965/beed623f-d59c-40ab-aac4-b6fef10c9b71)

### Files content:
### jumble.py
``` python
#! /usr/bin/env python3
# Need print_flag.py (secret) in the same directory as this file
import print_flag
import os
import secrets
import string

def get_rand_key(charset: str = string.printable):
    chars_left = list(charset)
    key = {}
    for char in charset:
        val = secrets.choice(chars_left)
        chars_left.remove(val)
        key[char] = val
    assert not chars_left
    return key

def subs(msg: str, key) -> str:
    return ''.join(key[c] for c in msg)

with open(os.path.join(os.path.dirname(__file__), 'print_flag.py')) as src, open('print_flag.py.enc', 'w') as dst:
    key = get_rand_key()
    print(key)
    doc = print_flag.decode_flag.__doc__
    assert doc is not None and '\n' not in doc
    dst.write(doc + '\n')
    dst.write(subs(src.read(), key))
```

### print_flag.py.enc
```
{'the_quick_brown_fox_jumps_over_the_lazy_dog': 123456789.0, 'items':[]}
.=Z4%\E4GDe4 eaZoPpUe:bDMoUEPZGt\ AwbbSUY YX@Wt{ZJZ]Sm1zYF5~ss3RGWKpYz,\5O1@Y7{kn:,%Nm\p@`JJ]bbY @ZE a E\ ^\g/bZZZZE P%EeZ]]>zUDe^E a E\ Y^\ggbbY @ZSp Sl^g/bZZZZ]]]F+
Xf5}IX7|0X17s+XB&N)KXn+(X,O+1qXCQ*)`X7|0]]]bZZZZt\\ EPZY SUY X@Wt{>XXYUSXXZD\ZeUPZ,Ue ZteYZY SUY X@Wt{>XXYUSXX>%oo E^gjm/wh?ZJJZE a E\ ^Sp Sl>XXYUSXXgbbY @ZY SUY X@Wt{^SUY g/bZZZZ]]]y8Pp X	%DSlXGEUeX@U$Xz%Mo\XUa EXPp XWtkXYU{8/ZRm:whA~_3>6'Z8DP M\8/j?V]]]bZZZZE P%EeZGt\ Aw>GAwY SUY ^SUY g>Y SUY ^gbbD@ZXXetM XXZJJZ]XXMtDeXX]/bZZZZSp Sl^gbZZZZoEDeP^Y SUY X@Wt{^SUY YX@Wt{ggb
```

### Introduction
First, we should try to understand what the provided jumble.py script does. After looking and running the code we see that print_flag.py needs to exist in the same directory as jumble.py, which the CTF author kindly tells us:
> "# Need print_flag.py (secret) in the same directory as this file"

Now, I will not go through each part of the code individually, but based on code analyzes we can determine the following:

1. def get_rand_key returns a random key in a dictonary form with following charset:
```0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ!"#$%&'()*+, -./:;<=>?@[\]^_`{|}~ ```
What basically happens is that we feed secrets.choice() with our charset and the function returns one input character randomly. We build the key dictorary up this way, so we end up with 
``` key = {'0': 'Unique random character from charset', '1': 'Unique random character from charset', '2': 'Unique random character from charset', '3': 'Unique random character from charset'}... ```
3. def subs() takes the plaintext and the random key as an input and performs a substition of plaintext resulting in a cipher text. 
4. We now understand how print_flag.py.enc was created in the first place.  

At this point we know that we are dealing with a substitution cipher and we know that we have the encrypted output. jumble.py also gives us a valueble clue that helps us solve the puzzle:
```python
doc = print_flag.decode_flag.__doc__
    assert doc is not None and '\n' not in doc
    dst.write(doc + '\n')
```
Based on the code above we can deduce that the original print_flag.py must have included a function called decode_flag with a docstring. Subsequently, we see that the docstring is written to the first line of our output file, print_flag.py.enc. Let's take a look at the first line:
> {'the_quick_brown_fox_jumps_over_the_lazy_dog': 123456789.0, 'items':[]}
This results in the following print_flag.py:
``` python
def decode_flag():
    """{'the_quick_brown_fox_jumps_over_the_lazy_dog': 123456789.0, 'items':[]}"""
```    
A quick Google Search reveals the following citation on Wikipedia ""The quick brown fox jumps over the lazy dog" is an English-language pangram — a sentence that contains all the letters of the alphabet."
Isn't that just great? We have determined that we are dealing with a substituion cipher, and we have been provided with pangram that was in the original plaintext, and we have the resulting ciphertext. We see that the pangram stars with 

