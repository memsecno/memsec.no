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
First, we should try to understand what the provided jumble.py script does. After looking and running the code we see that print_flag.py needs to exist in the same directory as jumble.py, which the CTF author kindly tells us
> "# Need print_flag.py (secret) in the same directory as this file"
Now, I will not go through each part of the code individually, but based on code analyzes we determine the following:
1. get_rand_key function 
2. Second item
3. Third item
4. Fourth item
