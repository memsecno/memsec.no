---
title: "Electromagnetic Fault Injection in a Microcontroller"
author: "Martin Svalstuen Brunæs"
date: "2024-06-16"
tags: ["fault injection", "electromagnetic", "hardware attack", "flash memory dump", "microcontroller"]
layout: post
categories: post
---

### Introduction
Lately, I've been interested in fault injection attacks, and the Hardware Hacking Handbook [1] has been valuable for learning about them. I decided to try out one of the practical examples showcased in the book, and we will take a look at that experiment in this blog post. The experiment is based on that a cheap BBQ ligher can be used to create a spark including electromagnetic field in near vicinity of the microcontroller such that the field interferes with the microcontroller, causing unexpected behavior such as skipping instructions. In practice, this means that we place the BBQ ligher leads on top of the ATMEGA328P SMD package and make sparks. 


| ![signal-2024-06-01-142351_002](https://github.com/memsecno/memsec.no/assets/13424965/76730552-e2c5-43be-938b-621eac712c87) |
|:--:|
| <b>Principle of electromagnetic fault injection, image from [2]</b>|


### Target: ATMEGA328P microcontroller.
The target is an ATMEGA 328P microcontroller in a SMD package, used by the Arduino Nano development board, which is a popular development board for educational and hobbyist purposes. To determine if a fault has been successfully induced in the microcontroller, we use a nested loop and print a count variable that should behave predictably under normal operation. If a fault occurs, the count variable may not produce the expected value, indicating that a fault has occurred.

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

### Test Setup

Here is a picture of my test setup, featuring the Arduino Nano mounted on a breadboard, a BBQ lighter, and alligator clips to hold the lighter's leads in place. The Arduino Nano is connected to my laptop via USB without any isolator. It is crucial to keep the spark away from the ATMEGA328P leads or any other connections on the Arduino, as the high voltage could damage both the Arduino and the connected laptop. Although a decoupler can be used to prevent this risk, I accepted the risk and did not encounter any issues.

| ![signal-2024-06-01-142351_002](https://github.com/memsecno/memsec.no/assets/13424965/df73b92d-573a-4533-ad50-2f4d92800029) |
|:--:|
| <b>Test Setup</b>|

| ![signal-2024-06-01-142351_002](https://github.com/memsecno/memsec.no/assets/13424965/72e00ab7-f9f4-49a2-a0ba-d68f68e15483) |
|:--:|
| <b>Leads from the BBQ ligher is placed on top of the microcontroller</b>|


### Testing
Once the Arduino Nano was running and printing to the serial monitor, I began pressing the lighter to create sparks. It faulted on the second attempt! At this time, I was quite happy that the microcontroller faulted so quickly and effortlessly. That's pretty cool! After experimenting with different wire placements, the fault occurred consistently every 3rd to 5th attempt. I also encountered that sometimes the fault would cause the Arduino Nano to reboot.  

| ![signal-2024-06-01-142450_002](https://github.com/memsecno/memsec.no/assets/13424965/b2a550a2-7159-4c7e-877e-141582b4371b) |
|:--:|
| <b>The Spark</b>|

| ![signal-2024-06-01-142450_002](https://github.com/memsecno/memsec.no/assets/13424965/9f91e3ac-9979-42d8-bf25-a7fab55edefa) |
|:--:|
| <b>Serial Monitor: The Glitch!</b>|


### Flash Memory Dump
While testing various wire placements and the repeatability of the tests, the serial monitor suddenly began printing a whole bunch of ASCII characters, which was very surprising! Even more surprisingly was that the ASCII characters were recognizable from earlier projects that I used the Arduino Nano for.

| ![signal-2024-06-01-142450_002](https://github.com/memsecno/memsec.no/assets/13424965/0d6c9b64-f1b2-4111-b4fa-f3bc2a5c2c62) |
|:--:|
| <b>Glitch occured which caused a memory dump</b>|

Most of the ASCII output seen in the picture above was from a Riscure HW hacking CTF (https://github.com/Riscure/Rhme-2016). Specifically, the strings are from the casino.hex binary, which also can be found by analyzing it in Ghidra:

| ![signal-2024-06-01-142450_002](https://github.com/memsecno/memsec.no/assets/13424965/1141bed4-eaed-47a9-9c40-4186d0546545) |
|:--:|
| <b>Strings found in the casino.hex binary using Ghidra</b>|

### Security Implications
This is quite interesting from a security perspective! It implies that the Arduino Nano does not overwrite all flash memory when a new program is written. I did some research on the Arduino Nano's bootloader and found that it uses either optiboot or ATmegaBOOT [3]. Since my Arduino Nano works using the selection of Processor set to the "Old Bootloader" variant, I know that mine uses the ATmegaBOOT bootloader [3]. From my research [4][5], it is my understanding that ATmegaBOOT only erases the pages necessary for the new program data to be written. This means that any previous program data larger than the latest program data written to flash remains persistent and can potentially be recovered.

### Conclusion
Trying out fault injection on the ATMEGA328P with a basic BBQ lighter has been really fun. By causing electromagnetic interference near the microcontroller, I saw firsthand how even small disturbances, like sparks from the lighter, can mess with the microcontroller's normal behavior, such as skipping instructions. 

During my tests, I observed that ASCII characters from previous projects suddenly appeared on the serial monitor during fault injection. This occurred because when new programs are written to the Arduino Nano (e.g., write to flash memory of ATMEGA328P), not all data from the old program was erased. Only the memory pages (based on program size) required by the new program were cleared and replaced. Consequently, remnants of old program code could still reside in the microcontroller's flash memory, which is quite interesting from a security viewpoint.

Overall, this experiment taught me a lot about how susceptible microcontrollers like the ATMEGA328P are to EM interference. Ultimately, it is essential to consider physical attacks, such as fault injection, when securing systems as part of a holistic risk assessment.

#### Sources
[1] O'Flynn, C., & van Woudenberg, J. (2021). The hardware hacking handbook: Breaking embedded security with hardware attacks. No Starch Press. \
[2] https://www.researchgate.net/figure/Principle-of-electromagnetic-fault-injection_fig1_341810842 \
[3] https://arduino.stackexchange.com/questions/51866/arduino-nano-atmega328p-bootloader-difference \
[4] https://github.com/arduino/ArduinoCore-avr/blob/master/bootloaders/atmega8/ATmegaBOOT.c \
[5] https://www.avrfreaks.net/s/topic/a5C3l000000UcMSEA0/t161252

