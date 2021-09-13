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
