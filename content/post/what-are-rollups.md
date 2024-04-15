+++
categories = ["Ethereum", "Blockchain", "web3"]
date = 2022-04-11T07:45:03Z
description = ""
draft = false
image = "https://images.unsplash.com/photo-1638818834413-c3769cfdd561?crop=entropy&cs=tinysrgb&fit=max&fm=jpg&ixid=MnwxMTc3M3wwfDF8c2VhcmNofDIwfHxldGhlcmV1bXxlbnwwfHx8fDE2NDk2NjMwMjM&ixlib=rb-1.2.1&q=80&w=2000"
slug = "what-are-rollups"
summary = "Rollups offer faster and cheaper transactions for dApp developers and their customers. This post summarizes my research on rollups, and a few things dApp developers should know when picking the right blockchain protocol."
tags = ["Ethereum", "Blockchain", "web3"]
title = "What are Rollups in Blockchain Networks 🍣"

+++


Increasing the transactional throughput of public blockchains is a key focus for blockchain researchers today. [EIP-4844](https://eips.ethereum.org/EIPS/eip-4844) just came out, and it's old news that rollups will play a huge role in the future of scaling Ethereum. Ethereum's co-founder Vitalik Buterin described the concept of rollups back in 2014. Last year, Vitalik [claimed rollups to be the “only choice" for making gas fees more affordable](https://vitalik.ca/general/2021/01/05/rollup.html).

Rollups offer faster and cheaper transactions for dApp developers and their customers. This post summarizes my research on rollups, and a few things dApp developers should know when picking the right blockchain protocol.

{{< figure src="https://res.craft.do/user/full/9d54cc03-adfe-f72f-3389-565eb7356d1d/doc/1C445B41-B14F-4E8F-9208-E6D7E9565D52/86D1780C-B4FD-4AF8-BF0B-256F24627357_2/AxrzzGRiNYaxyhbjmnipvXDP38mulKJn0G1WKVtYy6Iz/Image.jpeg" >}}

## Why do we need rollups? 🛼

> Rollups are in the short and medium-term, and possibly in the long term, the only trustless scaling solution for Ethereum.

Thanks Vitalik for that intro. If you’re familiar with Ethereum, you also know about the current gas prices crises. Basically, if you transfer coins on Ethereum, be prepared to fork over up to $10 in fees to send $1. The last sentence may not be an exaggeration. 😔

Rollups are a layer 2 scaling solution to drastically reduce the gas price on the Ethereum mainnet. Rollups are supposed to provide a way to reduce the costs and latency of decentralized applications (dApps) for users and developers.

In layer 2 scaling solutions, web3 apps send transactions to nodes that are part of the layer 2 network, then the network batches transactions into groups before _anchoring (publishing)_ them to layer 1, after which they are secured by layer 1 since they are publicly verifiable and cannot be altered. Thus rollups offer faster execution by executing transactions off-chain and publishing the proof of transactions on-chain.

> Rollups move computation (and state storage) off-chain, but keep some data per transaction on-chain.

A rolledup transaction could include tens of thousands of transactions, which means tens of thousands of transactions can be recorded on the mainchain for the price of one. Using compression algorithms, the more layer 2 transactions you can bundle in a single layer 1 transaction, the cheaper it is to store proof of transactions.

> [Jag Sidhu](https://jsidhu.medium.com/the-ultimate-guide-to-rollups-f8c075571770) writes “some Ethereum engineers got these individual account updates down to a few bytes (8–12 bytes depending on the implementation) which means that a block with 1 megabyte of bandwidth would be able to roughly process 83k — 125k account adjustments per block and around 5500 to 8300 TPS theoretically assuming 15 second block times.”

{{< figure src="https://res.craft.do/user/full/9d54cc03-adfe-f72f-3389-565eb7356d1d/doc/1C445B41-B14F-4E8F-9208-E6D7E9565D52/3B62E811-91E8-408B-9B3C-574AA058D50D_2/90FGxI1BElqAd7GnK8ViyaIAo4JYzzIgsNZkhbmqXKwz/Image.jpeg" >}}

[Image source](https://wwz.unibas.ch/fileadmin/user_upload/wwz/00_Professuren/Schaer_DLTFintech/Lehre/Tobias_Schaffner_Masterthesis.pdf)

## Types of rollups

The paper subtly categorizes rollups into two prominent categories: Arbitrum and Optimism with fees that are ~3-8x lower gas fees than L1 and ZK-rollups, with ~40-100x lower gas fees than Ethereum mainnet.

So, what’s the difference between Arbitrum and Optimism that provide single-digit gains than ZK-rollups with triple-digit gains? That’s because there are two types of layer 2 scaling solutions: Optimistic and ZK.

### Optimistic rollups

[Arbitrium](https://offchainlabs.com) and [Optimism](https://www.optimism.io) are layer 2 protocols that use “optimistic rollup” (OR) to scale Ethereum. An optimistic rollup network assumes that transactions are valid by default and only performs calculations, via a fraud proof, in the event of a challenge.

In other words, when an application transacts on an optimistic rollup network like Arbitrum, the actual transfer of funds (from _accountA_ to _accountB_) happens on the Arbitrum. The transaction is then published on Ethereum mainnet.

Remember that an optimistic rollup network assumes all transactions are valid, at least initially. So what happens if a transaction in invalid?

This is indeed a problem with optimistic rollups. Because, every transaction is assumed valid, optimistic rollups have a _withdrawal time (7-14 days)_ constraints while the network waits for _someone else_ to challenge the state of the network.

Optimistic Rollups rely on fraud proofs to avoid re-computations. The state is proposed to Ethereum by a “bonded” actor. Anyone who wants to challenge the actor may claim a bounty by proving that the state update is inaccurate. To accomplish this, the challenger must provide the data required by the smart contract to prove the inaccuracy. [This thread](https://threadreaderapp.com/thread/1395812308451004419.html) goes over the key difference between Optimism and Arbitrum fraud proof mechanism.

ZK-rollups don’t have the withdrawal time constraint because they include a validity proof.

### ZK Rollups

For Ethereum — and EVM compatible chains — to become world’s next distributed computing platform, gas prices have to be massively reduced, until it is cheaper to do things at internet scale. ZK rollups (ZKR) promise could be the key to achieving that level of scalability.

ZK rollups like [zkSync](https://zksync.io) are popular because they don’t have the _withdrawal time_ problem that optimistic rollups do. Withdrawal times in zkSync, a ZK rollup live on Ethereum mainnet, are 10 minutes to 7 hours during low usage. Moreover, ZK rollups gets cheaper and faster as the usage increases, so in the future things will become faster.

## But, what does ZK stand for?

ZK rollups are based on the concept of provers and verifiers. ZK stands for Zero Knowledge.

ZKR "roll-up" off-chain transactions and generate a cryptographic proof known as a zk-SNARK. The acronym [zk-SNARK](https://en.wikipedia.org/wiki/Non-interactive_zero-knowledge_proof) stands for “Zero-Knowledge Succinct Non-Interactive Argument of Knowledge". The zk-SNARK is the proof of validity of the transactions in the form of a hash and is eventually placed on the main chain.

A special ZK Rollup smart contract, which resides on Layer 1, maintains the status of the transfers made on rollup chain. The status can only be updated with a validity card; the zk-SNARK. The zk-SNARK is a hash that represents the blockchain's validity status.

“Zero-knowledge” proofs allow one party (the prover) to prove to another (the verifier) that a statement is true without revealing any information beyond the validity of the statement itself. For example, given the hash of a random number, the prover could convince the verifier that there indeed exists a number with this hash value, while disclosing what that random number is.

> In a zero-knowledge “Proof of Knowledge” the prover can convince the verifier not only that the number exists, but that they in fact know such a number – again, without revealing any information about the number.

zk-SNARK’s succinct proofs are only a few hundred bytes and can be verified within a few milliseconds. The ZK proof mathematically proves that no fraud has occurred.

[zkSync](https://zksync.io) is a ZKR live on Ethereum mainnet. [Immutable X](https://www.immutable.com) and [Loopring](https://loopring.io/#/) also use ZKR. [Zcash](https://z.cash) is the first widespread application of zk-SNARKs. Polygon is focused on Zero-Knowledge (ZK) cryptography as the end game for blockchain scaling. There's a lot of innovation happening in this space. [L2beat.com](https://l2beat.com) provides details about Ethereum layer 2 scaling solutions.

### ZKR core components

ZKRs execute transactions on sidechain and roll them on the mainchain. ZKR use two transactors and relayers to achieve this.

* **Transactor** create and broadcast transaction data (indexed address, value, network fee, and nonce) to the network. Transactor corresponds to an external account on Ethereum. Smart contracts then record addresses to one Merkle Tree and the transaction value to another.
* **Relayers** collect a large number of transactions creating rollups. Relayers generate the ZK proof that creates the blockchain _state_ before and after each transaction. The resulting changes reach the mainchain in a verifiable hash. Although anyone can become a relayer, you must first stake their cryptocurrency in a smart contract to ensure honesty.

> This “**state**” is essentially a database which represents new balances and adjustments to accounts as users transact with their accounts inside of the rollup

### How do rollups reduce gas?

Rollups don’t actually reduce the gas on Ethereum. Recall that a rollup is a layer 2 sidechain; when using a rollup, you won’t be sending transactions on Ethereum mainnet; instead transactions will be submitted to the L2.

Users of a dApp running the ZK-Rollup scheme will pay less in transaction fees.

## Are Optimistic Rollups a temporary solution?

This seems to be a common question in the community. If ZKR are faster, then why even bother with OR?

Optimistic rollups have a first-mover advantage. First of all, the main reason why OR were more popular in the past was because until recently ZKRs didn't support Solidity smart contracts. ZKRs have to generate validation proofs, and the earliest iterations were not EVM and Solidity compatible. That changed in 2021. Now you can take your Solidity smart contract and deploy it on a ZKR with a few (relatively minor) changes.

On Feb 2022, zkSync 2.0 became available on Ethereum’s testnet. [zkEVM](https://docs.zksync.io/zkevm/) is a virtual machine that executes smart contracts in a way that is compatible with zero-knowledge-proof computation.

Only time will tell who wins.

### How will the merge affect this?

Simply put, it will not. I’ll provide a detailed answer in another post.

## Topics we skipped

In optimistic rollups, when transactions are ready to be rolled up, a **sequencer** is a specially designated full node that can control the ordering of transactions. Sequencers bundle transactions and submit both the transaction data and the new L2 state root to L1. Kyle Charbonnet has explained Optimism's optimistic rollup implementation in detail [here](https://medium.com/privacy-scaling-explorations/an-introduction-to-optimisms-optimistic-rollup-8450f22629e8).

[**ZK-STARK** (Zero-Knowledge Scalable Transparent ARguments of Knowledge)](https://starkware.co/stark/). The proof system used in ZK-SNARK requires a trusted party, or parties, to initially set up the ZK proof system. A dishonest trusted party could compromise the privacy of the system. ZK-STARKS improve on this technology by removing the need for a trusted setup.

## Conclusion

Blockchain is a fast-moving space. Millions of dollars continue to be funneled into building scalable future blockchain networks. It’s hard to tell if ZKRs will be the silver bullet to address Ethereum’s data availability and scaling problems. In the short term, it does look like ZKRs are a step in the right direction.

---

### References

[DeFi Explained: ZK Rollups](https://www.reddit.com/r/CryptoCurrency/comments/nctot7/defi_explained_zk_rollups/)

[What are ZK-Rollups and why they're the best investment you can make in 2022.](https://www.reddit.com/r/CryptoCurrency/comments/rvktc1/what_are_zkrollups_and_why_theyre_the_best/)

[The Ultimate Guide to Rollups](https://jsidhu.medium.com/the-ultimate-guide-to-rollups-f8c075571770)

[February 20, 2021: Optimistic vs ZK-Rollups, ELI5 🧒🧑‍🏫](https://curve.substack.com/p/february-20-2021-optimistic-vs-zk?s=r)

[Rollups – The Ultimate Ethereum Scaling Solution – Finematics](https://finematics.com/rollups-explained/)

[An Incomplete Guide to Rollups](https://vitalik.ca/general/2021/01/05/rollup.html)

[How does Optimism’s Rollup really work?](https://research.paradigm.xyz/optimism)

[What are zk-SNARKs? | Zcash](https://z.cash/technology/zksnarks/)

* [Scaling Public Blockchains](https://wwz.unibas.ch/fileadmin/user_upload/wwz/00_Professuren/Schaer_DLTFintech/Lehre/Tobias_Schaffner_Masterthesis.pdf)[An Introduction to Optimism’s Optimistic Rollup](https://medium.com/privacy-scaling-explorations/an-introduction-to-optimisms-optimistic-rollup-8450f22629e8)

