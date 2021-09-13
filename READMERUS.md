# Инструкция по созданию Смарт контракта ERC20 в Juno testnet lucina
Создадим свой собственный memecoin!

Для этого нам понадобиться:
1. Написать свой смарт контракт(без знаний языка и прочего)
2. Сохранить его в блокчейне
3. Создать экземпляр смарт контракта
4. Выполнить команды, предусмотренные для смарт-контракта

Мы выполним все эти пункты ниже:

# Установка

Следуйте этим шагам для установки rust и junod [installation page](https://docs.junochain.com/smart-contracts/installation).


# Rust

Предполагая, что вы никогда не работали с растом , вам сначала нужно установить некоторые инструменты. Стандартный подход - использовать rustup для поддержки зависимостей и обработки обновления нескольких версий Cargo и rustc, которые мы будете использовать.



Для начало , [установим rustup](https://rustup.rs/). После установки проверим есть ли у нас wasm32 target:

```bash
rustup default stable
cargo version
# If this is lower than 1.49.0+, update
rustup update stable

rustup target list --installed
rustup target add wasm32-unknown-unknown
```
# Сбилдим Juno для использование в тестнете
Используя go 1.16.3 для компилирования исполняймого бинарника junod 
```bash
# clone juno repo
git clone https://github.com/CosmosContracts/juno.git && cd juno

# build juno executable
make install

which junod
```
# Скачмваем, компилируем и сохраняем


```bash
# get the code
git clone https://github.com/CosmWasm/cosmwasm-examples
cd cosmwasm-examples
git fetch
git checkout v0.10.0 # current at time of writing
cd contracts/erc20
```
# Компиляция
We can compile our contract like so:

```bash
# compile the wasm contract with stable toolchain
rustup default stable
cargo wasm
```
Однако нам необходимо создать оптимизированную версию для ограничения использования газа, поэтому мы запустим:

```bash
sudo docker run --rm -v "$(pwd)":/code \
    --mount type=volume,source="$(basename "$(pwd)")_cache",target=/code/target \
    --mount type=volume,source=registry_cache,target=/usr/local/cargo/registry \
    cosmwasm/rust-optimizer:0.11.4
```

Это приведет к созданию артефакта cw_erc20.wasm в каталоге артефактов. 

# Выгружаем
Сейчас мы можем выгрузить или сохранить, в блокчейне используя свою ноду или публичную rpc ноду 

```bash
cd artifacts
junod tx wasm store cw_erc20.wasm  --from <your-key> --chain-id=<chain-id> --gas auto
```
!! Вам нужно будет найти в выходных данных этой команды идентификатор кода контракта.
В JSON файле, этоьбудет выглядит вот так  {"key":"code_id","value":"6"} в выводе команды.


# Инициализируем контракт
А теперь пора повеселиться. Давайте настроим и запустим этот контракт.

Теперь мы загрузили контракт, теперь нам нужно его инициализировать.
Мы используем Web34ever Coin как пример здкесь - $WEB coin задэплоен в Juno testnet.

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

