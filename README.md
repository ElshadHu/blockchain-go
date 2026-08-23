# blockchain-go

When it comes to `Block` entity:

`Timestamp` is the current timestamp (when the block is created)
`Data` is the actual valuable information containing in the block
`Hash` is the hash of the block
`PrevBlockHash` stores the hash of the previous block

## Blockchain

In its essence blockchain is just a database with certain structure: it's an ordered, back-linked list. Which means that blocks are stored in the insertion order and that each block is linked to the previous one. This structure allow to quickly get the latest block in  a chain and to efficiently get a block by its hash.

In Golang this structure can be implemented by using an array and a map:
the array would keep ordered hashes, and the map would keep hash -> block pairs (maps are unordered). But for out blockchain prototype we will just use an array, because we don't need to get blocks by their hash for now.

To add a new block we need an existing block, but there're not blocks in our blockchain! So, in any blockchain, there must be at least one block, and such block, the first in the chain, is called genesis block.

## Proof-of-Work

This whole "do hard work and prove" mechanism is called proof-of-work.
It is hard because it requires a lot of computational power: even high performance computers cannot do it quickly. In Bitcoin the goal of such work is to find a hash for a block. that meets our requirements. And it's this hash that serves as a proof. Thus, finding a proof is the actual work.


## Hashing

1. Original data cannot be restored from a hash. Thus hashing is not enrcyption.
2. Certain data can have only one hash and the hash is unique
3. Changin even one byte in input data will result in a completely different hash

In blockchain, hashing is used to guarantee the consistency of a block. The input data for a hashing algorithm contains the hash of the previous block, thus making it impossible to modify a block in the chain: one hash to recalculate its hash and hashes of all the blocks after it.

Bitcoin uses Hashcash, a Proof-of-Work algorithm that was initially developed to prevent email spam. It can be split into the following steps:

1. Take some publicly known data (in case of email, it's receiver's email address, in case Bitcoin, it's block headers)
2. Add a counter to it. The counter starts at 0.
3. Get a hash of the data + counter combination.
4. Check that the hash meets certain requirements.
1. If it down you're done
2. If it doesn't, increase the counter and repeat the steps 3 and 4.

24 is an arbitrary number, our goal is to have a target that takes less than 256 bits in memory. And we want the difference to be significant enoguh, but not too big, because the bigger the difference the more difficult it's to find a proper hash

## Persistence and CLI

So far, we've built a blockchain with a proof-of-work system, which makes mining possiblem. We need to store a blockchain in a database, after that we will make a simple command-line interface to perform operations with the blockchain.
Blockchain is distributed but now and focus on the database part

## What We Use

We will use BoltDB, because:
1. It's simple and minimalistic.
2. It's implemented in Go.
3. It doesn't require to run a server
4. It allows to build the data structure we want

BoltDB is a key/value storage. Key/Value pairs are stored in buckets, which are  intended to group similar pairs. There are no datatypes and it is just a byte array whcih meains we need to serialize them. We will use encoding/gob because it is simple and is the part of standard Go library. We will need to serialize Go structs , implement a mechanism of converting a Go struct into a byte array and restoring it back from a byte array.

## Database Structure

We will use two buckets to store data:

1. `blocks` stores metadata 
2. `chainstate` store the state of a chain, which is all currently unspent transaction outputs and some metadata

We will store the whole DB as a single file, without storing blocks in separate files. So we won't need anything related to file numbers.:

`32-byte block-hash -> Block structure (serialized)`
`'l' -> the hash of the last block in a chain`

When it comes to BoltDB, operations with a database are run withing a transaction. And there are two types of transactions: 
read only and read-write.

Also we store DB connection, because we want it open it once and keep it open while the program is running. Thus,the Blockchain structure now looks like this:
```go
type Blockchain struct {
    top []byte
    db *bolt.DB
}
```

From now on we need to store blocks in the DB.
All new blocks are now saved in a database, so we can reopen a blockchain and add a new block to it.
But after implementing this, we lost a nice feature:
we cannot print out blockchain blocks anymore because we don't store blocks in an array any longer. Let's fix this flaw!

BoltDB allows to iterate over all the keys in a bucket, but the keys are stored in byte-sorted order, and we want blocks to be printed in the order they take in a blockchain. 
Also, because we don't want to load all the blocks into memory (our blockchain DB could be huge!.. or let's just pretend it could), we'll read them one by one. For this purpose, we'll need a blockchain iterator

## CLI

Until now our implementation hasn't provided any interface to interact with the program: we've simply executed `NewBlockChain`,`bc.AddBlock`

We want to have these commands:
`blockchain_go addblock "Pat 0.0333 for a coffee"`
`blockchain_go printchain`

All command-line related operations will be processed by the `CLI` struct
```go
type CLI struct {
    bc *Blockchain
}
```

## Transactions

Transactions are the heart of Bitcoin and the only purpose of blockchain is to store transactions in a secure and reliable way, so no would modify them after they are created.

If you have developed a web app, in order to implement payments you would likely to create `accounts` and `transactions` tables.
In Bitcoin, payments are realized in completely different way. There are:
1. No accounts.
2. No balances.
3. No addresses.
4. No coins.
5. No senders and receivers

Since blockchain is a public and open db, we don't want to store sensitive data about wallet owners.

### Bitcoin Transaction

A transaction is a combination of inputs and outputs

```go
type Transaction struct {
    ID []byte
    Vin []TXInput
    Vout []TXOutput
}
```
Inputs of a new transaction reference outputs of a prev transaction.(there is an exception though which will be discusssed later).
Outputs are where coints are actually stored.

`ScriptPubKey` will store an arbitrary string(user defined wallet address).
One important thing about outputs is that they are **indivisible**.

Here is the input of the transaction

```go
type TXInput struct {
Txid []byte
Vout int
ScriptSig string
}
```
`ScriptSig` is a script which provides data to be used in an output's `ScriptPubKey`. 
If the data is correct the output can be unlocked, and its value can be used to generate new outputs; if it's not correct, the output cannot be referenced in the input. This is the mechanism that guarantees that users cannot spend coins belonging to other people.

We will implement keys and signatures checking in the next one, currently it `ScriptSig` will be store an arbitrary user defined wallet address.

Let's sum it up. Outputs are where "coins" are stored. Each output comes with an unclocking script, which determines the logic of unlocking the output.
Every new transaction must have at least one input one output. An input references an output from a previous transaction and provides data (the `ScriptSig` field) that is used in the output's unlocking script to unlock and use its value to create new outputs.

But what came first: inputs or outputs?

# The egg

In Bitcoin, it's the egg that came before the chicken. The inputs-referencing-outputs logic is the classical "chicke or the egg" situation:
inputs produce outputs and outputs make inputs possible. And in Bitcoin, outputs come before inputs.

When a miner starts mining a block, it adds a coinbase transaction to it. A `coinbase transaction` is a special type of transactions, which doesn't require previously existing outputs. It creates outputs ("coins") out of nowhere
The egg without a chicken. This is the reward miners get for mining new blocks.

As you know, there’s the genesis block in the beginning of a blockchain. It’s this block that generates the very first output in the blockchain. And no previous outputs are required since there are no previous transactions and no such outputs.

Coinbase transaction has only one input. In our implementation its Txid is empty and `Vout` equals to -1. Also a coinbase transaction doesn't store a script in `ScriptSig`. Instead an arbitrary data is stored there.

`subsidy` is the amount of reward. In Bitcoin, this number is not stored anywhere and calculated based only on the total number blocks:
the number of blocks divided by `210000`. Mining the genesis block produced 50 BTC, and every 210000 blocks the reward is halved. In our implementation, we will store the reward as a constanst (at least for now)

### Storing Transactions In Blockchain

From now on, every block must store at least one transaction and it's no more possible to mine blocks without transactions. This means that we should remove the `Data` field of `Block` and store transactions instead

Again, we are using hashing as a mechanism of providing unique representation of data. We want all transactions in a block to be uniquely identified by single hash

## Unspent Transaction Outputs

Wen eed to find all unspent transaction outputs. Unspent means that these outputs weren't referenced in any inputs.

Of course, when we check balance, we don't need all of them, but only those can be unblocked by the key we own (currently we don't have keys implemented and will use user defined addresses instead). First let's define locking-unlocking methods on inputs and outputs.

Since transactions are stored in blocks, we have to check every block in a blockchain. We start with outputs:

```go
if out.CanBeUnlockedWith(address) {
	unspentTXs = append(unspentTXs, tx)
}
```

If an output was locked by the same address we're searching unspent transaction outputs for, then this is the output we want. But before taking it, we need to check if an output was already referenced in an input:

```go
if spentTXOs[txID] != nil {
    for _, spentOut := range spentTXOs[txID] {
        if spentOut == outIdx {
            continue Outputs
        }
    }
}
```

## Public-key Cryptography

Public-key cryptography algorithms use pairs of keys: public and private keys. Public keys are not sensitive and can be disclosed to anyone.

Private and public keys are just random sequences of bytes, thus they cannot be printed on the screen and read by a human. That’s why Bitcoin uses an algorithm to convert public keys into a human readable string.


## Digital Signatures

In maths and cryptography there is a concept of digital signature - algorithms that guarantee:
1. that data wasn't modified while being transferred from a sender to a recipient
2. that data was created by a certain sender
3. that the sender cannot deny sending the data

Digital signing happens with the usage of private key, and verification requires a public key.

 In order to sign we need two things:
 1. data to sign;
 2. private key;

 The operation of signing produces a signature, which is stored in transaction inputs. In order to verify a signature, the following is required:

 1. data that was signed
 2. the signature
 3. public key


 Digital signatures are not encryption, you cannot reconstruct the data from a signature. This is similar to hashing: you run data through a hashing algorithm and get a unique representation of the data. The difference between signatures and hashes is key pairs: they make signature verification possible.
But key pairs can also be used to encrypt data: a private key is used to encrypt, and a public key is used to decrypt the data. Bitcoin doesn’t use encryption algorithms though.

Every transaction input in Bitcoin is signed by the one who created transaction. Every transaction must be verified before being put in a block. Verification means:
1. Checking that inputs have permission to use outputs from previous transactions
2. Checking that the transaction signature is correct.

Schematically it loos like this

                    Signing
    Private Key  + Data  = Signature
    key/pair        Copy     Copy
                    Verification
    Public Key  + Data   = Signature


Let's now review the full lifecycle of the transaction:
1. In the beginning, there is the genesis block that contains a coinbase transaction. there are no real inputs in coinbase transactions, so signing is not necessary.
The output of the coinbase transaction contains a hashed public key
2. When one sends coins, a transaction is created. Inputs of the transaction will reference outpus from prev transactions. Every input will store a public key (not hashed) and a signature of the whole transaction
3. Other nodes in the Bitcoin netwoek that receive the transaction will verify it. Besides other thing, they will check that:
the hash of the public key in an input matchs the hash of the referenced output (this ensures that the sender spends only coins belonging to them); the signature is correct.
4. When a miner node is ready to mine a new block, it'll put the transaction in a block and start mining it
5. When the block is mined, every other node in the network receives a message saying the block is mined and adds the block to the blockchain.
5. After the block is added to the block chain, the transaction is completed, its outputs can be referenced in new transactions.

## Elliptic Curve Cryptography

Public and private keys are sequences of random bytes.
Since it's private keys are used to identify owners of coins, there's a required condition: the randomness algorithm must produce truly random bytes. We don't want to accidentally generate a private key that's owned by someone else. What we need to know is that these curves can be used to generate really big and random numbers

## Base58

Bitcoinused the Base58 algorithm to convert public keys into human readable format. The algorithm is very similar
to a famous Base64, but it uses shorter alphabet: some letters were removed from the alphabet to avoid some attacks that use letters similarity.
Thus, there are no these symbols: 0 (zero), O (capital o), I (capital i), l (lowercase L), because they look similar. Also, there are no + and / symbols.


Thus, the above mentioned decoded public key consists of three parts:

Version  Public key hash                           Checksum
00       62E907B15CBF27D5425399EBF6F0FB50EBB88F18  C29B7D93    

Overall structure for base58 is 

                Public Key
                  |
            RIPEMD160(SHA256(PubKey))
                        |            |
    Version+            Public key has      +        SHA256 (SHA256(PubKeyHash))

                    |
                    Base58Encode
                    |
                    BitcoinAddress


## Wallet

A wallet is nothing but a key pair. We'll also need the Wallets type to keep a collection of wallets, save them to file, and load them from it. in the construction function os `Wallet` a new key pair is generated.

Heare are the steps to convert a public key into Base58 address:
1. Take the public key and hash it twice with RIPEMD(SHA256(PubKey)) hashing algorithms
2. Prepend the version of the address generation algo to the hash
3. Calculate the checksum by hashing the result of step 2 with SHA256(SHA256(payload)). The checksum is the first four bytes of the resulted hash
4. Append the checksum to the `version + PubKeyHash` combination
4. Encode the `version + PubKeyHash + checksum` combination with Base58

Notice, we no longer use `ScriptPubKey` and `ScriptSig` fields, because we are not going to implement a scripting language. . Instead, ScriptSig is split into Signature and PubKey fields, and ScriptPubKey is renamed to PubKeyHash. We’ll implement the same outputs locking/unlocking and inputs signing logics as in Bitcoin, but we’ll do this in methods instead.



### The UTXO Set

we told the way Bitcoin Core stores blocks in a database. It was said that blocks are stored in blocks and db and transaction outputs are stored in blocks db and transaction outputs are stored in chainstate.

Since that article, we’ve already implemented transactions, but we haven’t used the chainstate to store their outputs. So, this is what we’re going to do now.

chainstate doesn’t store transactions. Instead, it stores what is called the UTXO set, or the set of unspent transaction outputs. Besides this, it stores “the block hash up to which the database represents the unspent transaction outputs”, which we’ll omit for now because we’re not using block heights (but we’ll implement them in next articles).

The function finds transactions with unspent outputs. Since transactions are stored in blocks, it iterates over each block in the blockchain and checks every transaction in it. As of September 18, 2017, there’re 485,860 blocks in Bitcoin and the whole database takes 140+ Gb of disk space. This means that one has to run a full node to validate transactions. Moreover, validating transactions would require iterating over many blocks.

The solution to the problem is to have an index that stores only unspent outputs, and this is what UTXO set does: this is a cache that is built from all blockchain transactions (by iterating over blocks,yes, but this is done only once), and is later used to calculate balance and validate new transactions. The UTXO set is about 2.7 GB as of sep of 2017.

Blockchain.FindUnspentTransactions – the main function that finds transactions with unspent outputs. It’s this function where the iteration of all blocks happens.
Blockchain.FindSpendableOutputs – this function is used when a new transaction is created. If finds the enough number of outputs holding required amount. Uses Blockchain.FindUnspentTransactions.
Blockchain.FindUTXO – finds unspent outputs for a public key hash, used to get balance. Uses Blockchain.FindUnspentTransactions.
Blockchain.FindTransaction – finds a transaction in the blockchain by its ID. It iterates over all blocks until finds it.

So what we want is the following methods:

1. `Blockchain.FindUTXO` - find all unspent outputs by iterating over blocks
2. `UTXOSet.Reindex` - uses `FindUTXO` to find unspent outputs, and stores them in a db. This where caching happens.
3. `UTXOSet.FindSpendableOutputs` - – analog of Blockchain.FindSpendableOutputs, but uses the UTXO set.
4. UTXOSet.FindUTXO - analog of `Blockchain.FindUTXO` but uses the UTXO set.
5. `Blockchain.FindTransaction` remains the same.

## Merkle Tree

There is one more optimization mechanism I'd like to discuss in this post.

As it was said above, the full Bitcoin database (blockchain) takes more than 140 GB of disk space. Because of the decentralized nature of Bitcoin, every node in the network must be independent and self-sufficient, every node must store a full copy of the blockchain. With many people starting using Bitcoin, this rule becomes more difficult to follow: it's not likely that everyone will run a full node. Also since nodes are full-fledged participants of the network, they have responsibilities: they must verify transactions and blocks. Also there's certain internet traffic required to interact with other nodes and download new blocks.

In the original Bitcoin paper, there is a solution for this: Simplified Payment Verification (SPV). SPW is a light Bitcoin node that doesn't download the whole blockchain and doesn't verify blocks and transactions.
Instead, it finds transactions in blocks (to verify payments) and is linked to a full node to retrieve just necessary data.This mechanism allows having multiple light wallet nodes with running just one full node.

For SPV to be possible, there should be a way to check if a block contains certain transaction without downloading the whole block. And this is where Merkle tree comes into play.

Merkle trees are used by Bitcoin to obtain transactions hash, which is then saved in block headers and is considered by the proof-ow-work system. Until now, we just concatenated hashes of each transaction in a block and applied SHA256 to them. This is also a good way of getting a unique represatation.This is also a good way of getting a unique representation of block transactions, but it doesn’t have benefits of Merkle trees.

The number of the leaves in the merkle tree must be even. If it is odd, in that case there the last transaction is duplicated (in the merkle tree, not in the block)

Moving from the bottom up, leaves are grouped in pairs, their hashes are concatenated and a new hash is obtained from the concatenated hashes. The new hashes form new tree nodes. This process is repeated until there's just one node, which is called the root of the tree. The root hash is then used as the unique representation of the transactions, is saced in block headers, and is used in the proof-of-work system

The benefit of Merkle trees is that a node can verify membership of certain transaction without downloading the whole block. Just a transaction hash, a Merkle tree root hash, and merkle path are required for this.
