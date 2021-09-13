# Smart Contract ERC20 Tutorial on Juno testnet
Make your own memecoin!
This will take you through uploading your own memecoin to the juno testnet.

There are four steps involved in working with a smart contract.
1. Write the smart contract (already done here!)
2. Store the smart contract on chain
3. Instantiate the smart contract (configure and initialise it)
4. Execute commands provided by the smart contract

We will go through all of these in this tutorial.

# Installation

Follow the steps on the [installation page](https://docs.junochain.com/smart-contracts/installation).
The short version is that you will need rust and junod available.

# Rust

Assuming you have never worked with rust, you will first need to install some tooling. The standard approach is to use rustup to maintain dependencies and handle updating multiple versions of cargo and rustc, which you will be using.

First, [install rustup](https://rustup.rs/). Once installed, make sure you have the wasm32 target:

```bash
rustup default stable
cargo version
# If this is lower than 1.49.0+, update
rustup update stable

rustup target list --installed
rustup target add wasm32-unknown-unknown
```
# Building Juno for testnet use
A testnet running the Juno chain has been launched to save you of the hassle of running a local network and speed up your development.
Use go 1.16.3 for compiling the junodexecutable if you are building from source. If you already are running a validator node, it's likely junod is already accessible. If which junod shows output, then you're probably good to go.

```bash
# clone juno repo
git clone https://github.com/CosmosContracts/juno.git && cd juno

# build juno executable
make install

which junod
```
# Download, Compile, Store
Now we're going to download a contract, compile it, and upload it to the Juno chain.

```bash
# get the code
git clone https://github.com/CosmWasm/cosmwasm-examples
cd cosmwasm-examples
git fetch
git checkout v0.10.0 # current at time of writing
cd contracts/erc20
```
# Compile
We can compile our contract like so:

```bash
# compile the wasm contract with stable toolchain
rustup default stable
cargo wasm
```
However, we want to create an optimised version to limit gas usage, so we're going to run:

```bash
sudo docker run --rm -v "$(pwd)":/code \
    --mount type=volume,source="$(basename "$(pwd)")_cache",target=/code/target \
    --mount type=volume,source=registry_cache,target=/usr/local/cargo/registry \
    cosmwasm/rust-optimizer:0.11.4
```
This will result in an artifact called cw_erc20.wasm being created in the artifacts directory.

# Uploading
You can now upload, or 'store' this to the chain via your local node.

```bash
cd artifacts
junod tx wasm store cw_erc20.wasm  --from <your-key> --chain-id=<chain-id> --gas auto
```
!! You will need to look in the output for this command for the code ID of the contract. 
In the JSON, it will look like {"key":"code_id","value":"6"} in the output.

Alternatively, you can capture the output of the command run above, by doing these steps instead, and use the jq tool installed earlier to get the code_id value:

```bash
cd artifacts
RES=$(junod tx wasm store cw_erc20.wasm  --from <your-key> --chain-id=<chain-id> --gas auto -y)
CODE_ID=$(echo $RES | jq -r '.logs[0].events[0].attributes[-1].value')
```

You can now see this value with:
echo $CODE_ID

# Initialise the Contract
Now it's time for the fun stuff. Let's configure and get this contract up-and-running.

Now we've uploaded the contract, now we need to initialise it.
We're using the Web34ever Coin example here - $web3 was meme coin deployed to a Juno testnet.

# Generate JSON with arguments

To generate the JSON, you can use jq, or, if you're more familiar with JS/node, write a hash and encode it using the node CLI.
This example uses the node REPL. If you have node installed, just type node in the terminal and hit enter to access it.

```bash
> const initHash = {
  name: "Web34ever Coin",
  symbol: "web3",
  decimals: 6,
  initial_balances: [
    { address: "juno1034x52hzw8flvtdc6r4984te2720q72c23y7vd", amount: "12345678000"},
  ]
};
< undefined
> JSON.stringify(initHash);
< '{"name":"Web34ever Coin","symbol":"web3","decimals":6,"initial_balances":[{"address":"juno1034x52hzw8flvtdc6r4984te2720q72c23y7vd","amount":"12345678000"}]}'
```
# Instantiate the contract
Note also that the --amount is used to initialise the new account associated with the contract.
In the example below, 6 is the value of $CODE_ID.


```bash
junod tx wasm instantiate 6 \
    '{"name":"Web34ever Coin","symbol":"web3","decimals":6,"initial_balances":[{"address":"<validator-self-delegate-address>","amount":"12345678000"}]}' \
    --amount 50000ujuno  --label "Web34evercoin erc20" --from web34ever --chain-id lucina --gas auto -y
```

If this succeeds, look in the output and get contract address from output e.g juno1a2b.... or run:

```bash
CONTRACT_ADDR=$(junod query wasm list-contract-by-code $CODE_ID | jq -r '.[0].address')
```

This will allow you to query using the value of $CONTRACT_ADDR

```bash
junod query wasm contract $CONTRACT_ADDR
```

# Query and run commands
How to query and execute commands on your shiny new contract

Now you can check that the contract has assigned the right amount to the self-delegate address:

```bash
junod query wasm contract-state smart <contract-address> '{"balance":{"address":"<validator-self-delegate-address>"}}'
```
From the example above, it will return:
data:
  balance: "12345678000"
  
  Using the commands supported by execute work the same way. The incantation for executing commands on a contract via the CLI is:
```bash
junod tx wasm execute [contract_addr_bech32] [json_encoded_send_args] --amount [coins,optional] [flags]
```
You can omit --amount if not needed for execute calls.
In this case, your command will look something like:
```bash
junod tx wasm execute <contract-addr> '{"transfer":{"amount":"200","owner":"<validator-self-delegate-address>","recipient":"<recipient-address>"}}' --from <your-key> --chain-id <chain-id>
```
In the folder contracts/erc20 within cosmwasm-examples, for example, you can see the schemas:

```bash
tree schema

schema
├── allowance_response.json
├── balance_response.json
├── constants.json
├── execute_msg.json
├── instantiate_msg.json
└── query_msg.json
```
Each of the JSON files above is a JSON schema, specifying the correct shape of JSON that it accepts.

That's all , enjoy your own memcoin on juno, you will have opportunity make this in mainnet after launch. Good luck!
