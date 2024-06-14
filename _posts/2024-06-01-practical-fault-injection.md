---
title: "BBQ ligher fault injection caused memory dump"
author: "Martin Svalstuen Brunæs"
date: "2024-06-01"
tags: ["fault injection", "hardware attack", "memory dump"]
layout: post
categories: post
---

### Introduction
I have read the hardware hacking handbook [1] and it shows a practical example of fault injection. 

### Setup
| ![signal-2024-06-01-142351_002](https://github.com/memsecno/memsec.no/assets/13424965/ea2cfb6a-de7c-4cd7-a5e0-e841fc029c49) |
|:--:|
| <b>Setup</b>|



### Glitching
| ![signal-2024-06-01-142450_002](https://github.com/memsecno/memsec.no/assets/13424965/9f91e3ac-9979-42d8-bf25-a7fab55edefa) |
|:--:|
| <b>Glitch occured</b>|
### Fault: Memory dump


| ![signal-2024-06-01-142450_002](https://github.com/memsecno/memsec.no/assets/13424965/0d6c9b64-f1b2-4111-b4fa-f3bc2a5c2c62) |
|:--:|
| <b>Glitch occured which caused memory dump</b>|


### Conclusion
It is interesting and knowledgeful how easy it is to start experimenting with fault injection. 


#### Sources
[1] O'Flynn, C., & van Woudenberg, J. (2021). The hardware hacking handbook: Breaking embedded security with hardware attacks. No Starch Press.
