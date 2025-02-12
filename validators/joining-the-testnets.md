---
description: General instructions on how to join the Juno testnet
cover: ../.gitbook/assets/Discord Invite (29).png
coverY: 260
---

# Joining Testnet

## Current testnet

Below is the list of Juno testnets and their current status. You will need to know the version tag for installation of the `junod` binary.

For details of upgrades on the current testnet, as well as syncing, you can [check out the testnets repo, which is the definitive source of truth](https://github.com/CosmosContracts/testnets).

If you get stuck, then please ask on Discord.

| chain-id | Current Github version tag |
| -------- | -------------------------- |
| uni-6    | v27.0.0                    |

## Minimum Hardware Requirements

The minimum recommended hardware requirements for running a validator for the Juno testnets are:

| Requirements                                                                          |
| ------------------------------------------------------------------------------------- |
| <ul><li>16GB RAM</li><li>200GB of disk space</li><li>2 Cores (modern CPU's)</li></ul> |

{% hint style="warning" %}
These specifications are the minimum recommended. As Juno Network is a smart contract platform, it can at times be very demanding on hardware. Low spec validators WILL get stuck on difficult to process blocks.
{% endhint %}

{% hint style="info" %}
Note that the testnets accumulate data as the blockchain continues. This means that you will need to expand your storage as the blockchain database gets larger with time.
{% endhint %}

## junod Installation

To get up and running with the junod binary, please follow the instructions [here](getting-setup.md)

## Configuration of Shell Variables

For this guide, we will be using shell variables. This will enable the use of the client commands verbatim. It is important to remember that shell commands are only valid for the current shell session, and if the shell session is closed, the shell variables will need to be re-defined.

If you want variables to persist for multiple sessions, then set them explicitly in your shell .profile, as you did for the Go environment variables.

To clear a variable binding, use `unset $VARIABLE_NAME` . Shell variables should be named with ALL CAPS.

### Choose a testnet

Set the `CHAIN_ID`:

```bash
CHAIN_ID=uni-6
```

### Set your moniker name

Choose your `<moniker-name>`, this can be any name of your choosing and will identify your validator in the explorer. Set the `MONIKER_NAME`:

```bash
MONIKER_NAME=<moniker-name>

#Example
MONIKER_NAME="Validatron 9000"
```

### **Set persistent peers**

Persistent peers will be required to tell your node where to connect to other nodes and join the network. To retrieve the peers for the chosen testnet:

```bash
#Set the base repo URL for the testnet & retrieve peers
CHAIN_REPO="https://raw.githubusercontent.com/CosmosContracts/testnets/main/$CHAIN_ID" && \
export PEERS="$(curl -s "$CHAIN_REPO/persistent_peers.txt")"

# check it worked
echo $PEERS
```

{% hint style="info" %}
NB: If you are unsure about this, you can ask in discord for the current peers and explicitly set them in `~/.juno/config/config.toml` instead.
{% endhint %}

### Set minimum gas prices

In `$HOME/.juno/config/app.toml`, set gas prices:

```
# note testnet denom
sed -i.bak -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0.0025ujunox\"/" ~/.juno/config/app.toml
```

## Setting up the Node

{% hint style="info" %}
Running a node is different from running a Validator. In order to run a Validator, you must create and sync a node, and then upgrade it to a Validator.
{% endhint %}

These instructions will direct you on how to initialise your node, synchronise to the network and upgrade your node to a validator.

### **Initialize the chain**

```bash
junod init $MONIKER_NAME --chain-id $CHAIN_ID
```

This will generate the following files in `~/.juno/config/`

* `genesis.json`
* `node_key.json`
* `priv_validator_key.json`

{% hint style="info" %}
Note that this means if you jumped ahead and already downloaded the genesis file, this command will replace it and you will get an error when you attempt to start the chain.
{% endhint %}

### Download the genesis file

```
curl https://raw.githubusercontent.com/CosmosContracts/testnets/main/$CHAIN_ID/genesis.json > ~/.juno/config/genesis.json
```

This will replace the genesis file created using `junod init` command with the genesis file for the testnet.

### **Set persistent peers**

Using the peers variable we[ set earlier](joining-the-testnets.md#set-persistent-peers), we can set the `persistent_peers` in `~/.juno/config/config.toml`:

```bash
sed -i.bak -e "s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" ~/.juno/config/config.toml
```

### **Create a local key pair**

Create a new key pair or restore a key for your validator:

```bash
# Create new keypair
junod keys add <key-name>

# Restore existing juno wallet with mnemonic seed phrase.
# You will be prompted to enter mnemonic seed.
junod keys add <key-name> --recover

# Query the keystore for your public address
junod keys show <key-name> -a
```

Replace `<key-name>` with a key name of your choosing.

{% hint style="danger" %}
After creating a new key, the key information and seed phrase will be shown. It is essential to write this seed phrase down and keep it in a safe place. The seed phrase is the only way to restore your keys.
{% endhint %}

### **Get some testnet tokens**

Testnet tokens can be requested from the faucet. To request tokens, visit [HERE](https://faucet.testnet.chaintools.tech/uni-6/) and enter your Juno address in the address field like this: `https://faucet.testnet.chaintools.tech/uni-6/<your_juno_address_here>`.

## Setup cosmovisor

Follow [these](setting-up-cosmovisor.md) instructions to setup cosmovisor and start the node.

## Syncing the node

After starting the junod daemon, the chain will begin to sync to the network. The time to sync to the network will vary depending on your setup, but could take a very long time. To query the status of your node:

```bash
# Query via the RPC (default port: 26657)
curl http://localhost:26657/status | jq .result.sync_info.catching_up
```

If this command returns `true` then your node is still catching up. If it returns `false` then your node has caught up to the network current block and you are safe to proceed to upgrade to a validator node.

{% hint style="info" %}
Validators and sentries can rapidly join the network with state-sync. See instructions for using state-sync [here](joining-the-testnets.md#undefined).
{% endhint %}

## Upgrade to a validator

To upgrade the node to a validator, you will need to submit a `create-validator` transaction:

```bash
junod tx staking create-validator \
  --amount 9000000ujunox \
  --commission-max-change-rate "0.1" \
  --commission-max-rate "0.20" \
  --commission-rate "0.1" \
  --min-self-delegation "1" \
  --details "validators write bios too" \
  --pubkey=$(junod tendermint show-validator) \
  --moniker $MONIKER_NAME \
  --chain-id $CHAIN_ID \
  --gas-prices 0.025ujunox \
  --from <key-name>
```

## Backup critical files

There are certain files that you need to backup to be able to restore your validator if, for some reason, it damaged or lost in some way. Please make a secure backup of the following files located in `~/.juno/config/`:

* `priv_validator_key.json`
* `node_key.json`

It is recommended that you encrypt the backup of these files.
