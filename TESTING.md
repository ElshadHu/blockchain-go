# Testing

```bash
# compile everything
go build ./...

# catch suspicious constructs
go vet ./...

# list misformatted files (empty = good)
gofmt -l .

# wipe stale state (old address format)
rm -f blockchain.db wallet.dat

# build the binary
go build -o blockchain .

# create three wallets
./blockchain createwallet
./blockchain createwallet
./blockchain createwallet

# list them
./blockchain listaddresses

# grab addresses into vars (zsh, 1-indexed)
ADDRS=($(./blockchain listaddresses))
A=${ADDRS[1]}; B=${ADDRS[2]}; C=${ADDRS[3]}

# genesis block, reward to A
./blockchain createblockchain -address $A

# expect 10
./blockchain getbalance -address $A

# expect 0
./blockchain getbalance -address $B

# send A -> B
./blockchain send -from $A -to $B -amount 4

# expect 6
./blockchain getbalance -address $A

# expect 4
./blockchain getbalance -address $B

# spend a received output
./blockchain send -from $B -to $C -amount 2

# expect 2
./blockchain getbalance -address $C

# every block should show PoW: true
./blockchain printchain

# panics: Not enough funds
./blockchain send -from $A -to $B -amount 9999

# panics: Address is not valid
./blockchain getbalance -address notAnAddress

# panics: Recipient address is not valid
./blockchain send -from $A -to notAnAddress -amount 1

# Blockchain already exists
./blockchain createblockchain -address $A

# usage + exit 1
./blockchain

# usage + exit 1
./blockchain bogus

# flag usage + exit 1
./blockchain send -from $A -to $B -amount 0
```
