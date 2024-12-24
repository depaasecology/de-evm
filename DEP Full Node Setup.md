## 1. Download the Binary File

To set up a full node, download the appropriate binary file based on the blockchain height:

- **Before block height 10,286,999**: Use the binary file from version 11.0.2-rc1.
    - Link: [v11.0.2-rc1](https://github.com/depaasecology/de-evm/releases/tag/v11.0.2-rc1)
- **After block height 10,286,999**: Use the binary file from version 13.0.2.
    - Link: [v13.0.2](https://github.com/depaasecology/de-evm/releases/tag/v13.0.2)

## 2. Download the Genesis File

Download the genesis file required for initializing the blockchain:

- [Genesis File](https://storage.googleapis.com/evmosd_node_bucket/full_node/genesis.json)

## 3. Initialize the Node

Run the following command to initialize the node. By default, it will create a directory in your home folder under `.evmosd`:

```bash
evmosd init de --chain-id=demaster_9000-1

```

## 4. Configure JSON-RPC Module

Edit the `app.toml` file to enable and configure the JSON-RPC module:

```bash
vim ~/.evmosd/config/app.toml

```

Update the following section:

```toml
[json-rpc]
enable = true
address = "0.0.0.0:8545"
ws-address = "0.0.0.0:8546"
api = "eth,txpool,net,debug,web3"

```

- **8545**: Default RPC port for EVM; set IP to `0.0.0.0`.
- **8546**: Default WS port for EVM; set IP to `0.0.0.0`.

## 5. Configure Validator Seeds

Retrieve the latest seed nodes from the following link:

- [Seeds File](https://raw.githubusercontent.com/depaasecology/mainnet/refs/heads/main/demaster_202002-1/seeds.txt)

Edit the `config.toml` file to add validator seed nodes:

```bash
vim ~/.evmosd/config/config.toml

```

Update the following line:

```toml
seeds = "<SEED_NODES_FROM_FILE>"

```

Replace `<SEED_NODES_FROM_FILE>` with the content of the downloaded `seeds.txt` file.

## 6. Copy the Genesis File

Copy the downloaded genesis file into the `.evmosd` configuration folder:

```bash
cp genesis.json ~/.evmosd/config/

```

## 7. Run the Full Node

Start the full node using the following command:

```bash
nohup evmosd start --pruning=nothing --log_level info --home ~/.evmosd >> evmos.log &

```

### Explanation:

- `-pruning=nothing`: Retains all blockchain states without deletion.
    - **Options:**
        - `everything`: Deletes all saved states except the current one.
        - `nothing`: Saves all states.
        - `default`: Saves the last 100 states and every 10,000th block state.
        - `custom`: Allows customized pruning using `pruning-keep-recent`, `pruning-keep-every`, and `pruning-interval` parameters.
- `-home ~/.evmosd`: Specifies the home directory for `evmosd`. The default directory is used here.
- Logs will be written to `evmos.log`.

---

### Notes

- Ensure the correct binary file is used based on the blockchain height.
- Keep your `app.toml` and `config.toml` configurations updated for proper synchronization.

### Binary File Links:

- [v11.0.2-rc1](https://github.com/depaasecology/de-evm/releases/tag/v11.0.2-rc1) (Before block height 10,286,999)
- [v13.0.2](https://github.com/depaasecology/de-evm/releases/tag/v13.0.2) (After block height 10,286,999)