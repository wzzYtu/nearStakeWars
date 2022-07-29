
# Create a near validator using a Contabo Cloud Service

The NEAR protocol is an underlying blockchain that uses the unique sharding technology of the Night Shadow Protocol to achieve scalability. In 2020, the protocol went live as a decentralized cloud infrastructure for onboard decentralized applications (DApps).

NEAR Blockchain utilizes Rainbow Bridge and Layer 2 solution Aurora to enable cross-chain interoperability. Users can transfer ERC-20 tokens and other assets in the Ethereum blockchain into the NEAR protocol network, enjoying the convenience of the latter's high throughput and low transaction fees.

NEAR is the native token of the NEAR protocol and is used to pay transaction fees and data storage fees. NEAR token holders can also use the NEAR wallet to stake tokens to earn rewards, or use tokens to vote on governance proposals.

NEAR validator is an important guarantee to ensure the safe operation of the network. This article will introduce how to use the Contabo cloud server to build a NEAR validator node.

## 1. Create a near wallet
- Open the link (https://wallet.shardnet.near.org/) in the upper right corner of the webpage, where you can import or create a new wallet;
- If you are a new user, you can choose to create a wallet. To create a wallet, you can use your email address, mobile phone number, annotated words and Ledger hardware wallet. For Web3 users who are more accustomed to the latter two methods, this article will introduce how to use the mnemonic to create a wallet;
- Then you need to create a unique Account ID, which has a unified format AccountName.shardnet.near;
- After creating a unique Account ID, a set of 12-word mnemonics will be generated, which need to be stored properly. Owning the mnemonic is equivalent to using it for all near tokens in the account;
- In order to ensure that the user has saved the mnemonic, the web wallet will randomly check the nth mnemonic;
- Finally, enter the mnemonic, and you can successfully create a wallet. Note that the wallet address created on this page is for testing, and the near in this address is only for testing and has no actual value;

![avatar](../picture/walletCreate.png)

- The process of creating a wallet on the main network is different from that in the test network. In the main network, a small amount of NEAR Token needs to be sent to the newly created wallet to activate the wallet. Other content is basically the same as the test network method, which can be used for reference.

## 2. Create a validator
- This tutorial uses ubuntu 20.04 to run the validator, you can use the following command to update the server program.
```bash
sudo apt update && sudo apt upgrade -y
````

- Since near-cli is installed using npm, node needs to be installed first
```bash
# install node
curl -sL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install build-essential nodejs
PATH="$PATH"

# Check node and npm versions
node -v

npm -v
````

- install near-cli
```bash
sudo npm install -g near-cli
````

After installing near-cli successfully, you can use `near proposals` to view all validators in the proposal state, use `near validators current` to view all current validators, and use `near validators next` to view those who will become validators node

- Check whether the server configuration meets the requirements, the recommended configuration for the test network is 4-Core CPU with AVX support/8GB DDR4/500G SSD
```bash
lscpu | grep -P '(?=.*avx )(?=.*sse4.2 )(?=.*cx16 )(?=.*popcnt )' > /dev/null \
  && echo "Supported" \
  || echo "Not supported"
````

- Preliminary preparation, install development tools, Python pip and rust
```bash
# install development tools
sudo apt install -y git binutils-dev libcurl4-openssl-dev zlib1g-dev libdw-dev libiberty-dev cmake gcc g++ python docker.io protobuf-compiler libssl-dev pkg-config clang llvm cargo

# install python3-pip
sudo apt install python3-pip

# Configure environment variables
USER_BASE_BIN=$(python3 -m site --user-base)/bin
export PATH="$USER_BASE_BIN:$PATH"

# Configure the development environment
sudo apt install clang build-essential make

# Install rust, use the default installation method
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
# configure rust environment variables
source $HOME/.cargo/env
````

- Download near source code
```bash
git clone https://github.com/near/nearcore
cd nearcore
git fetch

# switch to the latest branch
git checkout <commit>

# compile source code
cargo build -p neard --release --features shardnet
````

- Initialize the near node
```bash
./target/release/neard --home ~/.near init --chain-id shardnet --download-genesis
````
The `--home` parameter specifies the initialization file location and data storage location. It is recommended to use an expandable magnetic field to store block data to ensure smooth operation of the node for a longer time.

- Start the near full node
```bash
cd ~/nearcore
./target/release/near --home ~/.near run
````

- Authorize the wallet
```bash
near login
````
This step will generate a link, which needs to be opened in the browser. Those who use cloud services can open the link locally by modifying the ip address in the url to complete the authorization.

- Create validator_key.json
```bash
near generate-key <pool_id>
````
This command creates `YOUR_WALLET.json` under `~/.near-credentials/shardnet/`, you can use the command `cp ~/.near-credentials/shardnet/YOUR_WALLET.json ~/.near/validator_key.json` Copy to `.near` directory
- Use the daemon to start the node, the server file can refer to
```bash
[Unit]
Description=NEARd Daemon Service

[Service]
Type=simple
User=<USER>
#Group=near
WorkingDirectory=/home/<USER>/.near
ExecStart=/home/<USER>/nearcore/target/release/near run
Restart=on-failure
RestartSec=30
KillSignal=SIGINT
TimeoutStopSec=45
KillMode=mixed

[Install]
WantedBy=multi-user.target
````

- Deploy the Staking Pool contract
```bash
near call factory.shardnet.near create_staking_pool '{"staking_pool_id": "<pool id>", "owner_id": "<accountId>", "stake_public_key": "<public key>", "reward_fee_fraction": {"numerator" : 5, "denominator": 100}, "code_hash":"DD428g9eqLL8fWUxv8QSpVFzyHi1Qd16P8ephYCTmMSZ"}' --accountId="<accountId>" --amount=30 --gas=300000000000000
````

![avatar](../picture/StakingPool.png)

- make a mortgage
```bash
near call <staking_pool_id> deposit_and_stake --amount <amount> --accountId <accountId> --gas=300000000000000
````


- De-collateralize
```bash
near call <staking_pool_id> unstake '{"amount": "<amount yoctoNEAR>"}' --accountId <accountId> --gas=300000000000000
````

- Redeem all proceeds
```bash
near call <staking_pool_id> withdraw_all --accountId <accountId> --gas=3000000000000000
````

## 3. Install monitoring program

The smooth operation of nodes is the premise to ensure the security of the Near network and maximize the benefits of validators. The official provides a complete monitoring program to monitor whether the validator node is running smoothly. To use the monitoring program, you need to open port 3030.

- Install monitoring tools
```bash
sudo apt install curl jq
````

- View the version the node is running

```bash
curl -s http://127.0.0.1:3030/status | jq .version
````

![avatar](../picture/nearVersion.png)

- Check Delegators and Stake Command:
```bash
near view <your pool>.factory.shardnet.near get_accounts '{"from_index": 0, "limit": 10}' --accountId <accountId>.shardnet.near
````

![avatar](../picture/checkStake.png)


- Check Reason Validator Kicked Command:
```bash
curl -s -d '{"jsonrpc": "2.0", "method": "validators", "id": "dontcare", "params": [null]}' -H 'Content-Type: application/json ' 127.0.0.1:3030 | jq -c '.result.prev_epoch_kickout[] | select(.account_id | contains ("<POOL_ID>"))' | jq .reason
````

- Check Blocks Produced / Expected Command:
```bash
curl -s -d '{"jsonrpc": "2.0", "method": "validators", "id": "dontcare", "params": [null]}' -H 'Content-Type: application/json ' 127.0.0.1:3030 | jq -c '.result.current_validators[] | select(.account_id | contains ("POOL_ID"))'
````

![avatar](../picture/createBlock.png)