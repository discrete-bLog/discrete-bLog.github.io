---
layout: post
author: RJ Rybarczyk
title: Bitcoin P2PKH Transaction Breakdown
---

(originally posted [here](https://medium.com/coinmonks/bitcoin-p2pkh-transaction-breakdown-bb663034d6df) on 19 Sep 2018)

The goal of this post is to give a thorough introduction to the most common type of Bitcoin transactions, *Pay-to-Public-Key-Hash* (P2PKH).
To achieve this, the following is presented:

- A succinct overview of the concept of an unspent transaction output (UTXO) and how a transaction is formed
- A breakdown of each piece of a P2PKH transaction
- An overview of how an unlocking and locking script work in conjunction to spend funds

A P2PKH transaction is the type of transactions that most people make when they move a specified amount of Bitcoin from one address to another usually via a wallet interface.
An example transaction is used to clearly demonstrate part of what is going on behind the scenes of an average wallet.

## Unspent Transaction Outputs (UTXOs)
A very important concept to understand before diving into the transaction breakdown is that of an unspent transaction output or UTXO.
In a Bitcoin transaction, UTXOs are what is being consumed, or spent.
A UTXO can only be spent once. After it is spent it is then referred to as a spent transaction output.

Spending a UTXO is similar to taking a $20 bill to buy something for $10 but instead of the cashier keeping the $20 bill, she lights it on fire and creates two new $10 bills out of thin air, keeping one and returning the other to you as change.

In this analogy the incinerated $20 bill started as a UTXO but once it was consumed became a spent transaction output which spawned two new $10 bills representing two new UTXOs ready to be spent in future transactions.

All the Bitcoin available for consumption on the network are referred to as the *UTXO set*.

## Transaction Building Process
Each transaction consists of the *version number*, *input*, *output*, and *locktime*.

The input contains an *outpoint(s)*, a *sequence number*, and an *unlocking script*, also called the *scriptSig*.

The output contains the *value* of the amount being spent and the *locking script*, also referred to as the *scriptPubKey*.

The locktime dictates at what time the transaction becomes valid.

Every transaction has a minimum of one input and one output.
The inputs contains logic that tells the network which UTXOs to consume (via the outpoints) and proves that it is allowed to consume them (via the unlocking scripts).
The output contains logic that tells the network the conditions in which a future transaction is allowed to consume the newly created UTXOs (via the locking script).

The figure below aims to map out the relationship between confirmed transaction with spent outputs, confirmed transactions with unspent outputs, and new, unsubmitted transactions.

[comment]:<> (Transaction Relationship image goes here)

### Steps to Build a Transaction
The steps to build a P2PKH transaction are as follows:
1. Identify a previous transaction that contains UTXOs that you have control over (just some Bitcoin that you already own).
2. Build the input outpoints of the new transaction to identify the previous transaction's UTXO to spend.
3. Build the outputs of the new transaction so the locking script will contain the conditions in which the newly created UTXOs can be spent/unlocked by the next transaction.
4. Create the unlocking script so it meets the conditions set by the previous transactions output's locking script. This contains the recipient's signature and is created at the very last step, but is very much a part of the input and is physically placed in the middle of the transaction.

## Introduction to the Serialized Transaction
A complete signed P2PKH Bitcoin transaction is discussed in detail below.

```
01000000019c2e0f24a03e72002a96acedb12a632e72b6b74c05dc3ceab1fe78237f886c48010000006a47304402203da9d487be5302a6d69e02a861acff1da472885e43d7528ed9b1b537a8e2cac9022002d1bca03a1e9715a99971bafe3b1852b7a4f0168281cbd27a220380a01b3307012102c9950c622494c2e9ff5a003e33b690fe4832477d32c2d256c67eab8bf613b34effffffff02b6f50500000000001976a914bdf63990d6dc33d705b756e13dd135466c06b3b588ac845e0201000000001976a9145fb0e9755a3424efd2ba0587d20b1e98ee29814a88ac00000000
```

This transaction was randomly chosen from Blockchain.com and can be viewed [here](https://www.blockchain.com/en/btc/tx/e65ad475a01384b086ce0d04199835fdd580739422ece1e0f1c4e362d43735d9) ([raw values](https://blockchain.info/rawtx/e65ad475a01384b086ce0d04199835fdd580739422ece1e0f1c4e362d43735d9)).

From a UTXO with a value of 17,373,066 address `19iy8HKpG5EbsqB2GUNVPUDbQxiTrPXpsx` sends 390,582 satoshi to `1JKRgG4F7k1b7PbAhQ7heEuV5aTJDpK9TS` and receives 16,932,484 satoshi back as change.
The remaining delta between the inputs and outputs go to the miner as a transaction processing fee.

## Version Number

```
**01000000**019c2e0f24a03e72002a96acedb12a632e72b6b74c05dc3ceab1fe78237f886c48010000006a47304402203da9d487be5302a6d69e02a861acff1da472885e43d7528ed9b1b537a8e2cac9022002d1bca03a1e9715a99971bafe3b1852b7a4f0168281cbd27a220380a01b3307012102c9950c622494c2e9ff5a003e33b690fe4832477d32c2d256c67eab8bf613b34effffffff02b6f50500000000001976a914bdf63990d6dc33d705b756e13dd135466c06b3b588ac845e0201000000001976a9145fb0e9755a3424efd2ba0587d20b1e98ee29814a88ac00000000
```

The version number is four bytes long and is expressed as a hexadecimal value in little endian format.
There are two version types.
Version 01 indicates that there is no relative time lock.
Version 02 indicates that there may be a relative time lock.
Version 02 was introduced in [BIP0068](https://github.com/bitcoin/bips/blob/master/bip-0068.mediawiki) which added `OP_CHECKSEQUENCEVERIFY` along with [BIP0112](https://github.com/bitcoin/bips/blob/master/bip-0112.mediawiki#Bidirectional_Payment_Channels) which upgraded it.
Version 02 is used in conjunction with the sequence number which is described below.

The example transaction is version 01.

## Transaction Input

```
01000000**019c2e0f24a03e72002a96acedb12a632e72b6b74c05dc3ceab1fe78237f886c48010000006a47304402203da9d487be5302a6d69e02a861acff1da472885e43d7528ed9b1b537a8e2cac9022002d1bca03a1e9715a99971bafe3b1852b7a4f0168281cbd27a220380a01b3307012102c9950c622494c2e9ff5a003e33b690fe4832477d32c2d256c67eab8bf613b34effffffff**02b6f50500000000001976a914bdf63990d6dc33d705b756e13dd135466c06b3b588ac845e0201000000001976a9145fb0e9755a3424efd2ba0587d20b1e98ee29814a88ac00000000
```

Each transaction input points to a previous transactions output.
If the unlocking script in the current transaction input meets the conditions set by the output script of the previous transaction, then the UTXO that the previous transaction "holds" can be spent.

### Number of Outpoints

```
01000000**01**9c2e0f24a03e72002a96acedb12a632e72b6b74c05dc3ceab1fe78237f886c48010000006a47304402203da9d487be5302a6d69e02a861acff1da472885e43d7528ed9b1b537a8e2cac9022002d1bca03a1e9715a99971bafe3b1852b7a4f0168281cbd27a220380a01b3307012102c9950c622494c2e9ff5a003e33b690fe4832477d32c2d256c67eab8bf613b34effffffff02b6f50500000000001976a914bdf63990d6dc33d705b756e13dd135466c06b3b588ac845e0201000000001976a9145fb0e9755a3424efd2ba0587d20b1e98ee29814a88ac00000000
```

The next one to nine bytes in the transaction input defines the number of included outpoints and is of type VarInt.
There is always one outpoint for each UTXO being consumed.

The example transaction as 1 UTXO as an input.

## Transaction Outpoint

```
0100000001**9c2e0f24a03e72002a96acedb12a632e72b6b74c05dc3ceab1fe78237f886c4801000000**6a47304402203da9d487be5302a6d69e02a861acff1da472885e43d7528ed9b1b537a8e2cac9022002d1bca03a1e9715a99971bafe3b1852b7a4f0168281cbd27a220380a01b3307012102c9950c622494c2e9ff5a003e33b690fe4832477d32c2d256c67eab8bf613b34effffffff02b6f50500000000001976a914bdf63990d6dc33d705b756e13dd135466c06b3b588ac845e0201000000001976a9145fb0e9755a3424efd2ba0587d20b1e98ee29814a88ac00000000
```

Each outpoint is a reference to a previous transaction hash and a corresponding index that points to the exact UTXO that is being spent.

The first 32 bytes of the outpoint is the reference to the previous transaction hash containing the UTXO being consumed in little endian format.
For example, the previous transaction hash is:

```
486c887f2378feb1ea3cdc054cb7b6722e632ab1edac962a00723ea0240f2e9c
```

and when it is included in the transaction outpoint, it expressed in little endian format(reverse byte order):

```
01**9c2e0f24a03e72002a96acedb12a632e72b6b74c05dc3ceab1fe78237f886c48**01000000
```

The last four bytes of the outpoint define the index of which UTXO of the previous transaction is being consumed.
It is also expressed has a hexadecimal value in little endian format.

```
019c2e0f24a03e72002a96acedb12a632e72b6b74c05dc3ceab1fe78237f886c48*01000000*
```

The transaction containing the UTXO being spent can be viewed [here](http://486c887f2378feb1ea3cdc054cb7b6722e632ab1edac962a00723ea0240f2e9c/) ([raw values)](https://blockchain.info/rawtx/486c887f2378feb1ea3cdc054cb7b6722e632ab1edac962a00723ea0240f2e9c).
The first entry in the previous transaction's "out" key shows that a UTXO containing a decimal value 17,373,066 satoshi is being consumed in the current transaction.
Note that the current transaction does not explicitly include the 17,373,066 satoshi UTXO, only the reference to it via the outpoint contents.


## Unlocking Script

The unlocking script, also called scriptSig, contains the stack script (signature) and redeem script (public key) of the user building the transaction and receiving the funds.

```
01000000019c2e0f24a03e72002a96acedb12a632e72b6b74c05dc3ceab1fe78237f886c4801000000**6a47304402203da9d487be5302a6d69e02a861acff1da472885e43d7528ed9b1b537a8e2cac9022002d1bca03a1e9715a99971bafe3b1852b7a4f0168281cbd27a220380a01b3307012102c9950c622494c2e9ff5a003e33b690fe4832477d32c2d256c67eab8bf613b34e**ffffffff02b6f50500000000001976a914bdf63990d6dc33d705b756e13dd135466c06b3b588ac845e0201000000001976a9145fb0e9755a3424efd2ba0587d20b1e98ee29814a88ac00000000
```

### Unlocking Script Length

```
**6a**47304402203da9d487be5302a6d69e02a861acff1da472885e43d7528ed9b1b537a8e2cac9022002d1bca03a1e9715a99971bafe3b1852b7a4f0168281cbd27a220380a01b330701
```

The first 1–9 bytes is of type VarInt and the defines the number of succeeding bytes that comprise the stack script and redeem script.
In this case, the first byte (0x6a) declares that the next 106 bytes are pushed to the stack (discussed below).

### Stack Script (Signature)

```
01000000019c2e0f24a03e72002a96acedb12a632e72b6b74c05dc3ceab1fe78237f886c48010000006a**47304402203da9d487be5302a6d69e02a861acff1da472885e43d7528ed9b1b537a8e2cac9022002d1bca03a1e9715a99971bafe3b1852b7a4f0168281cbd27a220380a01b33**07012102c9950c622494c2e9ff5a003e33b690fe4832477d32c2d256c67eab8bf613b34effffffff02b6f50500000000001976a914bdf63990d6dc33d705b756e13dd135466c06b3b588ac845e0201000000001976a9145fb0e9755a3424efd2ba0587d20b1e98ee29814a88ac00000000
```

The stack script contains the signature of the person submitting the transaction to the network and the sighash.
It is part of the proof that confirms the user (signer) is allowed to spend the UTXOs pointed to by the transaction outpoint.

The first 1–9 bytes is of type VarInt and the defines the number of succeeding bytes that comprise the signature.

```
**47**304402203da9d487be5302a6d69e02a861acff1da472885e43d7528ed9b1b537a8e2cac9022002d1bca03a1e9715a99971bafe3b1852b7a4f0168281cbd27a220380a01b330701
```

In this case, the first byte (0x47) declares that the next 71 bytes are the signature and sighash.
The signature is derived from the users private key.

```
47**304402203da9d487be5302a6d69e02a861acff1da472885e43d7528ed9b1b537a8e2cac9022002d1bca03a1e9715a99971bafe3b1852b7a4f0168281cbd27a220380a01b3307**01
```

The signature can be further broken down as follows:

- `**30**` - DER signature marker
- `**44**` - declares signature is 68 bytes in length
- `**02**` - r value marker
- `**20**` - declare r value is 32 bytes in length
- `**3da9d487be5302a6d69e02a861acff1da472885e43d7528ed9b1b537a8e2cac9**` - r value
- `**02**` - s value marker
- `**20**` - declare s value is 32 bytes in length
- `**02d1bca03a1e9715a99971bafe3b1852b7a4f0168281cbd27a220380a01b3307**` - s value


Finally, a single byte representing the sighash flag is appended to the signature.
In a P2PKH transaction, the sighash flag could be one of the following:


- `SIGHASH_ALL (0x01)` - the signature applies to all the inputs and the outputs, this is the most common for P2PKH transactions
- `SIGHASH_SINGLE (0x03)` - the signature applies to all the inputs and a single output
- `SIGHASH_ANYONECANPAY` combined with `SIGHASH_ALL (0x81)` - the signature applies to one input and all outputs
- `SIGHASH_ANYONECANPAY` combined with `SIGHASH_SINGLE (0x83)` - the signature applies to one input and one output

```
47304402203da9d487be5302a6d69e02a861acff1da472885e43d7528ed9b1b537a8e2cac9022002d1bca03a1e9715a99971bafe3b1852b7a4f0168281cbd27a220380a01b3307**01**
```

### Redeem Script

```
01000000019c2e0f24a03e72002a96acedb12a632e72b6b74c05dc3ceab1fe78237f886c48010000006a47304402203da9d487be5302a6d69e02a861acff1da472885e43d7528ed9b1b537a8e2cac9022002d1bca03a1e9715a99971bafe3b1852b7a4f0168281cbd27a220380a01b330701**2102c9950c622494c2e9ff5a003e33b690fe4832477d32c2d256c67eab8bf613b34e**ffffffff02b6f50500000000001976a914bdf63990d6dc33d705b756e13dd135466c06b3b588ac845e0201000000001976a9145fb0e9755a3424efd2ba0587d20b1e98ee29814a88ac00000000
```

The redeem script contains the conditions set by the scriptPubKey.
In P2PKH transactions, the redeem script is just the public key of the user (signer).

The first 1–9 bytes of the redeem script of type VarInt and the defines the number of succeeding bytes that comprise the public key hash.

```
**21**02c9950c622494c2e9ff5a003e33b690fe4832477d32c2d256c67eab8bf613b34e
```

In this case, the first byte (0x21) declares that the next 33 bytes are the user's public key hash.
The hashing algorithm is a HASH160 which is a SHA256 followed by a RIPEMD160 hash.
The public key is derived from the user's private key.

```
21**02c9950c622494c2e9ff5a003e33b690fe4832477d32c2d256c67eab8bf613b34e**
```

## Sequence Number (nSequence)

```
01000000019c2e0f24a03e72002a96acedb12a632e72b6b74c05dc3ceab1fe78237f886c48010000006a47304402203da9d487be5302a6d69e02a861acff1da472885e43d7528ed9b1b537a8e2cac9022002d1bca03a1e9715a99971bafe3b1852b7a4f0168281cbd27a220380a01b3307012102c9950c622494c2e9ff5a003e33b690fe4832477d32c2d256c67eab8bf613b34e**ffffffff**02b6f50500000000001976a914bdf63990d6dc33d705b756e13dd135466c06b3b588ac845e0201000000001976a9145fb0e9755a3424efd2ba0587d20b1e98ee29814a88ac00000000
```

nSequence is four bytes long and is expressed as a hexadecimal value in little endian format.

nSequence was originally there so that a user could replace a previously submitted unconfirmed transaction with an new transaction having the same input but a higher nSequence.
This did not work in practice however, because miners have no incentive to confirm transactions with a higher nSequence, instead preferring to confirm transactions with a higher fee.

Originally nSequence was only used for for disabling nLockTime (discussed below).
If nSequence is set to 0xFFFFFFFF then the nLockTime is ignored.


[BIP0125](https://github.com/bitcoin/bips/blob/master/bip-0125.mediawiki) introduced a new use for nSequence, using it to signal RBF if its value is less than `0xFFFFFFFE`.
An example where this is desirable is a case where a user submits their transaction to the network but realizes the fee was not sufficient and wants to resubmit the transaction that allocates a higher fee.

Transactions often default the nSequence to be `0xFFFFFFFE`, opting out of RBF and allowing nLockTime.
To use nLockTime and opt in to RBF, it is typical to set the nSequence to `0xFFFFFFFD`.

In [BIP0068](https://github.com/bitcoin/bips/blob/master/bip-0068.mediawiki), nSequence was repurposed to allow for the use of relative time locks.
This functionality is realized if and only if the transaction is version 02.

nSequence is essentially a duration of time that must pass for the transaction to be confirmed and is specified via block height or in sets of `2⁹ (512)` seconds from the current time.

A relative time locked transaction is invalid until `(nSequence * 521 seconds)` or `(nSequence * number of blocks)` after its parents transaction(s) are confirmed.

18 of the 32 bits of the nSequence value are used for relative time locks. The remaining 14 bits are reserved for future upgrades, as shown below:
```
 31              22              15              7             0 
|D 0 0 0 0 0 0 0|0 T 0 0 0 0 0 0|V V V V V V V V|V V V V V V V V|
**D** Bit 31: Disable Flag. 0 - enable time lock, 1 - disable time lock
**T** Bit 22: Type Flag. 0 - time in blocks, time in 2⁹ second sets
**V** Bits 15-0: Value. Number of blocks or 2⁹ second sets
```

A comprehensive explanation of time locks can be found [here](https://prestwi.ch/bitcoin-time-locks/).

The example transaction is version 01 with a sequence number of `0xFFFFFFF`, opting out of RBF and disabling the use of nLockTime.


## Transaction Outputs

```
01000000019c2e0f24a03e72002a96acedb12a632e72b6b74c05dc3ceab1fe78237f886c48010000006a47304402203da9d487be5302a6d69e02a861acff1da472885e43d7528ed9b1b537a8e2cac9022002d1bca03a1e9715a99971bafe3b1852b7a4f0168281cbd27a220380a01b3307012102c9950c622494c2e9ff5a003e33b690fe4832477d32c2d256c67eab8bf613b34effffffff**02b6f50500000000001976a914bdf63990d6dc33d705b756e13dd135466c06b3b588ac845e0201000000001976a9145fb0e9755a3424efd2ba0587d20b1e98ee29814a88ac**00000000
```

The transaction output contains the output scripts and the amount that those scripts control.
Most often, the transaction has more than one output.
In a typical P2PKH a transaction has an output containing the spend output and the change output.
Each output contains the amount, in satoshis, being consumed and the conditions that must be met in order to spend them.


### Number of Outputs

```
01000000019c2e0f24a03e72002a96acedb12a632e72b6b74c05dc3ceab1fe78237f886c48010000006a47304402203da9d487be5302a6d69e02a861acff1da472885e43d7528ed9b1b537a8e2cac9022002d1bca03a1e9715a99971bafe3b1852b7a4f0168281cbd27a220380a01b3307012102c9950c622494c2e9ff5a003e33b690fe4832477d32c2d256c67eab8bf613b34effffffff**02**b6f50500000000001976a914bdf63990d6dc33d705b756e13dd135466c06b3b588ac845e0201000000001976a9145fb0e9755a3424efd2ba0587d20b1e98ee29814a88ac00000000
```

The first byte in the transaction output defines the number of outputs and is of type VarInt.

In this transaction there are two outputs, one for the amount being spent from the incoming UTXO and the other for the change amount.


### Outputs

```
01000000019c2e0f24a03e72002a96acedb12a632e72b6b74c05dc3ceab1fe78237f886c48010000006a47304402203da9d487be5302a6d69e02a861acff1da472885e43d7528ed9b1b537a8e2cac9022002d1bca03a1e9715a99971bafe3b1852b7a4f0168281cbd27a220380a01b3307012102c9950c622494c2e9ff5a003e33b690fe4832477d32c2d256c67eab8bf613b34effffffff02**b6f50500000000001976a914bdf63990d6dc33d705b756e13dd135466c06b3b588ac845e0201000000001976a9145fb0e9755a3424efd2ba0587d20b1e98ee29814a88ac**00000000
```

The first 16 bytes of each output represents the amount being consumed as dictated by the unlocking (detailed below) and expressed as a hexadecimal value in little endian format.

```
01000000019c2e0f24a03e72002a96acedb12a632e72b6b74c05dc3ceab1fe78237f886c48010000006a47304402203da9d487be5302a6d69e02a861acff1da472885e43d7528ed9b1b537a8e2cac9022002d1bca03a1e9715a99971bafe3b1852b7a4f0168281cbd27a220380a01b3307012102c9950c622494c2e9ff5a003e33b690fe4832477d32c2d256c67eab8bf613b34effffffff02**b6f5050000000000**1976a914bdf63990d6dc33d705b756e13dd135466c06b3b588ac**845e020100000000**1976a9145fb0e9755a3424efd2ba0587d20b1e98ee29814a88ac00000000
```

The second part of the output is a variable length value called the locking script, or equivalently, the scriptPubKey.
The locking script defines the conditions in which the specified can be consumed.
In other words it only lets the user with the correct signature access the funds.

```
01000000019c2e0f24a03e72002a96acedb12a632e72b6b74c05dc3ceab1fe78237f886c48010000006a47304402203da9d487be5302a6d69e02a861acff1da472885e43d7528ed9b1b537a8e2cac9022002d1bca03a1e9715a99971bafe3b1852b7a4f0168281cbd27a220380a01b3307012102c9950c622494c2e9ff5a003e33b690fe4832477d32c2d256c67eab8bf613b34effffffff02b6f5050000000000**1976a914bdf63990d6dc33d705b756e13dd135466c06b3b588ac**845e020100000000**1976a9145fb0e9755a3424efd2ba0587d20b1e98ee29814a88ac00000000**
```

This P2PKH transaction is consuming a UTXO previous controlled by the address `19iy8HKpG5EbsqB2GUNVPUDbQxiTrPXpsx` with the value of 17,373,066 satoshi.
The first output of this transaction creates a new UTXO with the value of 390,582 satoshi that is controlled by receiving address, `1JKRgG4F7k1b7PbAhQ7heEuV5aTJDpK9TS`.

The second output of this transaction creates a new UTXO with the value of 16,932,484 satoshi that is controlled by the original sending address, `19iy8HKpG5EbsqB2GUNVPUDbQxiTrPXpsx`.
This is the "change" from the transaction.


#### First Output

The first output amount in decimal value is 390,582 satoshi (`0x0005F5B6` hex).

```
**b6f5050000000000**1976a914bdf63990d6dc33d705b756e13dd135466c06b3b588ac
```

The locking script is derived from the address of the user receiving the funds.

```
b6f5050000000000**1976a914bdf63990d6dc33d705b756e13dd135466c06b3b588ac**
```

The locking script is just a script, a piece of code, that tells the network how this amount can be spent.
The code is written in Bitcoin Script, which is a stack-based Reverse Polish Notation (RPN) language that is Turing incomplete.
It is a very primitive language that essentially just pops values on and off the stack in and performs operations on the items via a series of commands called [OP_CODES](https://en.bitcoin.it/wiki/Script).


The locking script is derived from the address of the person receiving the funds, and is broken down as follows:


- `**19**` - (25 bytes) byte length of the following unlocking script
- `**76**` - `OP_DUP`: Duplicates the top stack item.
- `**a9**` - `OP_HASH160`: The input is hashed twice: first with SHA-256 and then with RIPEMD-160
- `**14**` - (20 bytes) length of the following public key hash
- `**bdf63990d6dc33d705b756e13dd135466c06b3b5**` - public key hash of funds receiver
- `**88**` - `OP_EQUALVERIFY`: Returns 1 if the inputs are exactly equal, 0 otherwise. Then runs OP_VERIFY which fails and marks the transaction as invalid of the top stack value is false
- `**ac**` - `OP_CHECKSIG`: The entire transaction's outputs, inputs, and script are hashed. The signature used by OP_CHECKSIG must be a valid signature for this hash and public key. If it is, 1 is returned, 0 otherwise


#### Second Output

```
01000000019c2e0f24a03e72002a96acedb12a632e72b6b74c05dc3ceab1fe78237f886c48010000006a47304402203da9d487be5302a6d69e02a861acff1da472885e43d7528ed9b1b537a8e2cac9022002d1bca03a1e9715a99971bafe3b1852b7a4f0168281cbd27a220380a01b3307012102c9950c622494c2e9ff5a003e33b690fe4832477d32c2d256c67eab8bf613b34effffffff02b6f50500000000001976a914bdf63990d6dc33d705b756e13dd135466c06b3b588ac845e020100000000**1976a9145fb0e9755a3424efd2ba0587d20b1e98ee29814a88ac**00000000
```

The second output value is 16,932,484 satoshi.

```
**845e020100000000**1976a9145fb0e9755a3424efd2ba0587d20b1e98ee29814a88ac
```

The locking script for the second output sends 16,932,484 satoshi back to the sender.
This is the sender's change.

```
845e020100000000**1976a9145fb0e9755a3424efd2ba0587d20b1e98ee29814a88ac**
```

The locking script can be broken down as follows


- `**19**` - (25 bytes) byte length of the following unlocking script
- `**76**` - `OP_DUP`: Duplicates the top stack item.
- `**a9**` - `OP_HASH160`: The input is hashed twice: first with SHA-256 and then with RIPEMD-160.
- `**14**` - (20 bytes) length of the following public key hash
- `**5fb0e9755a3424efd2ba0587d20b1e98ee29814a**` - public key hash of funds receiver
- `**88**` - `OP_EQUALVERIFY`: Returns 1 if the inputs are exactly equal, 0 otherwise. Then runs OP_VERIFY which fails and marks the transaction as invalid of the top stack value is false
- `**ac**` - `OP_CHECKSIG`: The entire transaction's outputs, inputs, and script are hashed. The signature used by OP_CHECKSIG must be a valid signature for this hash and public key. If it is, 1 is returned, 0 otherwise


#### Transaction Fee

The transaction fee that goes to the miner is the delta between the total input and output amounts.
In the example transaction, the miner fee is calculated as:

```
miner_fee = (input_amount) - (spend_amount + change_amount)
miner_fee = (17373066) - (390582 + 16932484)
miner_fee = 50000 satoshi
```


## Locktime (nLockTime)

```
01000000019c2e0f24a03e72002a96acedb12a632e72b6b74c05dc3ceab1fe78237f886c48010000006a47304402203da9d487be5302a6d69e02a861acff1da472885e43d7528ed9b1b537a8e2cac9022002d1bca03a1e9715a99971bafe3b1852b7a4f0168281cbd27a220380a01b3307012102c9950c622494c2e9ff5a003e33b690fe4832477d32c2d256c67eab8bf613b34effffffff02b6f50500000000001976a914bdf63990d6dc33d705b756e13dd135466c06b3b588ac845e0201000000001976a9145fb0e9755a3424efd2ba0587d20b1e98ee29814a88ac0**0000000**
```

nLockTime allows for the use of absolute time locks.
It is four bytes long and is expressed as a hexadecimal value in little endian format.
Absolute time locks enables the user to submit a transaction to the network that will remain invalid until a specified time has passed.
nLockTime is essentially a target in time for the transaction to be confirmed and is specified via block number or epoch time.

If nLockTime is 0, then there is no time lock. The transaction is immediately valid.

If nLockTime is greater than 0 and less than 500,000,000, the absolute time lock is measured in terms of block numbers.

If nLockTime is greater than or equal to 500,000,000, the absolute time lock is measured in terms of epoch time.

It is important to note that if nSequence is set to 0xFFFFFFFF then nLockTime is completely disabled.

The example transaction has a nLockTime that is 0.


## Unlocking the Locking Script

Below shows how the locking and unlocking scripts work together to unlock the funds in a P2PKH output.
Only the values for the first output are shown, but this logic is applicable to both outputs.

The left most column displays the locking script contents, the middle column displays the unlocking script contents, and (remembering that Bitcoin Script is a Forth-like RPN language), the right most column shows how the stack is built.

```
|   Locking Script  |   Unlocking Script    |       Script      |
|                   |                       |                   |
|   OP_DUP          |   <signature>         |   <signature>     |
|   OP_HASH160      |   <pubkey>            |   <pubkey>        |
|   <pubkey hash>   |                       |   OP_DUP          |
|   OP_EQUALVERIFY  |                       |   OP_HASH160      |
|   OP_CHECKSIG     |                       |   <hash>          |
|                   |                       |   OP_EQUALVERIFY  |
|                   |                       |   OP_CHECKSIG     |
```

where, for the first output

```
<pubkey hash> - bdf63990d6dc33d705b756e13dd135466c06b3b5
<signature> - 304402203da9d487be5302a6d69e02a861acff1da472885e43d7528ed9b1b537a8e2cac9022002d1bca03a1e9715a99971bafe3b1852b7a4f0168281cbd27a220380a01b330701
<pubkey> - 02c9950c622494c2e9ff5a003e33b690fe4832477d32c2d256c67eab8bf613b34e
```

The script is executed as follows

```
|     Script           |        Stack         |     Processed       |
|                      |                      |                     |
|   <signature>        |                      |                     |
|   <pubkey>           |                      |                     |
|   OP_DUP             |                      |                     |
|   OP_HASH160         |                      |                     |
|   <pubkey hash>      |                      |                     |
|   OP_EQUALVERIFY     |                      |                     |
|   OP_CHECKSIG        |                      |                     |
```

Push the signature and then the public key onto the stack:

```
|     Script           |        Stack         |     Processed       |
|                      |                      |                     |
|                      |                      |                     |
|                      |                      |                     |
|   OP_DUP             |                      |                     |
|   OP_HASH160         |                      |                     |
|   <pubkey hash>      |                      |                     |
|   OP_EQUALVERIFY     |    <pubkey hash>     |                     |
|   OP_CHECKSIG        |    <signature>       |                     |
```

`OP_DUP` duplicates the top stack item, the public key, and pushes to the top of the stack:

```
|     Script            |       Stack         |     Processed       |
|                       |                     |                     |
|                       |                     |                     | 
|                       |                     |                     | 
|                       |                     |                     | 
|   OP_HASH160          |                     |                     | 
|   <pubkey hash>       |    <pubkey>         |                     | 
|   OP_EQUALVERIFY      |    <pubkey>         |                     | 
|   OP_CHECKSIG         |    <signature>      |     OP_DUP
```

`OP_HASH160` pops the top item off the stack and performs a `SHA256` hash and then a `RIPEMD160` hash on it:

```
<pubkey hash> = RIPEMD160(SHA256(public key))
```

It then pushes that hash to the top of the stack:

```
|     Script            |       Stack         |     Processed       |
|                       |                     |                     |
|                       |                     |                     | 
|                       |                     |                     | 
|                       |                     |                     | 
|                       |                     |                     | 
|   <pubkey hash>       |    <pubkey hash>    |                     | 
|   OP_EQUALVERIFY      |    <pubkey>         |                     | 
|   OP_CHECKSIG         |    <signature>      |     OP_HASH160      |
```

The public key hash is pushed to the top of the stack:

```
|     Script            |       Stack         |     Processed       |
|                       |                     |                     |
|                       |                     |                     | 
|                       |                     |                     | 
|                       |                     |                     | 
|                       |    <pubkey hash>    |                     | 
|                       |    <pubkey hash>    |                     | 
|   OP_EQUALVERIFY      |    <pubkey>         |                     | 
|   OP_CHECKSIG         |    <signature>      |                     |
```

`OP_EQUALVERIFY` pops top two items off the stack and checks that they are equivalent.
If the two public key hashes are not equal, then the recipient attempting to claim the funds provided an incorrect public key either by accident or maliciously.
`OP_EQUALVERIFY` will fail the script and the transaction is invalid if this occurs.

```
|     Script            |       Stack         |     Processed       |
|                       |                     |                     |
|                       |                     |                     | 
|                       |                     |                     | 
|                       |                     |                     | 
|                       |                     |                     | 
|                       |                     |                     | 
|                       |    <pubkey>         |                     | 
|   OP_CHECKSIG         |    <signature>      |     OP_EQUALVERIFY  |
```

Finally, `OP_CHECKSIG` pops the top two items off the stack, which should be a public key and a signature, and checks that the signature is valid for the public key.

If `OP_CHECKSIG` runs successfully, it pushes a 1 to the stack and the transaction is valid and complete.

If the user did not provide a valid signature for the public key, the `OP_CHECKSIG` pushes 0 to the stack, indicating the transaction is invalid.


```
|     Script            |       Stack         |     Processed       |
|                       |                     |                     |
|                       |                     |                     | 
|                       |                     |                     | 
|                       |                     |                     | 
|                       |                     |                     | 
|                       |                     |                     | 
|                       |                     |                     | 
|                       |    1                |     OP_CHECKSIG     |
```
