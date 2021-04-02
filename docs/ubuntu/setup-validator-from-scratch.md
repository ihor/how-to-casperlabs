# Setup a validator node from scratch on Ubuntu 20.04

> **Note**  
> Do not execute all the commands below as root. sudo is included where it is required. 

## Set version and network you're going to set up

Set a variable defining the version of the node package you're setting up. For `1.0.0`, use `1_0_0`

```CASPER_VERSION=1_0_0```

Set a variable defining the network name you're trying to set up. For example, for Main Net, use `casper`, while for Test Net use `testnet`

```CASPER_NETWORK=casper```


## Install software

### Install pre-requisites

```
sudo apt install dnsutils
```

The node uses ```dig``` to get external IP for autoconfig during the installation process

### Install helpers

```
sudo apt install jq
```

We will use ```jq``` to process JSON responses from API later in the process

### Remove Previous Versions

If you were running previous versions of the casper-node on this machine, first stop and remove the old versions:

```
sudo systemctl stop casper-node-launcher.service
sudo apt remove -y casper-client
sudo apt remove -y casper-node-launcher
sudo rm /etc/casper/casper-node-launcher-state.toml
sudo rm -rf /etc/casper/1_0_*
sudo rm -rf /var/lib/casper/*
```

### Install Casper node

> **Note**  
> Different networks (e.g. Main Net, Test Net) may use different binaries, and the versions references below may be 
> outdated, or not appropriate for the network you're trying to join. First verify the binary version you need
> before installing!
> 
> The binaries below are for Casper Node v1.0.0, the main net genesis.

```
cd ~
curl -JLO https://bintray.com/casperlabs/debian/download_file?file_path=casper-node-launcher_0.3.2-0_amd64.deb
curl -JLO https://bintray.com/casperlabs/debian/download_file?file_path=casper-client_1.0.0-0_amd64.deb
sudo apt install -y ./casper-node-launcher_0.3.2-0_amd64.deb ./casper-client_1.0.0-0_amd64.deb
```

## Build smart contracts that are required to bond to the network 

### Install pre-requisites for building smart contracts

```
sudo apt purge --auto-remove cmake
wget -O - https://apt.kitware.com/keys/kitware-archive-latest.asc 2>/dev/null | gpg --dearmor - | sudo tee /etc/apt/trusted.gpg.d/kitware.gpg >/dev/null
sudo apt-add-repository 'deb https://apt.kitware.com/ubuntu/ focal main'   
sudo apt update
sudo apt install cmake

curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

sudo apt install libssl-dev
sudo apt install pkg-config
sudo apt install build-essential
```

### Build smart contracts

#### Pull sources

Go to your home directory and clone the node repository. Later we will use this path to the smart contracts in our bonding request.

```
cd ~

git clone git://github.com/CasperLabs/casper-node.git
cd casper-node/
```

#### Checkout the release branch

> **Note**  
> Verify that the version of your contracts matches the version of the casper-node software you have
> installed.

```
git checkout release-1.0.0
```

#### Build the contracts

```
make setup-rs
make build-client-contracts -j
```

## Generate keys and fund your account 

### Generate node keys

Navigate to the default key directory:
```
cd /etc/casper/validator_keys
```
 
And execute the following command to generate the keys:
```
sudo casper-client keygen .
```

It will create three files in the ```/etc/casper/validator_keys``` directory:

- ```secret_key.pem``` - your private key; never share it with anyone
- ```public_key.pem``` - your public key
- ```public_key_hex``` - hex representation of your public key; copy it to your machine to create an account

Save your keys to a safe place. 

### Create account

Go to [Clarity](https://clarity.make.services/#/accounts) and login using your Github or Google account. On the "Account" tab, click the "Import Account Key" button and select the file containing the hex representation of you public key ```public_key.pem```. Give it a name and hit "Save".  

### Fund account

To fund an account visit the [Faucet](https://clarity.make.services/#/faucet) page. Select the account you want to fund and hit "Request Tokens". Wait until the request transaction succeeds.


## Configure and Run the Node

### Set up configuration

```
sudo -u casper /etc/casper/pull_casper_node_version.sh $CASPER_VERSION $CASPER_NETWORK
sudo -u casper /etc/casper/config_from_example.sh $CASPER_VERSION
```

### Get known validator IP

> **Note**  
> Getting a known validator IP and setting your trusted hash is only required if you are joining
> a network after Genesis. If you are a Genesis validator no trusted hash is needed.

Let's get a known validator IP first. We'll use it multiple times later in the process.

```
KNOWN_ADDRESSES=$(cat /etc/casper/$CASPER_VERSION/config.toml | grep known_addresses)
KNOWN_VALIDATOR_IPS=$(grep -oE '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' <<< "$KNOWN_ADDRESSES")
IFS=' ' read -r KNOWN_VALIDATOR_IP _REST <<< "$KNOWN_VALIDATOR_IPS"

echo $KNOWN_VALIDATOR_IP
```

After running the commands above the ```$KNOWN_VALIDATOR_IP``` variable will contain IP address of a known validator.

### Set trusted hash

Get the trusted hash from the network:

```
# Get trusted_hash into config.toml
sudo sed -i "/trusted_hash =/c\trusted_hash = '$(curl -s $KNOWN_VALIDATOR_IP:8888/status | jq -r .last_added_block_info.hash | tr -d '\n')'" /etc/casper/$CASPER_VERSION/config.toml
```

### Start the node

```
sudo logrotate -f /etc/logrotate.d/casper-node
sudo systemctl start casper-node-launcher; sleep 2
systemctl status casper-node-launcher
```

### Monitor the node status

#### Check the node log

```
sudo tail -fn100 /var/log/casper/casper-node.log /var/log/casper/casper-node.stderr.log
```

#### Check if a known validator sees your node among peers

```
curl -s http://$KNOWN_VALIDATOR_IP:8888/status | jq .peers
```

You should see your IP address on the list

#### Check the node status

```
curl -s http://127.0.0.1:8888/status
```

#### Wait for node to catch up
Before you do anything, such as trying to bond as a validator or perform any RPC calls, make sure your node has fully 
caught up with the network. You can recognize this by log entries that tell you that joining has finished, and that the
RPC and REST servers have started:

```
{"timestamp":"Feb 09 02:28:35.577","level":"INFO","fields":{"message":"finished joining"},"target":"casper_node::cli"}
{"timestamp":"Feb 09 02:28:35.578","level":"INFO","fields":{"message":"started JSON-RPC server","address":"0.0.0.0:7777"},"target":"casper_node::components::rpc_server::http_server"}
{"timestamp":"Feb 09 02:28:35.578","level":"INFO","fields":{"message":"started REST server","address":"0.0.0.0:8888"},"target":"casper_node::components::rest_server::http_server"}
```

## Bond to the network

Once you ensure that your node is running correctly and is visible by other proceed to bonding.

### Check your balance

Check your balance to ensure you have funds to bond:

To get the balance we need to perform the following three query commands:

1. Get the state root hash (has to be performed for each balance check because the hash changes with time): 

    ```
    casper-client get-state-root-hash --node-address http://127.0.0.1:7777 | jq -r
    ```
2. Get the main purse associated with your account:

    ```
    casper-client query-state --node-address http://127.0.0.1:7777 --key <PUBLIC_KEY_HEX> --state-root-hash <STATE_ROOT_HASH> | jq -r
    ```

3. Get the main purse balance:

    ```
    casper-client get-balance --node-address http://127.0.0.1:7777 --purse-uref <PURSE_UREF> --state-root-hash <STATE_ROOT_HASH> | jq -r
    ```

If you followed the installation steps from this document you can run the following script to check the balance:

```
PUBLIC_KEY_HEX=$(cat /etc/casper/validator_keys/public_key_hex)
STATE_ROOT_HASH=$(casper-client get-state-root-hash --node-address http://127.0.0.1:7777 | jq -r '.result | .state_root_hash')
PURSE_UREF=$(casper-client query-state --node-address http://127.0.0.1:7777 --key "$PUBLIC_KEY_HEX" --state-root-hash "$STATE_ROOT_HASH" | jq -r '.result | .stored_value | .Account | .main_purse')
casper-client get-balance --node-address http://127.0.0.1:7777 --purse-uref "$PURSE_UREF" --state-root-hash "$STATE_ROOT_HASH" | jq -r '.result | .balance_value'
```

### Make bonding request

To bond to the network as a validator you need to submit your bid using ```casper-client```:

```
casper-client put-deploy \
        --chain-name "<CHAIN_NAME>" \
        --node-address "http://127.0.0.1:7777/" \
        --secret-key "/etc/casper/validator_keys/secret_key.pem" \
        --session-path "$HOME/casper-node/target/wasm32-unknown-unknown/release/add_bid.wasm" \
        --payment-amount 1000000000 \
        --gas-price=1 \
        --session-arg=public_key:"public_key='<PUBLIC_KEY_HEX>'" \
        --session-arg=amount:"u512='9000000000000000'" \
        --session-arg=delegation_rate:"u64='10'"
```

Where:
- ```amount``` - This is the amount that is being bid. If the bid wins, this will be the validator’s initial bond amount. Recommended bid in amount is 90% of your faucet balance.  This is 900,000 CSPR  or 9000000000000000 motes as an argument to the add_bid contract deploy. 
- ```delegation_rate``` - The percentage of rewards that the validator retains from delegators that delegate their tokens to the node.

Replace:
- ```<CHAIN_NAME>``` with the chain name you are joining
- ```<PUBLIC_KEY_HEX>``` with the hex representation of you public key 

Remember the ```deploy_hash``` returned in the response to query its status later.

If you followed the installation steps from this document you can run the following script to bond. It substitutes the public key hex value for you and sends recommended argument values:

```
PUBLIC_KEY_HEX=$(cat /etc/casper/validator_keys/public_key_hex)
CHAIN_NAME=$(curl -s http://127.0.0.1:8888/status | jq -r '.chainspec_name')

casper-client put-deploy \
    --chain-name "$CHAIN_NAME" \
    --node-address "http://127.0.0.1:7777/" \
    --secret-key "/etc/casper/validator_keys/secret_key.pem" \
    --session-path "$HOME/casper-node/target/wasm32-unknown-unknown/release/add_bid.wasm" \
    --payment-amount 1000000000 \
    --gas-price=1 \
    --session-arg=public_key:"public_key='$PUBLIC_KEY_HEX'" \
    --session-arg=amount:"u512='9000000000000000'" \
    --session-arg=delegation_rate:"u64='10'"
```

### Check that you bonding request worked

Sending a transaction to the network does not mean that the transaction processed successfully. It’s important to check to see that the contract executed properly:

```
casper-client get-deploy --node-address http://127.0.0.1:7777 <DEPLOY_HASH> | jq .result.execution_results
```

Replace ```<DEPLOY_HASH>``` with the deploy hash of the transaction you want to check.

### Query the auction info and look for your bid

To determine if the bid was accepted, execute the following command:

```
casper-client get-auction-info --node-address http://127.0.0.1:7777
```

The bid should appear among the returned ```bids```. If the public key associated with a bid appears in the ```validator_weights``` structure for an era, then the account is bonded in that era.
