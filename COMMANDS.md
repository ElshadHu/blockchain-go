# Commands

## Build and checks

```bash
go build -o blockchain .
go vet ./...
gofmt -l .
```

## Single node

```bash
export NODE_ID=3000
rm -f blockchain_3000.db wallet_3000.dat

./blockchain createwallet
./blockchain createwallet
./blockchain createwallet
./blockchain listaddresses

ADDRS=($(./blockchain listaddresses))
A=${ADDRS[1]}; B=${ADDRS[2]}; C=${ADDRS[3]}

./blockchain createblockchain -address $A
./blockchain getbalance -address $A     # 10
./blockchain getbalance -address $B     # 0

./blockchain send -from $A -to $B -amount 4 -mine
./blockchain getbalance -address $A     # 16 (-mine pays the 10-coin reward to the sender)
./blockchain getbalance -address $B     # 4

./blockchain send -from $B -to $C -amount 2 -mine
./blockchain getbalance -address $B     # 12
./blockchain getbalance -address $C     # 2

./blockchain reindexutxo                # Done! There are 4 transactions in the UTXO set.
./blockchain printchain | grep -c "PoW: true"    # 3
```

## Expected failures

```bash
./blockchain send -from $A -to $B -amount 9999 -mine        # Not enough funds
./blockchain getbalance -address notAnAddress               # Address is not valid
./blockchain send -from $A -to notAnAddress -amount 1 -mine # Recipient address is not valid
./blockchain createblockchain -address $A                   # Blockchain already exists.
NODE_ID=9999 ./blockchain getbalance -address $A            # No existing blockchain found. Create one first.
env -u NODE_ID ./blockchain printchain                      # NODE_ID env . var is not set!
./blockchain                                                # usage
./blockchain send -from $A -to $B -amount 0                 # flag usage
./blockchain send -from $A -to $B -amount 1                 # localhost:3000 is not available
```

## Multi-node network (experimental)

The networking side was not the focus of this project. It works in the sequence below, but it is a minimal implementation: one hard-coded central node on `localhost:3000`, no peer discovery, no fork handling, and no locking around shared state. Feel free to try it.

```bash
# terminal 1: central node
export NODE_ID=3000
rm -f blockchain_3000.db wallet_3000.dat blockchain_genesis.db
./blockchain createwallet                           # CENTRAL
./blockchain createblockchain -address <CENTRAL>
cp blockchain_3000.db blockchain_genesis.db

# terminal 2: wallet node
export NODE_ID=3001
rm -f blockchain_3001.db wallet_3001.dat
./blockchain createwallet                           # WALLET_1
./blockchain createwallet                           # WALLET_2
./blockchain createwallet                           # WALLET_3

# terminal 1
./blockchain send -from <CENTRAL> -to <WALLET_1> -amount 10 -mine
./blockchain send -from <CENTRAL> -to <WALLET_2> -amount 10 -mine
./blockchain startnode

# terminal 2
cp blockchain_genesis.db blockchain_3001.db
./blockchain startnode                              # Ctrl-C after three "Added block" lines
./blockchain getbalance -address <WALLET_1>         # 10
./blockchain getbalance -address <WALLET_2>         # 10

# terminal 3: miner node
export NODE_ID=3002
rm -f blockchain_3002.db wallet_3002.dat
./blockchain createwallet                           # MINER
cp blockchain_genesis.db blockchain_3002.db
./blockchain startnode -miner <MINER>

# terminal 2
./blockchain send -from <WALLET_1> -to <WALLET_3> -amount 1
./blockchain send -from <WALLET_2> -to <WALLET_3> -amount 1    # terminal 3: New block is mined!
./blockchain startnode                              # Ctrl-C after four "Added block" lines
./blockchain getbalance -address <WALLET_1>         # 9
./blockchain getbalance -address <WALLET_2>         # 9
./blockchain getbalance -address <WALLET_3>         # 2
./blockchain getbalance -address <MINER>            # 10
./blockchain getbalance -address <CENTRAL>          # 10
./blockchain printchain | grep -c "PoW: true"       # 4
```

## Reset

```bash
rm -f blockchain_*.db wallet_*.dat blockchain
```
