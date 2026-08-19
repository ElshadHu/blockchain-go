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
