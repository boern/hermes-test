# Hermes relayer test

## Introduction

In this repository we will be working with the [Hermes IBC Relayer](https://hermes.informal.systems) in order to transfer fungible tokens ([ics-020](https://github.com/cosmos/ics/tree/master/spec/ics-020-fungible-token-transfer)) between two [Starport](https://github.com/tendermint/starport) custom [Cosmos SDK](https://github.com/cosmos/cosmos-sdk) chains.



## Setup Environment
1. Install docker and docker-compose 
2. Install hermes.   
Please follow the [instructions to install](https://hermes.informal.systems/installation.html) it locally on your mahcine.
## Run the chains
Follow this instructions to run the two chains

### -------------------------------
### Start chain-a
### -------------------------------
Open a terminal prompt:

```
cd /your/path/to/hermes-test

docker-compose up chain-a
```

### Restore key (Alice)

Open another terminal prompt in the same location (chain-a folder)

This command restores a key for a user in `chain-a` that we will be using during the workshop.

```
# into the container
docker exec -it chain-a bash

# restore the key for alice
chain-a keys add alice --recover --home .chain-a

# When prompted for the mnemonic please enter the following words:
oxygen soap solid wave swim dumb piece pass bronze horn bronze sweet acid radio reform clump team behind deer anxiety volcano reform jewel inspire

# query Alice's balance
chain-a --node tcp://localhost:26657 query bank balances $(chain-a --home .chain-a keys --keyring-backend="test" show alice -a)

# output :
balances:
- amount: "1000000"
  denom: coina
- amount: "1000000"
  denom: stake
pagination:
  next_key: null
  total: "0"

# exit container
exit
```


### -------------------------------
### Start chain-b
### -------------------------------

Open another terminal prompt:

```
cd /your/path/to/hermes-test

docker-compose up chain-b
```
### Restore key (Bob)

Open another terminal prompt in the same location (chain-b folder)

This command restores a key for a user in `chain-b` that we will be using during the workshop.

```
# into the container
docker exec -it chain-a bash

# restore the key for alice
chain-b keys add bob --recover --home .chain-b

# When prompted for the mnemonic please enter the following words:
become observe excess guitar wreck planet treat artist oblige depart fix deposit uphold drift sick amount agree frame pilot transfer nerve guitar section tail

# query Bob's balance
chain-b --node tcp://localhost:26557 query bank balances $(chain-b --home .chain-b keys --keyring-backend="test" show bob -a)

# output :
balances:
- amount: "1000000"
  denom: coinb
- amount: "1000000"
  denom: stake
pagination:
  next_key: null
  total: "0"

# exit container
exit
```


## IBC clients
### Configure Keys for relayer
```
cd ./relayer
```
#### Add keys for each chain

These keys will be used by Hermes to sign transactions sent to each chain. The keys need to have a balance on each respective chain in order to pay for the transactions.

```
hermes -c ./config.toml keys add chain-a -f ../chains/chain-a/alice_key.json 
```

```
hermes -c ./config.toml keys add chain-b -f ../chains/chain-b/bob_key.json 
```

### List keys

To ensure the keys were properly added let's list them.

```
hermes -c ./config.toml keys list chain-a
```

```
hermes -c ./config.toml keys list chain-b
```

### Create client

Create a chain-b client on chain-a:

```
hermes -j -c ./config.toml tx raw create-client chain-a chain-b | jq
```

Query the client state

```
hermes -j -c ./config.toml query client state chain-a 07-tendermint-0 | jq
```

Create a chain-a client on chain-b

```
hermes -j -c ./config.toml tx raw create-client chain-b chain-a | jq
```

Query the client state

```
hermes -j -c ./config.toml query client state chain-b 07-tendermint-0 | jq
```

### Update client

Update the chain-b client on chain-a

```
hermes -j -c ./config.toml tx raw update-client chain-a 07-tendermint-0 | jq
```

Query the client state

```
hermes -j -c ./config.toml query client state chain-a 07-tendermint-0 | jq
```

## IBC Connection
In this section we will run all the steps required to establish a connection handshake 

## Connection Handshake (chain-a -> chain-b)

The steps below need to succeed in order to have a connection opened between chain-a and chain-b

### ConnOpenInit

```
hermes -j -c ./config.toml tx raw conn-init chain-a chain-b 07-tendermint-0 07-tendermint-0 | jq
```

### ConnOpenTry

```
hermes -j -c ./config.toml tx raw conn-try chain-b chain-a 07-tendermint-0 07-tendermint-0 -s connection-0 | jq
```

### ConnOpenAck

```
hermes -j -c ./config.toml tx raw conn-ack chain-a chain-b 07-tendermint-0 07-tendermint-0 -d connection-0 -s connection-0 | jq
```

### ConnOpenConfirm

```
hermes -j -c ./config.toml tx raw conn-confirm chain-b chain-a 07-tendermint-0 07-tendermint-0 -d connection-0 -s connection-0 | jq
```

### Query connection

The commands below allow you to query the connection state on each chain.

```
hermes -j -c ./config.toml query connection end chain-a connection-0 | jq
```

```
hermes -j -c ./config.toml query connection end chain-b connection-0 | jq
```

## IBC Channel 
In this step, we will establish a channel between the chains

### Channel Handshake

The steps below need to succeed in order to have a channel opened between chain-a and chain-b

### ChanOpenInit

```
hermes -j -c ./config.toml tx raw chan-open-init chain-a chain-b connection-0 transfer transfer -o UNORDERED | jq
```

### ChanOpenTry

```
hermes -j -c ./config.toml tx raw chan-open-try chain-b chain-a connection-0 transfer transfer -s channel-0 | jq
```

### ChanOpenAck

```
hermes -j -c ./config.toml tx raw chan-open-ack chain-a chain-b connection-0 transfer transfer -d channel-0 -s channel-0 | jq
```

### ChanOpenConfirm

```
hermes -j -c ./config.toml tx raw chan-open-confirm chain-b chain-a connection-0 transfer transfer -d channel-0 -s channel-0 | jq
```

### Query channel

Use these commands to query the channel end on each chain

```
hermes -j -c ./config.toml query channel end chain-a transfer channel-0 | jq
```

```
hermes -j -c ./config.toml query channel end chain-b transfer channel-0 | jq
```

## IBC Relay Packets (Transfer Tokens)  
First let's query the balance for Alice and Bob to view how much tokens each have.

In the terminal prompt for each chain run the commands below

### Query balance - Alice

```
docker exec -it chain-a bash

chain-a --node tcp://localhost:26657 query bank balances $(chain-a --home .chain-a keys --keyring-backend="test" show alice -a)

exit
```

> Please note the amound to `coina` tokens that Alice has. We will check it again later.

### Query balance - Bob

```
docker exec -it chain-b bash
chain-b --node tcp://localhost:26557 query bank balances $(chain-b --home .chain-b keys --keyring-backend="test" show bob -a)
exit

```

> Please note that Bob only has `coinb` tokens (also some `stake` tokens)

### Fungible token transfer

Now we will transfer some tokens

### Send packet

``` 
hermes -j -c ./config.toml tx raw ft-transfer chain-a chain-b transfer channel-0 999 -o 1000 -n 1 -d coina | jq

```

### View response

The response has a data field that the information is encoded. To view it, you can use the command below. Just replace `[DATA]` with the value from the `data` field in the response

```
echo "[DATA]" | xxd -r -p | jq
```

### Query packet commitments

```
hermes -j -c ./config.toml query packet commitments chain-a transfer channel-0 | jq
```

### Query unreceived packets on chain b

```
hermes -j -c ./config.toml query packet unreceived-packets chain-b transfer channel-0 | jq
```

### Send recv_packet to chain b

```
hermes -j -c ./config.toml tx raw packet-recv chain-b chain-a transfer channel-0 | jq
```

### Query unreceived ack on chain a

```
hermes -j -c ./config.toml query packet unreceived-acks chain-a chain-b transfer channel-0 | jq
```

### Send ack to chain a

```
hermes -j -c ./config.toml tx raw packet-ack chain-a chain-b transfer channel-0 | jq
```

### Query balance - Alice

To ensure the tokens were transferred, query Alice balance.

```
docker exec -it chain-a bash
chain-a --node tcp://localhost:26657 query bank balances $(chain-a --home .chain-a keys --keyring-backend="test" show alice -a)
exit

```

### Query balance - Bob

Now we will query Bob's balance to ensure the tokens were transferred.

```
docker exec -it chain-b bash
chain-b --node tcp://localhost:26557 query bank balances $(chain-b --home .chain-b keys --keyring-backend="test" show bob -a)
exit
```

### View denom trace

In the command above to show the balance, the denom is hashed. In order to view the denom trace you can call the API endpoint below. Just replace the `[denom]` value with the value from the `denom` field (the one that starts with `ibc/`)
```
curl http://localhost:1318/ibc/applications/transfer/v1beta1/denom_traces/[denom]
```

## Send tokens back to chain-a

### Tranfer tokens back

Just replace the `[HASH]` with the hash in the `denom` field

```
hermes -j -c ./config.toml tx raw ft-transfer chain-b chain-a transfer channel-0 999 -o 1000 -n 1 -d ibc/[HASH] | jq
```

### Send recv_packet to chain b

```
hermes -j -c ./config.toml tx raw packet-recv chain-a chain-b transfer channel-0 | jq
```

### Send ack to chain b

```
hermes -j -c ./config.toml tx raw packet-ack chain-b chain-a transfer channel-0 | jq
```

### Query balance - Alice

To ensure the tokens were transferred back, query Alice balance again:

```
docker exec chain-a bash
chain-a --node tcp://localhost:26657 query bank balances $(chain-a --home .chain-a keys --keyring-backend="test" show alice -a)
exit
```

### Query balance - Bob

Query Bob's balance to ensure his token balance for the `ibc/[hash]` denom shows `0` balance

```
docker exec chain-b bash
chain-b --node tcp://localhost:26557 query bank balances $(chain-b --home .chain-b keys --keyring-backend="test" show bob -a)
exit
```

## Congratulations

If you successfully executed all the previous steps you should have a better understanding now how IBC prottocol works to connect two chains and transfer some tokens.

## References
* [Interchain Standards](https://github.com/cosmos/ibc)
* [IBC Protocol Website](https://ibcprotocol.org)
* [IBC Modules and Relayer in Rust](https://github.com/informalsystems/ibc-rs)
* [Hermes Documentation](https://hermes.informal.systems)
* [Starport](https://docs.starport.network/)