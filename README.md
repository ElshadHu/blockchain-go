# blockchain-go

A Bitcoin-style blockchain written from scratch in Go. It has proof-of-work mining, a persistent chain stored in BoltDB, ECDSA wallets with Base58 addresses, signed UTXO transactions, a cached UTXO set, Merkle-tree transaction roots, and a TCP peer-to-peer layer that lets several nodes run on one machine and sync with each other.

Everything lives in a single `main` package. There are no frameworks and no external crypto beyond the Go standard library, `golang.org/x/crypto/ripemd160`, and BoltDB via its maintained fork `go.etcd.io/bbolt`.

## Getting started

Build with `go build -o blockchain .` and set `NODE_ID` before running anything. It selects which database and wallet file a command touches (`blockchain_<NODE_ID>.db`, `wallet_<NODE_ID>.dat`) and doubles as the TCP port when the node is started as a server.

[COMMANDS.md](COMMANDS.md) has the full command sequences: a single-node walkthrough with expected balances, the error cases, and the experimental three-node network setup.

## Commands

| Command | What it does |
|---|---|
| `createwallet` | Generates a new ECDSA key pair and appends it to the wallet file |
| `listaddresses` | Prints every address in the wallet file |
| `createblockchain -address A` | Mines the genesis block and pays its coinbase reward to `A` |
| `getbalance -address A` | Sums the unspent outputs locked to `A` using the UTXO set |
| `send -from A -to B -amount N [-mine]` | Builds and signs a transaction. With `-mine` the sending node mines the block itself, otherwise the transaction is broadcast to the central node |
| `printchain` | Walks the chain from the tip back to genesis and prints every block |
| `reindexutxo` | Rebuilds the UTXO set from the full chain |
| `startnode [-miner A]` | Starts a TCP node on `localhost:<NODE_ID>`. With `-miner`, the node mines blocks from its mempool and pays rewards to `A` |

## How it fits together

```
                        ┌───────────┐
                        │    CLI    │
                        └─────┬─────┘
                              │
        ┌─────────────┬───────┴────────┬──────────────┐
        ▼             ▼                ▼              ▼
  ┌───────────┐ ┌────────────┐ ┌─────────────┐ ┌────────────┐
  │  Wallets  │ │ Blockchain │ │   UTXOSet   │ │   Server   │
  │ wallet_*  │ │  (blocks   │ │ (chainstate │ │TCP :NODE_ID│
  │   .dat    │ │   bucket)  │ │   bucket)   │ │            │
  └─────┬─────┘ └─────┬──────┘ └──────┬──────┘ └─────┬──────┘
        │             │               │              │
        │     ┌───────┴───────┐       │              │
        │     ▼               ▼       │              │
        │ ┌────────┐  ┌──────────────┐│              │
        │ │ Block  │  │ ProofOfWork  ││              │
        │ │ +Merkle│  └──────────────┘│              │
        │ └───┬────┘                  │              │
        │     ▼                       │              │
        └─▶ Transaction ◀─────────────┘ ◀────────────┘
            (sign / verify, inputs, outputs)
```

The `Blockchain` and `UTXOSet` types share one BoltDB file; they just use different buckets. The `Server` holds its own `Blockchain` handle and talks to other nodes over TCP.

## Blocks

```go
type Block struct {
    Timestamp     int64
    Transactions  []*Transaction
    PrevBlockHash []byte
    Hash          []byte
    Nonce         int
    Height        int
}
```

A block's hash is computed over its previous hash, its Merkle root, its timestamp, the difficulty and the nonce. Because the previous hash is part of the input, altering any block changes its hash and therefore invalidates every block after it. The first block in the chain is the genesis block. It is the only block with an empty `PrevBlockHash`, and it is created by `createblockchain`.

```
  genesis            block 1            block 2
┌───────────┐      ┌───────────┐      ┌───────────┐
│ Prev: ""  │◀─────│ Prev: h0  │◀─────│ Prev: h1  │◀─── tip ("l" key)
│ Hash: h0  │      │ Hash: h1  │      │ Hash: h2  │
│ Height: 0 │      │ Height: 1 │      │ Height: 2 │
└───────────┘      └───────────┘      └───────────┘
```

## Proof-of-work

Mining is the Hashcash scheme Bitcoin uses. The miner concatenates the block header fields with a nonce, hashes the result with SHA-256, and checks whether the hash, read as a 256-bit integer, is below a target. If it isn't, the nonce is incremented and the hash is recomputed.

```
target = 1 << (256 - targetBits)        // targetBits = 16 in proofofwork.go

nonce = 0
loop:
    hash = SHA256(prevHash ‖ merkleRoot ‖ timestamp ‖ targetBits ‖ nonce)
    if int(hash) < target: done
    nonce++
```

`targetBits` fixes the number of leading zero bits a valid hash must have. A larger value makes the search exponentially harder. Validation is cheap: any node can recompute the hash once and compare it with the target. `printchain` prints `PoW: true` for every block it validates.

## Hashing properties the design relies on

1. A hash cannot be reversed into its input. Hashing is not encryption.
2. The same input always produces the same hash.
3. Changing a single byte of the input produces an unrelated hash.

These three properties are what make the back-linked chain tamper-evident and what make proof-of-work a fair lottery.

## Persistence

Storage is BoltDB, a single-file key/value store written in Go that needs no server process. Values are plain byte slices, so structs are serialized with `encoding/gob` before being written and decoded after being read.

Two buckets are used in `blockchain_<NODE_ID>.db`:

```
blocks bucket
  <32-byte block hash>  ->  gob(Block)
  "l"                   ->  hash of the latest block (the tip)

chainstate bucket
  <transaction ID>      ->  gob(TXOutputs)   // only the outputs still unspent
```

The `Blockchain` struct holds just the tip hash and the open database handle:

```go
type Blockchain struct {
    tip []byte
    db  *bolt.DB
}
```

## Transactions

A transaction is a list of inputs and a list of outputs. There are no accounts and no balances. Value is held in outputs, and an input is a reference to an earlier output plus the proof needed to spend it.

```go
type Transaction struct {
    ID   []byte       // SHA-256 of the canonical encoding, before signing
    Vin  []TXInput
    Vout []TXOutput
}

type TXOutput struct {
    Value      int
    PubKeyHash []byte   // who can spend this output
}

type TXInput struct {
    Txid      []byte    // transaction that holds the output being spent
    Vout      int       // index of that output
    Signature []byte
    PubKey    []byte    // raw public key of the spender
}
```

Outputs are indivisible. Spending 4 coins from an output worth 10 creates two new outputs: 4 for the recipient and 6 of change back to the sender.

Instead of Bitcoin's Script language, locking and unlocking are plain methods. An output is locked to a public-key hash. An input unlocks it when the hash of the input's `PubKey` matches the output's `PubKeyHash` and the input's signature verifies.

### Coinbase transactions

Every block starts with a coinbase transaction. It has a single input with an empty `Txid` and `Vout` set to `-1`, references nothing, and creates `subsidy` coins (currently a constant of 10) out of nowhere. This is the miner's reward and the only place new coins enter the system. The genesis block's coinbase is what funds the very first address.


### Lifecycle of a payment

1. The genesis block's coinbase creates the first output, locked to one address.
2. `send` looks up enough unspent outputs for the sender, builds a transaction with inputs referencing them, adds a change output if needed, and signs every input.
3. The transaction either goes straight into a locally mined block (`-mine`) or is sent to the central node, which relays it to the miner.
4. The miner verifies every transaction in its mempool, adds its own coinbase, and mines the block.
5. The new block is announced to the other nodes, which fetch it, verify its proof-of-work, and append it.
6. Each node updates its UTXO set. The outputs of the new transaction can now be spent.

## Wallets and addresses

A wallet is a key pair:

```go
type Wallet struct {
    PrivateKey ecdsa.PrivateKey
    PublicKey  []byte   // X ‖ Y of the public point
}
```

`Wallets` keeps a map of address to wallet and persists it with gob to `wallet_<NODE_ID>.dat`. The private key never leaves that file.

Raw public keys are long and unreadable, so an address is derived from the public key the same way Bitcoin derives one:

```
            PublicKey
                │
                ▼
     RIPEMD160( SHA256( PublicKey ) )
                │
                ▼
            PubKeyHash  (20 bytes)
                │
                ▼
   version ‖ PubKeyHash                    version = 0x00
                │
                ├────────────────────────────┐
                │                            ▼
                │              SHA256( SHA256( version ‖ PubKeyHash ) )
                │                            │
                │                      first 4 bytes = checksum
                │                            │
                ▼                            ▼
   version ‖ PubKeyHash ‖ checksum   (1 + 20 + 4 = 25 bytes)
                │
                ▼
          Base58Encode
                │
                ▼
            Address
```

Decoding reverses the last step, and the checksum lets `ValidateAddress` reject typos and corrupted strings without hitting the database. The 20-byte `PubKeyHash` in the middle is what gets stored in transaction outputs.

Base58 is Base64 with the ambiguous characters removed: no `0`, `O`, `I` or `l`, and no `+` or `/`. That keeps addresses copyable by hand.

## UTXO set

A balance is the sum of the outputs an address has received and not yet spent. Scanning every block to find them gets slower as the chain grows, so the `chainstate` bucket keeps a copy of just the unspent outputs, keyed by transaction ID. `getbalance` and `send` read only that bucket.

- `Reindex` rebuilds the bucket from the whole chain. `reindexutxo` runs it from the command line.
- `Update` applies one new block: it removes the outputs the block spent and adds the ones it created.
- `FindUTXO` returns the unspent outputs for an address. `FindSpendableOutputs` picks enough of them to cover an amount.

## Merkle tree

Instead of hashing all of a block's transactions in one go, each transaction is hashed on its own, then neighbouring hashes are paired and hashed again until a single hash is left. That last hash is the Merkle root, and it is what the proof-of-work covers.

```
                      root = H(H01 ‖ H23)
                     ┌──────────┴──────────┐
              H01 = H(H0 ‖ H1)      H23 = H(H2 ‖ H3)
             ┌──────┴──────┐           ┌──────┴──────┐
          H0 = H(tx0)   H1 = H(tx1)   H2 = H(tx2)   H3 = H(tx3)
```

If there is an odd number of hashes at any level, the last one is paired with itself. Each transaction is hashed from a fixed byte layout of its fields, not from its gob encoding, so every node computes the same root.

The point of the tree is that proving one transaction is in a block takes only the hashes along its path to the root, not the whole block. That is what lets light wallets verify payments without storing the chain.

## Project layout

| File | Contents |
|---|---|
| `block.go` | `Block`, hashing of transactions via the Merkle tree, gob (de)serialization |
| `blockchain.go` | `Blockchain`: create, open, mine, add, find transactions and unspent outputs, sign and verify |
| `blockchain_iterator.go` | Tip-to-genesis iterator over the `blocks` bucket |
| `proofofwork.go` | Hashcash mining and validation |
| `transaction.go` | `Transaction`, coinbase creation, UTXO transaction building, signing and verification |
| `transaction_input.go` / `transaction_output.go` | Input and output types with their lock and unlock methods |
| `utxo_set.go` | The `chainstate` cache |
| `merkle_tree.go` | Merkle tree construction |
| `wallet.go` / `wallets.go` | ECDSA key pairs, address derivation and validation, wallet file persistence |
| `base58.go` | Base58 encode and decode |
| `server.go` | TCP node: message encoding, handlers, mempool, mining loop |
| `cli.go` and `cli_*.go` | Command-line parsing and one file per command |
| `utils.go` | Helpers such as `IntToHex` and `ReverseBytes` |
