---
layout:     post
title:      How Segwit and the Lightning Network Work
date:       2017-06-24
summary:    An ELI5 explanation of Segwit, the Lightning Network, and how they are related
categories: bitcoin cryptocurrency
---

**Preface**
For a long time, I’ve had a vague understanding of what Segwit is: that it’s some kind of scaling solution, and necessary for the “Lightning Network”. However, I was never able to find a simple, ELI5 explanation of how it works. I finally took the time today to really understand it, and so here’s my attempt to explain it concretely.
 
For the purposes of this explanation, let’s assume that Bitcoin transactions look like this<sup>[1](#fn1)</sup>:
```
Tx: {
    Input: [(<previous TxID>,<tx output #>), ...],
    Output: [(<destination addresses>, <amount>), ...],
    Signature: [(<TxIDs>, <TxID’s relevant cryptographic signature(s)>)]
}
```

`TxID` stands for Transaction ID, and is the hash of some transaction block (like this one). Note that it’s not part of the transaction itself, but calculated based on the contents of the transaction.
 
`Output` is a list of amounts and destinations. It’s a list because a transaction can produce multiple outputs. For each output, there’s a controlling address--that’s who needs to sign any transaction that uses the coins in that output.
 
`Input` is a list of outputs from previous transactions. The btc from those transactions will be spent in this transaction. And for each input we need a...
 
`Signature` is a list of the Signatures required to spend the transaction. Whoever owns the outputs of the previous transactions that are being used as the inputs to this one needs to use their private key to generate a Signature, saying they approve of their coins’ use.
 
Also, note that there’s currently a 1mb hard limit on the sizes of blocks, and that signatures are typically about half the data in a transaction. Transactions that can’t fit in a block wait in the “mempool” to be included in a future block.
 
**Malleability**
It turns out that you can produce a new valid signature for a transaction, given an existing valid one. That is, if I sign a transaction, anybody can produce another identical transaction (same inputs, outputs, etc), except with a different valid signature. This is simply a quirk of the underlying cryptography<sup>[2](#fn2)</sup>.
Typically, this isn’t a problem: If someone intercepts my transaction in the mempool and commits another, identical one, no harm is done. I don’t care whether my signature is bit-by-bit what ends up on the Bitcoin blockchain; I only care that you need my approval to use my Bitcoins. 
However, notice that changing my signature will change the TxID, since it’s based on all contents of the transaction. That means that, if I’m doing something that depends on the TxID of a transaction that isn’t committed to the blockchain yet, anyone (including myself) with knowledge of that uncommitted transaction can potentially sabotage any dependent transaction by committing a copy of the base Tx with a different signature.
Spoiler: the Lightning Network depends on uncommitted transactions!

**Segwit: a malleability fix**
Segwit stands for “segregated witness,” and “witness” is just another word for the signature. This fixes the problem of malleability by allowing for transactions that move the signatures off of the transaction, into another part of the block. So, a transaction looks like this:
```
Tx: {
    Input: [(<previous TxID>,<tx output #>), ...],
    Output: [(<destination addresses>, <amount>), ...],
}
```
And somewhere else in the block we have:
```
Sig: {
    Signature: [(TxID, <Input TxID’s relevant cryptographic signature(s)>)]
}
```
Now, if I change the signature, it no longer affects the hash of the transaction.
 
**Wait, I thought Segwit was about decreasing the size of transactions to effectively increase the blocksize limit?**
Actually, Segwit increases total number of bytes for each transaction (very slightly), but changes the way that block size is calculated so that the signatures don’t count as much against the limit. Essentially, it says that the Bitcoin protocol should pretend that 100kb of signature data only counts as 25kb, as far as the block size limit is concerned. This means that, for every Segwit transaction in a block, the limit is effectively increased by a small amount<sup>[3](#fn3)</sup>.
However, the real benefit of Segwit is that the “malleability fix” enables Lightning Network channels.
 
**The Lightning Network**
The premise of the Lightning Network is that it is possible for two parties to promise money to one another without committing any transactions to the blockchain, but also without trusting one another. This is a significant gain in efficiency: without it, every transaction has to be validated by each of the thousands of bitcoin nodes. With it, only two transactions need to be committed to the blockchain per channel: one to open the channel, and one to settle the final account.
A “channel” acts like a sort of escrow. That is, in order to participate in the Lightning Network, one needs to reserve whatever funds for however long the channel is open. While the channel is open, funds can be sent via the Lightning Network, but cannot be used for any other purpose on the blockchain.
To understand how channels work, we’ll have to understand a couple new kinds of transactions:
Timelock: a transaction that is invalid before some timestamp TS
Multisig: a transaction that requires multiple signatures to be valid. Effectively allows a transaction output to have shared ownership.
 
Let’s say Alice and Bob want to transact with each other multiple times without using the blockchain for every transaction. Here’s how it would work:
Alice creates a multisig transaction sending 100btc to a shared Alice-Bob output--one that requires both signatures--and signs it. We’ll call this transaction Tx0. And she only reveals Tx0 to Bob after:
Alice and Bob each sign a timelocked transaction for 30 days from now, sending 100btc back to Alice. This is not committed to the blockchain, and opens a “channel” between them.
With this setup, Alice has the ability to close the channel in 30 days, by committing the refund transaction. If nothing else happens, Alice will complete the channel having only paid the transaction fees for two transactions.
Now Alice wants to send 50btc to Bob without committing anything to the blockchain. To do so, she signs a contract with the outputs [(Bob, 50), (Alice: 50)], and timelocks it for 29 days.
The impetus is now on Bob to sign Alice’s transaction and commit it to the blockchain when 29 days have passed. 
If Bob commits this transaction, he will receive 50btc and Alice’s 30-day refund will become invalid. If he fails to do so, Alice can close the channel on day 30 and her btc will be refunded in full. Notice Alice’s transaction only has to be validated by Bob, and not by every node in the Bitcoin network.
Alice sends another 10btc to Bob. To do this, she signs a contract with the outputs [(Bob, 60), (Alice, 40)]. Again, she timelocks it at 29 days.
Alice doesn’t need an earlier timelock in this case, because the Bob still has the incentive to commit the transaction.
Bob sends 10btc back to Alice. He signs a contract with the outputs [(Bob, 50), (Alice, 50)], with a timelock of 28 days.
Here the timelock has to move forward in time, because the change in balance has switched directions. It’s now Alice’s responsibility to commit the 50/50 output on day 28, or risk Bob committing the 60/40 transaction on day 29.
 
With this channel, it’s possible for Alice and Bob to send money back and forth, without ever trusting the other, and without having to commit anything to the blockchain.
 
**Does this scale?**
What I’ve described above only explains how two actors can send funds through one channel. However, this model doesn’t scale: I don’t want to have to escrow funds and open a channel with every account in the Bitcoin network.
For the Lightning Network to be practical, it has to be a network: I need to be able to send funds through a series of channels. That way, I can create off-chain transactions with anyone as long as there is a continuous path of open channels between us.
To explain how this works, suppose that both Alice and Bob are connected to a third party, Carol, who runs a large bitcoin business and thus behaves as a sort of hub. But Alice and Bob do not have a direct channel open between one another. It is possible to set up a similar set of timelocked, multi-sig transactions such that a transfer can be processed via Carol, without any trust required between any of the parties.
The details of how this would work involve an additional transaction feature (hashlocks) are a bit too complicated to describe here, but the gist of it is that Alice proposes a transaction to Carol, and Carol proposes a transaction to Bob that is dependent on the first, both of which require Bob’s approval. Presumably, this complexity will be hidden behind their client software’s abstractions.
 
**How is this related to Segwit?**
In the example above, we’ve created a dependency on an uncommitted transaction: Carol’s transaction with Bob uses her transaction with Alice as input. The problem with this is that it’s vulnerable to a malleability attack: if the transaction is committed with a different, but valid signature (remember that this is possible), its hash will change, and thus its ID. 
And so if the Alice->Carol transaction’s ID changes, then the Carol->Bob transaction is no longer valid: the input transaction doesn’t exist!
With a Segwit transaction, however, this is no longer a risk: changing the signature of the transaction will not change its ID.
 
**How does this impact the blocksize debate?**
It seems that a lot of the Bitcoin community falls into one of two camps: either they fiercely believe in Segwit, or they fervently fight for a blocksize increase.
I suppose that there’s an economic point to be made in favor of Segwit instead of larger blocks: that encouraging the much more energy-efficient person-to-person lightning network is a great gain. However, it seems to me that small blocks are in fact a risk for the lightning network.
This is because of the time-dependent nature of lightning network transactions. If every block is full (as they currently are), then any given transaction runs the risk of not being included in a given block. This can happen indefinitely, as there is no first-in-first-out rule for miners to choose transactions. It’s entirely possible for a Lightning Network channel-closing transaction to sit around in the mempool until the next timelocked transaction becomes committable!
This means that larger blocks are not just a necessary adaptation to the increased traffic Bitcoin has seen. Larger would also decrease risks associated with the Lightning Network!
On the other side of this debate, I just see misaligned incentives: the Lightning Network clearly benefits users, but it also harms miners: miners make a (currently small) portion of their income from transactions, and each transaction that occurs off-chain is a potential loss. Since miners ultimately hold the power to make changes to Bitcoin’s protocol, it’s been very difficult to convince miners to upgrade to Segwit. And the community’s stratification on this issue, and the idea that the two protocol changes are somehow exclusive, has only made this problem worse.
Finally, there’s a hard- vs soft-fork argument, but I don’t believe this is a realistic one: both come with risks where, if the mining and user communities do not overwhelmingly support a change, bad things happen. Either way, the community has to come to a consensus.
 
 
 
<a name="fn1">1</a>: Transactions also include timestamps, version numbers, and a couple other fields that are irrelevant to this explanation, and are in a binary format.
<a name="fn2">2</a>: Specifically, for every ECDSA signature (r,s), the signature (r, -s (mod N)) is a valid signature of the same message.
<a name="fn3">3</a>: The amount of effectively added space varies based on the ratio of transaction data to signature data, but you can assume something between a 25 and 50% increase in effective number of transactions for blocks in a typical case.
