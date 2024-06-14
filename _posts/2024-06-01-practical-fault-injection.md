---
title: "BBQ ligher fault injection caused memory dump"
author: "Martin Svalstuen Brunæs"
date: "2024-06-01"
tags: ["fault injection", "hardware attack", "memory dump"]
layout: post
categories: post
---

### Introduction
I have been facinated lately with fault injection attacks, and the hardware hacking handbook [1] has been of great value to learn about fault injection. I figured I wanted to test out one of the practical examples that was showcases in the book, and so I did. 



### Target Setup: ATMEGA 328P microcontroller.
The following code [1] was uploaded to the microcontroller:
``` cpp
void setup() {
  // Initialize the serial communication at 9600 bits per second:
  Serial.begin(9600);
}

// Declare a global variable to keep count of iterations
unsigned long cnt = 0;
// Declare a global variable to keep track of loop executions
unsigned int loopcnt = 0;

void loop() {
  // Reset cnt to 0 at the start of each loop
  cnt = 0;
  // Increment the loop counter to keep track of how many times the loop has run
  loopcnt++;

  // Nested loops to increment cnt
  for (unsigned int i = 0; i < 500; i++) {
    for (volatile unsigned int j = 0; j < 500; j++) {
      // Increment cnt in the innermost loop
      cnt++;
    }
  }

  // Print the value of cnt and loopcnt to the Serial Monitor
  Serial.print(cnt);
  Serial.print(" ");
  Serial.println(loopcnt);

  // Check if cnt is not equal to 250000
  if (cnt != 250000) {
    // If there is a discrepancy, print a glitch message
    Serial.println("<---- GLITCH");
  } else {
    // If cnt equals 250000, do nothing
    Serial.print("");
  }
}
```

### Practical Fault Setup

Here is a picture of my set up with the BBQ lighter. The Arduino Nano is connected via USB to my laptop without any kinds of isolator. It is very important that the spark is kept away from the 328P leads or any other connections on the arduino, as this high voltage may damage both the arduino and the laptop connected. It is possible to acquire a decoupler to avoid this risk, but in my case I did not experience any trouble regarding this. 

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
