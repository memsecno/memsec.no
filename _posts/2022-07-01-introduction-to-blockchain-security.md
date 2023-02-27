---
title: "Introduction to Blockchain Security"
author: "Sirajuddin Asjad"
date: "2022-07-01"
tags: ["blockchain", "security testing", "fuzzing", "smart contracts"]
layout: post
categories: post
---

This an introduction post in a series of blog posts related to blockchain security. The security aspect of blockchain technology is huge and I have no intention of covering it all. I will focus on security vulnerabilities related to smart contracts and experiment with some security exploitation methods to make the most of these vulnerabilities. 

## What is Blockchain?
A blockchain is a distributed database or ledger that allows users and organizations to store and process data with the structured distributed blocks present in a blockchain network. Each new block stores a transaction or a bundle of transactions that is connected to all the previously available blocks in the form of a cryptographic chain. Blockchains are best known for their crucial role in cryptocurrency systems, such as Bitcoin, for maintaining a secure and decentralized record of transactions. The innovation with a blockchain is that it guarantees the fidelity and security of a record of data and generates trust without the need for a trusted third party [1].

## What is Smart Contracts?
TBD

## What is Solidity?
Solidity is an object-oriented, high-level programming language used to create smart contracts that automate transactions on the blockchain. After being proposed in 2014, the language was developed by contributors to the Ethereum project. The language is primarily used to create smart contracts on the Ethereum blockchain and create smart contracts on other blockchains [2].

## What is Blockchain Security? 
TBD

## Security Testing Methodologies
### Fuzz Testing
TBD

### Smart Contract Reverse Engineering
TBD

# Fuzzing Ethereum Smart Contracts
Echidna is an open source fuzzer designed to find bugs and vulnerabilities in Ethereum smarts contracts using grammar and coverage based fuzzing. The fuzzer generates test inputs based on the provided Solidity smart contract code and exploits the Contract Application Binary Interface (ABI), which is the standard way to interact with contracts in the Ethereum ecosystem, both from outside the blockchain and for contract-to-contract interaction. 

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

## References
[1] IBM (accessed 26 June 2022). ["What is blockchain technology?"](https://www.ibm.com/topics/what-is-blockchain) <br>
[2] MUO (accessed 26 June 2022). ["What Is Solidity and How Is It Used to Develop Smart Contracts?"](https://www.makeuseof.com/what-is-solidity/)
