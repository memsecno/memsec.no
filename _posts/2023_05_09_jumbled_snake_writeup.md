---
title: "Crypto: Can you unravel the jumbled snake?"
author: "Martin Svalstuen Brunæs"
date: "2023-05-15"
tags: ["CTF", "Crypto", "Python", "Substitution cipher"]
layout: post
categories: post
---

## San Diego CTF - Jumbled Snake Write Up
I attended the San Diego Capture The Flag (SDCTF) 2023, a jeopardy style CTF from May 6-8. In this blog post we will take a look at one of the crypto challenges in this CTF, namely, "jumbled snake". It was fun and a great learning experience!

### Challenge Text
![Screenshot from 2023-05-09 19-24-45](https://github.com/memsecno/memsec.no/assets/13424965/beed623f-d59c-40ab-aac4-b6fef10c9b71)

### Files provided:
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
### Substitution Cipher
A quick Google Search reveals the following citation on Wikipedia ""The quick brown fox jumps over the lazy dog" is an English-language pangram — a sentence that contains all the letters of the alphabet."
Isn't that just great? We have determined that we are dealing with a substituion cipher, and we have been provided with pangram that was in the original plaintext, and we have the resulting ciphertext. If we determine where the pangram appears in the ciphertext we can make a substitution table and use it on the ciphertext to get the plaintext. We see that before the pangram appears we observe triple quotes ("""). 
### Finding the encoded pangram in the ciphertext
We can write a python script that looks for all strings that start and end with three equal characters with a total length of 78 (length of pangram including triple quotes):
```python
import os

cipher = [] #Store the cipher once we found it
length = len("\"\"\"{'the_quick_brown_fox_jumps_over_the_lazy_dog': 123456789.0, 'items':[]}\"\"\"") #Length of the search-string.

with open(os.path.join(os.path.dirname(__file__), 'print_flag.py.enc')) as enc: #Open ciphertext file
    input = enc.read()

    for i in range(len(input)):
        if input[i] == input[i-1] and input[i] == input[i-2]:
            #If the current character equals the two previous.
            try:
                if input[i + length-3] == input[i] and input[i+length-4] == input[i] and input[i+length-5] == input[i]:
                    #If the same three characters appears at the end of the string
                    print("Found the matching pangram!")
                    for j in range(i-2, i+length-2, 1): #Append each character to an chipher array
                        cipher.append(input[j])
            except:
                break

cipherstring = ''.join(cipher) #Concatenate for readability
print(cipherstring)
```
Output:
```
Found the matching pangram!
]]]y8Pp X	%DSlXGEUeX@U$Xz%Mo\XUa EXPp XWtkXYU{8/ZRm:whA~_3>6'Z8DP M\8/j?V]]]
```
### Creating an initial substitution key
Now we need to create a substitution key that contains all the characters in the ciphertext and includes our panagram. Then we use the original jumble.py substition code to decode the cipher text:
```python
import secrets

pangram = "\"\"\"{'the_quick_brown_fox_jumps_over_the_lazy_dog': 123456789.0, 'items':[]}\"\"\""
ciphertext = "]]]y8Pp X	%DSlXGEUeX@U$Xz%Mo\XUa EXPp XWtkXYU{8/ZRm:whA~_3>6'Z8DP M\8/j?V]]]"

def get_rand_key(charset: str = string.printable):
    chars_left = list(charset)
    key = {}
    for char in charset:
        val = secrets.choice(chars_left)
        chars_left.remove(val)
        key[char] = val
    assert not chars_left
    return key
key = get_rand_key()

for i in range(len(pangram)):
    key[ciphertext[i]] = pangram[i]
print(key)

def subs(msg: str, key) -> str:
    test = ''.join(key[c] for c in msg)
    return test

with open(os.path.join(os.path.dirname(__file__), 'input.enc')) as enc, open('prototypedecoded.dec', 'w') as dec:

    dec.write(subs(enc.read(), key))

```
### Deducing unknown key-value pairs based on semi-decoded output
The resulting prototypedecoded.dec shows the following:
```
5z usrbinenv python3!import base64!!coded_flag s "c2ajdO"7PP91bl/hdjns"'afdygz^3nuE2shf9ss"!!def reverse.s[:!    return "".join.reversed.s[[!!def check.[:!    """O~C_r"#1_y

_ayP~_iuE8/_^~]_n'~aL_gJ	89_y

"""!    assert decode_flag.__doc__ is not none and decode_flag.__doc__.upper.[[2:45] ss reverse.check.__doc__[!!def decode_flag.code[:!    """{'the_quick_brown_fox_jumps_over_the_lazy_dog': 123456789.0, 'items':[]}"""!    return base64.b64decode.code[.decode.[!!if __name__ ss "__main__":!    check.[!    print.decode_flag.coded_flag[[!
```

We see that parts of the chiphertext is successfully converted to plaintext, but that our key is not complete. We can fill in the blanks by looking at common used characters such as "space", "lineshift", etc. to deduce the missing key value pairs for substitution. I.e. by  updating key-value pair from 'b':'!' to 'b':'\n' increases the readability by adding line shift. Doing the same for '()', 'N', and '=' gives the following output, that offers better readability:

```
5z usrbinenv python3
import base64

coded_flag = "c2ajdO"7PP91bl/hdjNs"'afdygz^3NuE2shf9=="

def reverse(s):
    return "".join(reversed(s))

def check():
    """O~C_r"#1_y

_ayP~_iuE8/_^~]_N'~aL_gJ	89_y

"""
    assert decode_flag.__doc__ is not None and decode_flag.__doc__.upper()[2:45] == reverse(check.__doc__)

def decode_flag(code):
    """{'the_quick_brown_fox_jumps_over_the_lazy_dog': 123456789.0, 'items':[]}"""
    return base64.b64decode(code).decode()

if __name__ == "__main__":
    check()
    print(decode_flag(coded_flag))

```

We see that this code when properly decoded will decode the coded_flag and print it. The check() function's docstring is still all scrambled up, but the function itself reveals
```
and decode_flag.__doc__.upper()[2:45] == reverse(check.__doc__)
```
which means that check() becomes:
```
def check():
    """GOD_YZAL_EHT_REVO_SPMUJ_XOF_NWORB_KCIUQ_EHT"""
    assert decode_flag.__doc__ is not None and decode_flag.__doc__.upper()[2:45] == reverse(check.__doc__)
```

The last crucial stage is to make sure that the coded_flag is propely decoded. Up until now our key only contains a lower case pangram, but now that we know that check() contains the same pangram with capital letters, this should be added to our key. This is done in the same way as the first pangram (see code in bottom of post). 

Now, the prototypedecoded.dec looks to be completely decoded. Let's test it!

```python
import base64

coded_flag = "c2RjdGZ7VV91blJhdjNsZWRfdEgzX3NuM2shfQ=="

def reverse(s):
    return "".join(reversed(s))

def check():
    """GOD_YZAL_EHT_REVO_SPMUJ_XOF_NWORB_KCIUQ_EHT"""
    assert decode_flag.__doc__ is not None and decode_flag.__doc__.upper()[2:45] == reverse(check.__doc__)

def decode_flag(code):
    """{'the_quick_brown_fox_jumps_over_the_lazy_dog': 123456789.0, 'items':[]}"""
    return base64.b64decode(code).decode()

if __name__ == "__main__":
    check()
    print(decode_flag(coded_flag))

```

By running this script we obtain the #### flag!
Script Output:
```
sdctf{U_unRav3led_tH3_sn3k!}
```

# Combined source code
```python

import os
import string
import secrets

cipher =  quick Goo[] #Store the cipher once we found it
#length = len("\"\"\"{'the_quick_brown_fox_jumps_over_the_lazy_dog': 123456789.0, 'items':[]}\"\"\"") #Length of the search-string.
length = len("\"\"\"GOD_YZAL_EHT_REVO_SPMUJ_XOF_NWORB_KCIUQ_EHT\"\"\"") #Length of the search-string.

with open(os.path.join(os.path.dirname(__file__), 'print_flag.py.enc')) as enc: #Open ciphertext file
    input = enc.read()

    for i in range(len(input)):
        if input[i] == input[i-1] and input[i] == input[i-2]:
            #If the current character equals the two previous.
            try:
                if input[i + length-3] == input[i] and input[i+length-4] == input[i] and input[i+length-5] == input[i]:
                    #If the same three characters appears at the end of the string
                    print("Found the matching pangram!")
                    for j in range(i-2, i+length-2, 1): #Append each character to an chipher list
                        cipher.append(input[j])
                    cipherstring = ''.join(cipher)  # Concatenate for readability
                    cipher = []
                    print(cipherstring)
            except:
                break


pangram = "\"\"\"{'the_quick_brown_fox_jumps_over_the_lazy_dog': 123456789.0, 'items':[]}\"\"\""
pangram1 = "GOD_YZAL_EHT_REVO_SPMUJ_XOF_NWORB_KCIUQ_EHT"
ciphertext = "]]]y8Pp X	%DSlXGEUeX@U$Xz%Mo\XUa EXPp XWtkXYU{8/ZRm:whA~_3>6'Z8DP M\8/j?V]]]"
ciphertext1 = """F+
Xf5}IX7|0X17s+XB&N)KXn+(X,O+1qXCQ*)`X7|0"""

def get_rand_key(charset: str = string.printable):
    chars_left = list(charset)
    key = {}
    for char in charset:
        val = secrets.choice(chars_left)
        chars_left.remove(val)
        key[char] = val
    assert not chars_left
    return key
key = get_rand_key()

for i in range(len(pangram)):
    key[ciphertext[i]] = pangram[i]
def subs(msg: str, key) -> str:
    test = ''.join(key[c] for c in msg)
    return test

keymod = {'0': '\r', '1': 'a', '2': '`', '3': '9', '4': '\x0b', '5': '"', '6': '0', '7': 'y', '8': "'", '9': 'T', 'a': 'v', 'b': '\n', 'c': '0', 'd': 'G', 'e': 'n', 'f': 'r', 'g': ')', 'h': '5', 'i': 'Q', 'j': '[', 'k': 'z', 'l': 'k', 'm': '2', 'n': '^', 'o': 'p', 'p': 'h', 'q': 'L', 'r': ';', 's': 'P', 't': 'a', 'u': '\x0c', 'v': '+', 'w': '4', 'x': 'j', 'y': '{', 'z': 'j', 'A': '6', 'B': 'i', 'C': 'g', 'D': 'i', 'E': 'r', 'F': 'O', 'G': 'b', 'H': 'x', 'I': '1', 'J': '=', 'K': '/', 'L': 'H', 'M': 'm', 'N': 'E', 'O': "'", 'P': 't', 'Q': 'J', 'R': '1', 'S': 'c', 'T': 'N', 'U': 'o', 'V': '}', 'W': 'l', 'X': '_', 'Y': 'd', 'Z': ' ', '!': 'm', '"': '3', '#': '&', '$': 'x', '%': 'u', '&': 'u', "'": ',', '(': ']', ')': '8', '*': '\t', '+': '~', ',': 'N', '-': '@', '.': '5', '/': ':', ':': '3', ';': '=', '<': 'B', '=': 'z', '>': '.', '?': ']', '@': 'f', '[': '>', '\\': 's', ']': '"', '^': '(', '_': '8', '`': '9', '{': 'g', '|': '\n', '}': '#', '~': '7', ' ': 'e', '\t': 'q', '\n': 'C', '\r': 'q', '\x0b': 'w', '\x0c': 'y'}

for i in range(len(pangram1)):
    keymod[ciphertext1[i]] = pangram1[i]

with open(os.path.join(os.path.dirname(__file__), 'input.enc')) as enc, open('prototypedecoded.dec', 'w') as dec:

    dec.write(subs(enc.read(), keymod))

```






