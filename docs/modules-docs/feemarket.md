# `feemarket`

## Abstract[](https://docs.evmos.org/protocol/modules/feemarket#abstract)

This document specifies the feemarket module which allows to define a global transaction fee for the network.

This module has been designed to support EIP1559 in cosmos-sdk.

The `MempoolFeeDecorator` in `x/auth` module needs to be overwritten to check the `baseFee` along with the `minimal-gas-prices` allowing to implement a global fee mechanism which vary depending on the network activity.

For more reference to EIP1559:

https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1559.md

### EIP-1559: Fee Market[](https://docs.evmos.org/protocol/modules/feemarket#eip-1559-fee-market)

[EIP-1559](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1559.md) describes a pricing mechanism that was proposed on Ethereum to improve to calculation of transaction fees. It includes a fixed-per-block network fee that is burned and dynamically expands/contracts block sizes to deal with peaks of network congestion.

Before EIP-1559 the transaction fee is calculated with

```text
fee = gasPrice * gasLimit
```



where `gasPrice` is the price per gas and `gasLimit` describes the amount of gas required to perform the transaction. The more complex operations a transaction requires, the higher the gas limit (see [Executing EVM bytecode](https://docs.evmos.org/protocol/modules/evm#executing-evm-bytecode)). To submit a transaction, the signer needs to specify the `gasPrice`.

With EIP-1559 enabled, the transaction fee is calculated with

```text
fee = (baseFee + priorityTip) * gasLimit
```



where `baseFee` is the fixed-per-block network fee per gas and `priorityTip` is an additional fee per gas that can be set optionally. Note, that both the base fee and the priority tip are gas prices. To submit a transaction with EIP-1559, the signer needs to specify the `gasFeeCap`, which is the maximum fee per gas they are willing to pay in total. Optionally, the `priorityTip` can be specified, which covers both the priority fee and the block's network fee per gas (aka: base fee).

TIP

The Cosmos SDK uses a different terminology for `gas` than Ethereum. What is called `gasLimit` on Ethereum is called `gasWanted` on Cosmos. You might encounter both terminologies on Evmos since it builds Ethereum on top of the SDK, e.g. when using different wallets like Keplr for Cosmos and Metamask for Ethereum.

### Base Fee[](https://docs.evmos.org/protocol/modules/feemarket#base-fee)

The base fee per gas (aka base fee) is a global gas price defined at the consensus level. It is stored as a module parameter and is adjusted at the end of each block based on the total gas used in the previous block and gas target (`block gas limit / elasticity multiplier`):

- it increases when blocks are above the gas target,
- it decreases when blocks are below the gas target.

Instead of burning the base fee (as implemented on Ethereum), the `feemarket` module allocates the base fee for regular [Cosmos SDK fee distribution](https://docs.cosmos.network/main/modules/distribution).

### Priority Tip[](https://docs.evmos.org/protocol/modules/feemarket#priority-tip)

In EIP-1559, the `max_priority_fee_per_gas`, often referred to as `tip`, is an additional gas price that can be added to the `baseFee` in order to incentivize transaction prioritization. The higher the tip, the more likely the transaction is included in the block.

Until the Cosmos SDK version v0.46, however, there is no notion of transaction prioritization. Thus, the tip for an EIP-1559 transaction on Evmos should be zero (`MaxPriorityFeePerGas` JSON-RPC endpoint returns `0`). Have a look at the [mempool](https://docs.evmos.org/validate/setup-and-configuration/mempool) docs to read more about how to leverage transaction prioritization.

### Effective Gas price[](https://docs.evmos.org/protocol/modules/feemarket#effective-gas-price)

For EIP-1559 transactions (dynamic fee transactions) the effective gas price describes the maximum gas price that a transaction is willing to provide. It is derived from the transaction arguments and the base fee parameter. Depending on which one is smaller, the effective gas price is either the `baseFee + tip` or the `gasFeeCap`

```text
min(baseFee + gasTipCap, gasFeeCap)
```



### Local vs. Global Minimum Gas Prices[](https://docs.evmos.org/protocol/modules/feemarket#local-vs-global-minimum-gas-prices)

Minimum gas prices are used to discard spam transactions in the network, by raising the cost of transactions to the point that it is not economically viable for the spammer. This is achieved by defining a minimum gas price for accepting txs in the mempool for both Cosmos and EVM transactions. A transaction is discarded from the mempool if it doesn't provide at least one of the two types of min gas prices:

Minimum gas prices are used to discard spam transactions in the network, by raising the cost of transactions to the point that it is not economically viable for the spammer. This is achieved by defining a minimum gas price for accepting txs in the mempool for both Cosmos and EVM transactions. A transaction is discarded from the mempool if it doesn't provide at least one of the two types of min gas prices:

1. the local min gas prices that validators can set on their node config and
2. the global min gas price, which is set as a parameter in the `feemarket` module, which can be changed through governance.

The lower bound for a transaction gas price is determined by comparing of gas price bounds according to three cases:

1. If the effective gas price (`effective gas price = base fee + priority tip`) or the local minimum gas price is lower than the global `MinGasPrice` (`min-gas-price (local) < MinGasPrice (global) OR EffectiveGasPrice < MinGasPrice`), then `MinGasPrice` is used as a lower bound.
2. If transactions are rejected due to having a gas price lower than `MinGasPrice`, users need to resend the transactions with a gas price higher or equal to `MinGasPrice`.
3. If the effective gas price or the local `minimum-gas-price` is higher than the global `MinGasPrice`, then the larger value of the two is used as a lower bound. In the case of EIP-1559, users must increase the priority fee for their transactions to be valid.

The comparison of transaction gas price and the lower bound is implemented through AnteHandler decorators. For EVM transactions, this is done in the `EthMempoolFeeDecorator` and `EthMinGasPriceDecorator` `AnteHandler` and for Cosmos transactions in `NewMempoolFeeDecorator` and `MinGasPriceDecorator` `AnteHandler`.

TIP

If the base fee decreases to a value below the global `MinGasPrice`, it is set to the `MinGasPrice`. This is implemented, so that the base fee can't drop to gas prices that wouldn't allow transactions to be accepted in the mempool, because of a higher `MinGasPrice`.

## State[](https://docs.evmos.org/protocol/modules/feemarket#state)

The x/feemarket module keeps in the state variable needed to the fee calculation:

Only BlockGasUsed in previous block needs to be tracked in state for the next base fee calculation.

|              | Description           | Key         | Value              | Store |
| ------------ | --------------------- | ----------- | ------------------ | ----- |
| BlockGasUsed | gas used in the block | `[]byte{1}` | `[]byte{gas_used}` | KV    |

## Begin block[](https://docs.evmos.org/protocol/modules/feemarket#begin-block)

The base fee is calculated at the beginning of each block.

### Base Fee[](https://docs.evmos.org/protocol/modules/feemarket#base-fee-1)

#### Disabling base fee[](https://docs.evmos.org/protocol/modules/feemarket#disabling-base-fee)

We introduce two parameters : `NoBaseFee`and `EnableHeight`

`NoBaseFee` controls the feemarket base fee value. If set to true, no calculation is done and the base fee returned by the keeper is zero.

`EnableHeight` controls the height we start the calculation.

- If `NoBaseFee = false` and `height < EnableHeight`, the base fee value will be equal to `base_fee` defined in the genesis and the `BeginBlock` will return without further computation.
- If `NoBaseFee = false` and `height >= EnableHeight`, the base fee is dynamically calculated upon each block at `BeginBlock`.

Those parameters allow us to introduce a static base fee or activate the base fee at a later stage.

#### Enabling base fee[](https://docs.evmos.org/protocol/modules/feemarket#enabling-base-fee)

To enable EIP1559 with the EVM, the following parameters should be set :

- NoBaseFee should be false
- EnableHeight should be set to a positive integer >= upgrade height. It defines at which height the chain starts the base fee adjustment
- LondonBlock evm's param should be set to a positive integer >= upgrade height. It defines at which height the chain start to accept EIP1559 transactions

#### Calculation[](https://docs.evmos.org/protocol/modules/feemarket#calculation)

The base fee is initialized at `EnableHeight` to the `InitialBaseFee` value defined in the genesis file.

The base fee is after adjusted according to the total gas used in the previous block.

```golang
parent_gas_target = parent_gas_limit / ELASTICITY_MULTIPLIER

if EnableHeight == block.number
    base_fee = INITIAL_BASE_FEE
else if parent_gas_used == parent_gas_target:
    base_fee = parent_base_fee
else if parent_gas_used > parent_gas_target:
    gas_used_delta = parent_gas_used - parent_gas_target
    base_fee_delta = max(parent_base_fee * gas_used_delta / parent_gas_target / BASE_FEE_MAX_CHANGE_DENOMINATOR, 1)
    base_fee = parent_base_fee + base_fee_delta
else:
    gas_used_delta = parent_gas_target - parent_gas_used
    base_fee_delta = parent_base_fee * gas_used_delta / parent_gas_target / BASE_FEE_MAX_CHANGE_DENOMINATOR
    base_fee = parent_base_fee - base_fee_delta
```



## End block[](https://docs.evmos.org/protocol/modules/feemarket#end-block)

The `block_gas_used` value is updated at the end of each block.

### Block Gas Used[](https://docs.evmos.org/protocol/modules/feemarket#block-gas-used)

The total gas used by current block is stored in the KVStore at `EndBlock`.

It is initialized to `block_gas` defined in the genesis.

## Keeper[](https://docs.evmos.org/protocol/modules/feemarket#keeper)

The feemarket module provides this exported keeper that can be passed to other modules, which require access to the base fee value

```go
type Keeper interface {
    GetBaseFee(ctx sdk.Context) *big.Int
}
```



## Events[](https://docs.evmos.org/protocol/modules/feemarket#events)

The `x/feemarket` module emits the following events:

### BeginBlocker[](https://docs.evmos.org/protocol/modules/feemarket#beginblocker)

| Type       | Attribute Key | Attribute Value |
| ---------- | ------------- | --------------- |
| fee_market | base_fee      | {baseGasPrices} |

### EndBlocker[](https://docs.evmos.org/protocol/modules/feemarket#endblocker)

| Type      | Attribute Key | Attribute Value |
| --------- | ------------- | --------------- |
| block_gas | height        | {blockHeight}   |
| block_gas | amount        | {blockGasUsed}  |

## Parameters[](https://docs.evmos.org/protocol/modules/feemarket#parameters)

The `x/feemarket` module contains the following parameters:

| Key                      | Type    | Default Values | Description                                                  |
| ------------------------ | ------- | -------------- | ------------------------------------------------------------ |
| NoBaseFee                | bool    | false          | control the base fee adjustment                              |
| BaseFeeChangeDenominator | uint32  | 8              | bounds the amount the base fee that can change between blocks |
| ElasticityMultiplier     | uint32  | 2              | bounds the threshold which the base fee will increase or decrease depending on the total gas used in the previous block |
| BaseFee                  | uint32  | 1000000000     | base fee for EIP-1559 blocks                                 |
| EnableHeight             | uint32  | 0              | height which enable fee adjustment                           |
| MinGasPrice              | sdk.Dec | 0              | global minimum gas price that needs to be paid to include a transaction in a block |

## Client[](https://docs.evmos.org/protocol/modules/feemarket#client)

### CLI[](https://docs.evmos.org/protocol/modules/feemarket#cli)

A user can query and interact with the `feemarket` module using the CLI.

#### Queries[](https://docs.evmos.org/protocol/modules/feemarket#queries)

The `query` commands allow users to query `feemarket` state.

```bash
evmosd query feemarket --help
```



##### Base Fee[](https://docs.evmos.org/protocol/modules/feemarket#base-fee-2)

The `base-fee` command allows users to query the block base fee by height.

```bash
evmosd query feemarket base-fee [flags]
```



Example:

```bash
evmosd query feemarket base-fee ...
```



Example Output:

```text
base_fee: "512908936"
```



##### Block Gas[](https://docs.evmos.org/protocol/modules/feemarket#block-gas)

The `block-gas` command allows users to query the block gas by height.

```bash
evmosd query feemarket block-gas [flags]
```



Example:

```bash
evmosd query feemarket block-gas ...
```



Example Output:

```text
gas: "21000"
```



##### Params[](https://docs.evmos.org/protocol/modules/feemarket#params)

The `params` command allows users to query the module params.

```bash
evmosd query params subspace [subspace] [key] [flags]
```



Example:

```bash
evmosd query params subspace feemarket ElasticityMultiplier ...
```



Example Output:

```text
key: ElasticityMultiplier
subspace: feemarket
value: "2"
```



### gRPC[](https://docs.evmos.org/protocol/modules/feemarket#grpc)

#### Queries[](https://docs.evmos.org/protocol/modules/feemarket#queries-1)

| Verb   | Method                                  | Description            |
| ------ | --------------------------------------- | ---------------------- |
| `gRPC` | `ethermint.feemarket.v1.Query/Params`   | Get the module params  |
| `gRPC` | `ethermint.feemarket.v1.Query/BaseFee`  | Get the block base fee |
| `gRPC` | `ethermint.feemarket.v1.Query/BlockGas` | Get the block gas used |
| `GET`  | `/ethermint/feemarket/v1/params`        | Get the module params  |
| `GET`  | `/ethermint/feemarket/v1/base_fee`      | Get the block base fee |
| `GET`  | `/ethermint/feemarket/v1/block_gas`     | Get the block gas used |

## AnteHandlers[](https://docs.evmos.org/protocol/modules/feemarket#antehandlers)

The `x/feemarket` module provides `AnteDecorator`s that are recursively chained together into a single [`Antehandler`](https://github.com/cosmos/cosmos-sdk/blob/v0.43.0-alpha1/docs/architecture/adr-010-modular-antehandler.md). These decorators perform basic validity checks on an Ethereum or Cosmos SDK transaction, such that it could be thrown out of the transaction Mempool.

Note that the `AnteHandler` is run for every transaction and called on both `CheckTx` and `DeliverTx`.

### Decorators[](https://docs.evmos.org/protocol/modules/feemarket#decorators)

### `MinGasPriceDecorator`[](https://docs.evmos.org/protocol/modules/feemarket#mingaspricedecorator)

Rejects Cosmos SDK transactions with transaction fees lower than `MinGasPrice * GasLimit`.

### `EthMinGasPriceDecorator`[](https://docs.evmos.org/protocol/modules/feemarket#ethmingaspricedecorator)

Rejects EVM transactions with transactions fees lower than `MinGasPrice * GasLimit`.

- For `LegacyTx` and `AccessListTx`, the `GasPrice * GasLimit` is used.
- For EIP-1559 (*aka.* `DynamicFeeTx`), the `EffectivePrice * GasLimit` is used.

TIP

**Note**: For dynamic transactions, if the `feemarket` formula results in a `BaseFee` that lowers `EffectivePrice < MinGasPrices`, the users must increase the `GasTipCap` (priority fee) until `EffectivePrice > MinGasPrices`. Transactions with `MinGasPrices * GasLimit < transaction fee < EffectiveFee` are rejected by the `feemarket` `AnteHandle`.

### `EthGasConsumeDecorator`[](https://docs.evmos.org/protocol/modules/feemarket#ethgasconsumedecorator)

Calculates the effective fees to deduct and the tx priority according to EIP-1559 spec, then deducts the fees and sets the tx priority in the response.

```text
effectivePrice = min(baseFee + tipFeeCap, gasFeeCap)
effectiveTipFee = effectivePrice - baseFee
priority = effectiveTipFee / DefaultPriorityReduction
```

