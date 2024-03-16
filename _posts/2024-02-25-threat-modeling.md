---
title: "What is threat modeling?"
author: "Martin Svalstuen Brunæs"
date: "2024-03-16"
tags: ["Threat Modeling", "Security Engineering", "Security Posture"]
layout: post
categories: post
---





### What is threat modeling?

Threat modeling “works to identify, communicate, and understand threats and mitigations within the context of protecting something of value.” [1]. Threat modeling can also refer to a set of activities where key stakeholders of the system-of-interest (SOI) evaluate the SOI’s security from different perspectives. The objective of threat modeling is to identify security threats to the SOI, thereby, enabling development teams or manufacturers to implement controls to mitigate the threats. 

Threat modeling can be done using a variety of different threat modeling methodologies. Each methodology can have different ways to identify and categorize threats to the SOI. Once threats have been identified and the consequence categorized (e.g., CIA triad), the results can be documented and handled through risk management. Subsequently, the threats can be mitigated, accepted, or deferred based on a risk-based evaluation. 

A key part of threat modeling is to identify the threats, which is usually governed by the threat modeling methodology. In my experience, simply put, visual illustrations plays a vital part to communicate how a system is structured (it’s architecture) and how it works (the system behavior). Any illustration that manages to communicate these two things are suitable as input to a threat modeling process, in my opinion. To identify threats one must understand the SOI and be able to communicate that understanding with other stakeholders in order to collectively reason about the SOI’s security. This is especially true for multidisciplinary systems where threats can emerge as a result of complex integration between many disciplines. 

Examples of visual illustrations are artifacts such as Physical Architecture, Logical Architecture, Functional Architecture, Activity Diagrams, Data Flow Diagrams, Attack trees, State Machines, Network Architecture, and so on. These diagrams helps us to reason about the system from different perspectives, together with a threat modeling methodology to guide us along. 


### How can you perform threat modeling?

Threat modeling can also be an integrated part of company's system development process through milestones or decision gates. The first step to performing threat modeling starts by understanding who the key stakeholders of the SOI are. This can be the the system architect, software architect, hardware architect, testers, security engineers, penetration testers, or others who understand parts of the SOI very well. Invite the stakeholders to a meeting and have each of them bring with them a design artifact. Examples can be physical architecture drawing, Data Flow Diagrams, PCB Layout, State Diagrams that can be used as means of reasoning about the SOI’s security. Have these artifacts available and use time blocks (e.g., 15 min each) to identify different attack vectors pr. illustration. Feel free to follow a known methodology or create your own that suites the system type (e.g., OT system) and the company's processes.


### When should you perform threat modeling?

Threat modeling can be applied any time that is suited, but the value added is usually greater when done early in the development stages. Threat modeling should be performed throughout the SOI’s life cycle, and it is paramount to think about security as an integrated part of the SOI’s design process. Thinking of security early on enables security considerations to influence the system architecture. The effectiveness and cost of implementing security controls tend to increase in the later stages of the development, as a result of high-level architecture being locked. However, that being said, one does not need to spend a lot of time to benefit from threat modeling. Having just a few sessions will probably allow you to identifying some low hanging fruits that when mitigated, improves the SOI’s security posture. 


### Example Illustrations

| ![space-1.jpg](https://github.com/memsecno/memsec.no/assets/13424965/d4033185-6761-4cf5-9e35-837d31511bd9) |
|:--:|
| <b>Attack tree for opening a safe [2]</b>|

This attack tree (inluding logic gates) is used to structure the information of possible attacks that results in a open safe. 


| ![space-1.jpg](https://github.com/memsecno/memsec.no/assets/13424965/374ca812-3d25-4102-9c67-34fa01147b64) |
|:--:|
| <b>Data Flow Diagram of a hypothetical kernel-mode Windows Driver Model (WDM) driver [3]</b>|

This Data Dlow Diagram visualizes the interactions between external entities (e.g., user and operating system) and the driver as well as showing internal behavior of the driver. This diagram can be used to identify assets to be attacked and which attack paths that can be used.


#### Sources
[1] https://owasp.org/www-community/Threat_Modeling \
[2] https://calibreone.com.au/what-are-cyber-attack-trees \
[3] https://learn.microsoft.com/en-us/windows-hardware/drivers/driversecurity/threat-modeling-for-drivers

