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
Мы можем скомпилировать наш контракт так:

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

# Генерируем JSON с аргументами

To generate the JSON, you can use jq, or, if you're more familiar with JS/node, write a hash and encode it using the node CLI.
This example uses the node REPL. If you have node installed, just type node in the terminal and hit enter to access it.

Чтобы сгенерировать JSON, вы можете использовать jq или, если вы более знакомы с JS / node, написать хэш и закодировать его с помощью node CLI .
В этом примере используется node REPL. Если у вас установлен node, просто введите "node" в терминале и нажмите Enter, чтобы получить к нему доступ.
```bash
node
```
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
# Создаем экземпляр контракта

Также обратите внимание, что --amount используется для инициализации новой учетной записи, связанной с контрактом.
В приведенном ниже примере 65 - это значение $ CODE_ID.

```bash
junod tx wasm instantiate 65 \
    '{"name":"Web34ever Coin","symbol":"WEB","decimals":6,"initial_balances":[{"address":"<validator-self-delegate-address>","amount":"12345678000"}]}' \
    --amount 50000ujuno  --label "Web34evercoin erc20" --from web34ever --chain-id lucina --gas auto -y
```

Если это удастся, посмотрите вывод и получите адрес контракта из вывода, например, juno1a2b ....

Далле можем создать переменную нашего смарт контракта для удобства:
```bash
CONTRACT_ADDR=[аддресс вашего контракта из предыдущей команды]
```
И сделаем запрос для нашего контракта

```bash
junod query wasm contract $CONTRACT_ADDR
```

# Запросы и команды
Как запрашивать и выполнять команды в новом нашем блестящем контракте

Теперь мы можем проверить, что в контракте указана правильная сумма для адреса самоделегирования:

```bash
junod query wasm contract-state smart <contract-address> '{"balance":{"address":"<validator-self-delegate-address>"}}'
```
Из приведенного выше примера он вернет:
data:
  balance: "12345678000"
  
Использование команд, поддерживаемых функцией execute, работает таким же образом. "Заклинание" для выполнения команд контракта через интерфейс командной строки выглядит так:

```bash
junod tx wasm execute [contract_addr_bech32] [json_encoded_send_args] --amount [coins,optional] [flags]
```
Для перевода можно использовать команду:

```bash
junod tx wasm execute <contract-addr> '{"transfer":{"amount":"200","owner":"<validator-self-delegate-address>","recipient":"<recipient-address>"}}' --from <your-key> --chain-id <chain-id>
```
В папке контрактов / erc20 внутри cosmwasm-examples, например, вы можете увидеть схемы:

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
Каждый из файлов JSON выше представляет собой схему JSON, определяющую правильную форму JSON, которую он принимает.

Вот и все, наслаждайтесь своим мемкойном на Juno, у вас будет возможность сделать это в основной сети после запуска. Удачи!
