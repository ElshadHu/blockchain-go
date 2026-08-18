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