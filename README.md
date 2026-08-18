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

To add a new block we need an existing block, but there're not blocks in our blockchain! So, in any blockchain, there must be at least one block, and such block, the first in the chain, is called genesis block
