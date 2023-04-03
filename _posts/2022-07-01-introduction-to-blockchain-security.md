---
title: "Introduction to Blockchain Security"
author: "Sirajuddin Asjad"
date: "2023-02-06"
tags: ["blockchain", "security testing", "fuzzing", "smart contracts"]
layout: post
categories: post
---

This an introduction post in a series of blog posts related to blockchain security. The security aspect of blockchain technology is huge and I have no intention of covering it all. I will focus on security vulnerabilities related to smart contracts and experiment with some security testing and exploitation methodologies to make the most of these vulnerabilities. 

## Introduction: What is Blockchain?
A blockchain is a distributed database or ledger that allows users and organizations to store and process data with the structured distributed blocks present in a blockchain network. Each new block stores a transaction or a bundle of transactions that is connected to all the previously available blocks in the form of a cryptographic chain. Blockchains are best known for their crucial role in cryptocurrency systems, such as Bitcoin, for maintaining a secure and decentralized record of transactions. The innovation with a blockchain is that it guarantees the fidelity and security of a record of data and generates trust without the need for a trusted third party.

![blockchain](https://unova.io/wp-content/uploads/2021/11/blockchainnetworkunova-2048x1740.png)

## What is Smart Contracts?
Smart contracts are self-executing contracts that contain the terms and conditions of a legal agreement between two or more parties. They are written in computer code and stored on a blockchain, which enables them to automatically execute and enforce the terms of the agreement. As a result of this automation, all participants can be immediately certain of the outcome without any intermediary’s involvement or time loss. One of the key benefits of smart contracts is that they are transparent, secure, and tamper-proof. Once a smart contract is deployed on the blockchain, it cannot be altered or deleted and its execution is guaranteed as long as the conditions are met. Smart contracts do not contain legal language, terms or agreements - only code that executes actions when specified conditions are met. 

## What is Solidity?
Solidity is an object-oriented, high-level programming language used to create smart contracts that automate transactions on the blockchain. After being proposed in 2014, the language was developed by contributors to the Ethereum project. The language is primarily used to create smart contracts on the Ethereum blockchain and create smart contracts on other blockchains.

## What is Blockchain Security? 
Blockchain security refers to the protection of blockchain-based systems against unauthorized access, modification or destruction of data. Since blockchains are decentralized, there is no central authority in control of the network. Instead, the network is made up of nodes, each of which stores a copy of the Blockchain. In order for a hacker to tamper with the blockchain network, they would in principle need to hack every single node in the network, which is an extremely difficult feat. 

It's important to note that not all blockchain hacks are due to technical vulnerabilities. Social engineering attacks, such as phishing, can be used to trick users into giving away their private keys or other sensitive information, allowing attackers to access their cryptocurrency holdings.

## Blockchain Security Challenges
Even though blockchain is a robust technology, it is not immune to exploitation by cyber criminals and there are several existing vulnerabilities that can be exploited by hackers using different methodologies and attack vectors. Some of the potential methodologies are: 

* Routing attacks - Blockchains rely heavily on real-time data transfers, which can be intercepted by hackers using routing attacks. Unfortunately, this interception can occur unnoticed as the data travels to Internet Service Providers (ISP), leaving blockchain users unaware of the security breach.

* Sybil attacks - Another potential attack methodology is the so-called sybil attack, which floods the target network with an overwhelming amount of false identities, causing the system to crash. The attack uses a single node to operate many active fake identities (or Sybil identities) simultaneously within a peer-to-peer network.

* 51% attacks - When it comes to large scale public blockchains, mining operations require a significant amount of computing power. Unfortunately, if a group of unethical miners manage to pool their resources and acquire more than 50% of a blockchain network's mining power, they can carry out what is known as a 51% attack. This type of attack allows them to take control of the ledger, which can have devastating consequences such as being able to prevent new transactions from gaining confirmations and allowing hackers to halt payments between some or all users. 

* Phishing attacks - Even though this one is pretty obvious, blockchain users are still prone to phishing attacks just like any regular phishing attack. An example for phishing attack is where cyber criminals send false but convincing-looking emails to wallet owners, asking for their credentials. 

# Hacking the Blockchain
Smart contracts are written in code and can naturally contain bugs and security flaws that can be exploited by hackers to steal cryptocurrency or manipulate the blockchain. A huge number of security vulnerabilities have been discovered in smart contracts through different testing and assessment techniques, and this has become a hot topic among the security community. Blockchain hacking and smart contract reverse engineering are interesting fields of research and it's gradually increasing in popularity as new security tools and assessment techniques are being developed and brought forward to the community. 

The Smart Contract Weakness Classification Registry (SWC Registry) offers a complete and up-to-date catalogue of known smart contract vulnerabilities and anti-patterns along with real-world examples. Some of the discovered flaws are quite serious, such as unencrypted data being exposed in public, signature validation flaws, poor authentication mechanisms and buffer overflows. This blog post will not cover these flaws in detail, but feel free to visit the public registry to read more about the discovered vulnerabilities. 

Link: https://swcregistry.io

## Reverse Engineering and Fuzz Testing
Smart contract fuzzing is a testing technique used to discover security vulnerabilities in smart contracts, which involves sending random or modified inputs to a smart contract to test its behavior and identify any potential vulnerabilities. The goal is to identify input values that cause the smart contract to behave unexpectedly, such as allowing access to funds or executing unintended transactions to an unauthorized user. Threat actors use fuzzing to find zero-day exploits in software, which is known as a fuzzing attack, and similarly security professionals leverage fuzzing techniques to assess the security and stability of applications. 

## Fuzzing Ethereum Smart Contracts
Echidna is an open source fuzzer designed to find bugs and vulnerabilities in Ethereum smarts contracts using grammar and coverage based fuzzing. The fuzzer generates test inputs based on the provided Solidity smart contract code and exploits the Contract Application Binary Interface (ABI), which is the standard way to interact with contracts in the Ethereum ecosystem, both from outside the blockchain and for contract-to-contract interaction. 

![echidna-fuzzer](https://i.imgur.com/saFWti4.png)
(image source: https://github.com/crytic/echidna)

Upon fuzzing a smart contract it is necessary to provide a list of conditions that should be satisfied during the testing procedure (e.g. “a user can never have a negative number of coins” or "the wallet balance must be greater than active transfer amount"). Based on these user-defined conditions, Echidna will generate a large number of random transaction sequences, calls the contracts with these transaction sequences and checks that the conditions are still satisfied after the contracts are executed. 

Since Echidna uses coverage-guided fuzzing, it will not only use randomization to generate transaction sequences, but it also considers how much of the contract code was reached by previous random sequences. Coverage allows bugs to be found more quickly since it favors sequences that go deeper into the contract source code. 

## Simple example
I will demonstrate a simple example just to explain the basics. The following Solidity code snippet is a simple contract which conceptualizes a blog structure called `MemsecBlog`. 

``` solidity
contract MemsecBlog
{
    uint public posts;
    uint public visitors;
    string[] public categories;

    function addBlogPost() external 
    {
        posts += 1;
    }

    function addCategory(string memory category) public 
    {
        categories.push(category);
    }
}

contract FuzzMemsecBlog is MemsecBlog
{
    function echidna_fuzz_addBlogPost() public view returns (bool) 
    {
        return posts <= 5;
    }    
}
```

This was a short introduction to Blockchain Security and some security testing techniques. In the next blog post I will go more in detail about reverse engineering smart contracts and how it can be exploited. 

## References
[1] https://www.ibm.com/topics/what-is-blockchain <br>
[2] https://www.makeuseof.com/what-is-solidity <br>
[3] https://unova.io/wp-content/uploads/2021/11/blockchainnetworkunova-2048x1740.png 
