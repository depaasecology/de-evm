# `revenue`

## Abstract[](https://docs.evmos.org/protocol/modules/revenue#abstract)

This document specifies the internal `x/revenue` module of the Evmos Hub.

The `x/revenue` module enables the Evmos Hub to support splitting transaction fees between block proposer and smart contract deployers. As a part of the [Evmos Token Model](https://evmos.blog/the-evmos-token-model-edc07014978b), this mechanism aims to increase the adoption of the Evmos Hub by offering a new stable source of income for smart contract deployers. Developers can register their smart contracts and everytime someone interacts with a registered smart contract, the contract deployer or their assigned withdrawal account receives a part of the transaction fees.

Together, all registered smart contracts make up the Evmos dApp Store: paying developers and network operators for their services via built-in shared fee revenue model.



## Evmos dApp Store[](https://docs.evmos.org/protocol/modules/revenue#evmos-dapp-store)

The Evmos dApp store is a revenue-per-transaction model, which allows developers to get payed for deploying their decentralized application (dApps) on Evmos. Developers generate revenue, every time a user interacts with their dApp in the dApp store, gaining them a steady income. Users can discover new applications in the dApp store and pay for the transaction fees that finance the dApp's revenue. This value-reward exchange of dApp services for transaction fees is implemented by the `x/revenue` module.

### Registration[](https://docs.evmos.org/protocol/modules/revenue#registration)

Developers register their application in the dApp store by registering their application's smart contracts. Any contract can be registered by a developer by submitting a signed transaction. The signer of this transaction must match the address of the deployer of the contract in order for the registration to succeed. After the transaction is executed successfully, the developer will start receiving a portion of the transaction fees paid when a user interacts with the registered contract.

TIP

**NOTE**: If your contract is part of a developer project, please ensure that the deployer of the contract (or the factory that deployes the contract) is an account that is owned by that project. This avoids the situtation, that an individual deployer who leaves your project could become malicious.

### Fee Distribution[](https://docs.evmos.org/protocol/modules/revenue#fee-distribution)

As described above, developers will earn a portion of the transaction fee after registering their contracts. To understand how transaction fees are distributed, we look at the following two things in detail:

- The transactions eligible are only [EVM transactions](https://docs.evmos.org/protocol/modules/evm) (`MsgEthereumTx`). Cosmos SDK transactions are not eligible at this time.
- The registration of factory contracts (smart contracts that have been deployed by other contracts) requires the identification original contract's deployer. This is done through address derivation.

#### EVM Transaction Fees[](https://docs.evmos.org/protocol/modules/revenue#evm-transaction-fees)

Users pay transaction fees to pay interact with smart contracts using the EVM. When a transaction is executed, the entire fee amount (`gasLimit * gasPrice`) is sent to the `FeeCollector` module account during the [Cosmos SDK AnteHandler](https://docs.cosmos.network/main/modules/auth#antehandlers) execution. After the EVM executes the transaction, the user receives a refund of `(gasLimit - gasUsed) * gasPrice`. In result a user pays a total transaction fee of `txFee = gasUsed * gasPrice` for the execution.

This transaction fee is distributed between developers and validators, in accordance with the `x/revenue` module parameters: `DeveloperShares`, `ValidatorShares`. This distribution is handled through the EVM's [`PostTxProcessing` Hook](https://docs.evmos.org/protocol/modules/revenue#hooks).

#### Address Derivation[](https://docs.evmos.org/protocol/modules/revenue#address-derivation)

dApp developers might use a [factory pattern](https://en.wikipedia.org/wiki/Factory_method_pattern) to implement their application logic through smart contracts. In this case a smart contract can be either deployed by an Externally Owned Account ([EOA](https://ethereum.org/en/whitepaper/#ethereum-accounts): an account controlled by a private key, that can sign transactions) or through another contract.

In both cases, the fee distribution requires the identification a deployer address that is an EOA address, unless a withdrawal address is set by the contract deployer during registration to receive transaction fees for a registered smart contract. If a withdrawal address is not set, it defaults to the deployer’s address.

The identification of the deployer address is done through address derivation. When registering a smart contract, the deployer provides an array of nonces, used to [derive the contract’s address](https://github.com/ethereum/go-ethereum/blob/d8ff53dfb8a516f47db37dbc7fd7ad18a1e8a125/crypto/crypto.go#L107-L111):

- If `MyContract` is deployed directly by `DeployerEOA`, in a transaction sent with nonce `5`, then the array of nonces is `[5]`.
- If the contract was created by a smart contract, through the `CREATE` opcode, we need to provide all the nonces from the creation path. E.g. if `DeployerEOA` deploys a `FactoryA` smart contract with nonce `5`. Then, `DeployerEOA` sends a transaction to `FactoryA` through which a `FactoryB` smart contract is created. If we assume `FactoryB` is the second contract created by `FactoryA`, then `FactoryA`'s nonce is `2`. Then, `DeployerEOA` sends a transaction to the `FactoryB` contract, through which `MyContract` is created. If this is the first contract created by `FactoryB` - the nonce is `1`. We now have an address derivation path of `DeployerEOA` -> `FactoryA` -> `FactoryB` -> `MyContract`. To be able to verify that `DeployerEOA` can register `MyContract`, we need to provide the following nonces: `[5, 2, 1]`.

TIP

**Note**: Even if `MyContract` is created from `FactoryB` through a transaction sent by an account different from `DeployerEOA`, only `DeployerEOA` can register `MyContract`.

## State[](https://docs.evmos.org/protocol/modules/revenue#state)

### State Objects[](https://docs.evmos.org/protocol/modules/revenue#state-objects)

The `x/revenue` module keeps the following objects in state:

| State Object         | Description                           | Key                                                          | Value             | Store |
| -------------------- | ------------------------------------- | ------------------------------------------------------------ | ----------------- | ----- |
| `Revenue`            | Fee split bytecode                    | `[]byte{1} + []byte(contract_address)`                       | `[]byte{revenue}` | KV    |
| `DeployerRevenues`   | Contract by deployer address bytecode | `[]byte{2} + []byte(deployer_address) + []byte(contract_address)` | `[]byte{1}`       | KV    |
| `WithdrawerRevenues` | Contract by withdraw address bytecode | `[]byte{3} + []byte(withdraw_address) + []byte(contract_address)` | `[]byte{1}`       | KV    |

#### Revenue[](https://docs.evmos.org/protocol/modules/revenue#revenue-1)

A Revenue defines an instance that organizes fee distribution conditions for the owner of a given smart contract

```go
type Revenue struct {
    // hex address of registered contract
    ContractAddress string `protobuf:"bytes,1,opt,name=contract_address,json=contractAddress,proto3" json:"contract_address,omitempty"`
    // bech32 address of contract deployer
    DeployerAddress string `protobuf:"bytes,2,opt,name=deployer_address,json=deployerAddress,proto3" json:"deployer_address,omitempty"`
    // bech32 address of account receiving the transaction fees it defaults to
    // deployer_address
    WithdrawerAddress string `protobuf:"bytes,3,opt,name=withdrawer_address,json=withdrawerAddress,proto3" json:"withdrawer_address,omitempty"`
}
```



#### ContractAddress[](https://docs.evmos.org/protocol/modules/revenue#contractaddress)

`ContractAddress` defines the contract address that has been registered for fee distribution.

#### DeployerAddress[](https://docs.evmos.org/protocol/modules/revenue#deployeraddress)

A `DeployerAddress` is the EOA address for a registered contract.

#### WithdrawerAddress[](https://docs.evmos.org/protocol/modules/revenue#withdraweraddress)

The `WithdrawerAddress` is the address that receives transaction fees for a registered contract.

### Genesis State[](https://docs.evmos.org/protocol/modules/revenue#genesis-state)

The `x/revenue` module's `GenesisState` defines the state necessary for initializing the chain from a previous exported height. It contains the module parameters and the revenues for registered contracts:

```go
// GenesisState defines the module's genesis state.
type GenesisState struct {
    // module parameters
    Params Params `protobuf:"bytes,1,opt,name=params,proto3" json:"params"`
    // active registered contracts for fee distribution
    Revenues []Revenue `protobuf:"bytes,2,rep,name=revenues,json=revenues,proto3" json:"revenues"`
}
```



## State Transitions[](https://docs.evmos.org/protocol/modules/revenue#state-transitions)

The `x/revenue` module allows for three types of state transitions: `RegisterRevenue`, `UpdateRevenue` and `CancelRevenue`. The logic for distributing transaction fees is handled through [Hooks](https://docs.evmos.org/protocol/modules/revenue#hooks).

#### Register Fee Split[](https://docs.evmos.org/protocol/modules/revenue#register-fee-split)

A developer registers a contract for receiving transaction fees, defining the contract address, an array of nonces for [address derivation](https://docs.evmos.org/protocol/modules/revenue#concepts) and an optional withdraw address for receiving fees. If the withdraw address is not set, the fees are sent to the deployer address by default.

1. User submits a `RegisterRevenue` to register a contract address, along with a withdraw address that they would like to receive the fees to
2. Check if the following conditions pass:
   1. `x/revenue` module is enabled
   2. the contract was not previously registered
   3. deployer has a valid account (it has done at least one transaction) and is not a smart contract
   4. an account corresponding to the contract address exists, with a non-empty bytecode
   5. contract address can be derived from the deployer’s address and provided nonces using the `CREATE` operation
   6. contract is already deployed
3. Store an instance of the provided fee.

All transactions sent to the registered contract occurring after registration will have their fees distributed to the developer, according to the global `DeveloperShares` parameter.

#### Update Fee Split[](https://docs.evmos.org/protocol/modules/revenue#update-fee-split)

A developer updates the withdraw address for a registered contract, defining the contract address and the new withdraw address.

1. User submits a `UpdateRevenue`
2. Check if the following conditions pass:
   1. `x/revenue` module is enabled
   2. the contract is registered
   3. the signer of the transaction is the same as the contract deployer
3. Update the fee with the new withdraw address. Note that if withdraw address is empty or the same as deployer address, then the withdraw address is set to `""`.

After this update, the developer receives the fees on the new withdraw address.

#### Cancel Fee Split[](https://docs.evmos.org/protocol/modules/revenue#cancel-fee-split)

A developer cancels receiving fees for a registered contract, defining the contract address.

1. User submits a `CancelRevenue`
2. Check if the following conditions pass:
   1. `x/revenue` module is enabled
   2. the contract is registered
   3. the signer of the transaction is the same as the contract deployer
3. Remove fee from storage

The developer no longer receives fees from transactions sent to this contract.

## Transactions[](https://docs.evmos.org/protocol/modules/revenue#transactions)

This section defines the `sdk.Msg` concrete types that result in the state transitions defined on the previous section.

### `MsgRegisterRevenue`[](https://docs.evmos.org/protocol/modules/revenue#msgregisterrevenue)

Defines a transaction signed by a developer to register a contract for transaction fee distribution. The sender must be an EOA that corresponds to the contract deployer address.

```go
type MsgRegisterRevenue struct {
    // contract hex address
    ContractAddress string `protobuf:"bytes,1,opt,name=contract_address,json=contractAddress,proto3" json:"contract_address,omitempty"`
    // bech32 address of message sender, must be the same as the origin EOA
    // sending the transaction which deploys the contract
    DeployerAddress string `protobuf:"bytes,2,opt,name=deployer_address,json=deployerAddress,proto3" json:"deployer_address,omitempty"`
    // bech32 address of account receiving the transaction fees
    WithdrawerAddress string `protobuf:"bytes,3,opt,name=withdraw_address,json=withdrawerAddress,proto3" json:"withdraw_address,omitempty"`
    // array of nonces from the address path, where the last nonce is
    // the nonce that determines the contract's address - it can be an EOA nonce
    // or a factory contract nonce
    Nonces []uint64 `protobuf:"varint,4,rep,packed,name=nonces,proto3" json:"nonces,omitempty"`
}
```



The message content stateless validation fails if:

- Contract hex address is invalid
- Contract hex address is zero
- Deployer bech32 address is invalid
- Withdraw bech32 address is invalid
- Nonces array is empty

### `MsgUpdateRevenue`[](https://docs.evmos.org/protocol/modules/revenue#msgupdaterevenue)

Defines a transaction signed by a developer to update the withdraw address of a contract registered for transaction fee distribution. The sender must be an EOA that corresponds to the contract deployer address.

```go
type MsgUpdateRevenue struct {
    // contract hex address
    ContractAddress string `protobuf:"bytes,1,opt,name=contract_address,json=contractAddress,proto3" json:"contract_address,omitempty"`
    // deployer bech32 address
    DeployerAddress string `protobuf:"bytes,2,opt,name=deployer_address,json=deployerAddress,proto3" json:"deployer_address,omitempty"`
    // new withdraw bech32 address for receiving the transaction fees
    WithdrawerAddress string `protobuf:"bytes,3,opt,name=withdraw_address,json=withdrawerAddress,proto3" json:"withdraw_address,omitempty"`
}
```



The message content stateless validation fails if:

- Contract hex address is invalid
- Contract hex address is zero
- Deployer bech32 address is invalid
- Withdraw bech32 address is invalid
- Withdraw bech32 address is same as deployer address

### `MsgCancelRevenue`[](https://docs.evmos.org/protocol/modules/revenue#msgcancelrevenue)

Defines a transaction signed by a developer to remove the information for a registered contract. Transaction fees will no longer be distributed to the developer, for this smart contract. The sender must be an EOA that corresponds to the contract deployer address.

```go
type MsgCancelRevenue struct {
    // contract hex address
    ContractAddress string `protobuf:"bytes,1,opt,name=contract_address,json=contractAddress,proto3" json:"contract_address,omitempty"`
    // deployer bech32 address
    DeployerAddress string `protobuf:"bytes,2,opt,name=deployer_address,json=deployerAddress,proto3" json:"deployer_address,omitempty"`
}
```



The message content stateless validation fails if:

- Contract hex address is invalid
- Contract hex address is zero
- Deployer bech32 address is invalid

## Hooks[](https://docs.evmos.org/protocol/modules/revenue#hooks)

The fees module implements one transaction hook from the `x/evm` module in order to distribute fees between developers and validators.

### EVM Hook[](https://docs.evmos.org/protocol/modules/revenue#evm-hook)

A [`PostTxProcessing` EVM hook](https://docs.evmos.org/protocol/modules/evm#hooks) executes custom logic after each successful EVM transaction. All fees paid by a user for transaction execution are sent to the `FeeCollector` module account during the `AnteHandler` execution before being distributed to developers and validators.

If the `x/revenue` module is disabled or the EVM transaction targets an unregistered contract, the EVM hook returns `nil`, without performing any actions. In this case, 100% of the transaction fees remain in the `FeeCollector` module, to be distributed to the block proposer.

If the `x/revenue` module is enabled and a EVM transaction targets a registered contract, the EVM hook sends a percentage of the transaction fees (paid by the user) to the withdraw address set for that contract, or to the contract deployer.

1. User submits EVM transaction (`MsgEthereumTx`) to a smart contract and transaction is executed successfully

2. Check if

   - fees module is enabled
   - smart contract is registered to receive fees

3. Calculate developer fees according to the `DeveloperShares` parameter. The initial transaction message includes the gas price paid by the user and the transaction receipt, which includes the gas used by the transaction.

   ```go
    devFees := receipt.GasUsed * msg.GasPrice * params.DeveloperShares
   ```

   

4. Transfer developer fee from the `FeeCollector` (Cosmos SDK `auth` module account) to the registered withdraw address for that contract. If there is no withdraw address, fees are sent to contract deployer's address.

5. Distribute the remaining amount in the `FeeCollector` to validators according to the [SDK Distribution Scheme](https://docs.cosmos.network/main/modules/distribution#the-distribution-scheme).

## Events[](https://docs.evmos.org/protocol/modules/revenue#events)

The `x/revenue` module emits the following events:

### Register Fee Split[](https://docs.evmos.org/protocol/modules/revenue#register-fee-split-1)

| Type               | Attribute Key          | Attribute Value           |
| ------------------ | ---------------------- | ------------------------- |
| `register_revenue` | `"contract"`           | `{msg.ContractAddress}`   |
| `register_revenue` | `"sender"`             | `{msg.DeployerAddress}`   |
| `register_revenue` | `"withdrawer_address"` | `{msg.WithdrawerAddress}` |

### Update Fee Split[](https://docs.evmos.org/protocol/modules/revenue#update-fee-split-1)

| Type             | Attribute Key          | Attribute Value           |
| ---------------- | ---------------------- | ------------------------- |
| `update_revenue` | `"contract"`           | `{msg.ContractAddress}`   |
| `update_revenue` | `"sender"`             | `{msg.DeployerAddress}`   |
| `update_revenue` | `"withdrawer_address"` | `{msg.WithdrawerAddress}` |

### Cancel Fee Split[](https://docs.evmos.org/protocol/modules/revenue#cancel-fee-split-1)

| Type             | Attribute Key | Attribute Value         |
| ---------------- | ------------- | ----------------------- |
| `cancel_revenue` | `"contract"`  | `{msg.ContractAddress}` |
| `cancel_revenue` | `"sender"`    | `{msg.DeployerAddress}` |

## Parameters[](https://docs.evmos.org/protocol/modules/revenue#parameters)

The fees module contains the following parameters:

| Key                        | Type    | Default Value |
| -------------------------- | ------- | ------------- |
| `EnableRevenue`            | bool    | `true`        |
| `DeveloperShares`          | sdk.Dec | `50%`         |
| `AddrDerivationCostCreate` | uint64  | `50`          |

### Enable Revenue Module[](https://docs.evmos.org/protocol/modules/revenue#enable-revenue-module)

The `EnableRevenue` parameter toggles all state transitions in the module. When the parameter is disabled, it will prevent any transaction fees from being distributed to contract deployers and it will disallow contract registrations, updates or cancellations.

#### Developer Shares Amount[](https://docs.evmos.org/protocol/modules/revenue#developer-shares-amount)

The `DeveloperShares` parameter is the percentage of transaction fees that is sent to the contract deployers.

#### Address Derivation Cost with CREATE opcode[](https://docs.evmos.org/protocol/modules/revenue#address-derivation-cost-with-create-opcode)

The `AddrDerivationCostCreate` parameter is the gas value charged for performing an address derivation in the contract registration process. A flat gas fee is charged for each address derivation iteration. We allow a maximum number of 20 iterations, and therefore a maximum number of 20 nonces can be given for deriving the smart contract address from the deployer's address.

## Clients[](https://docs.evmos.org/protocol/modules/revenue#clients)

### CLI[](https://docs.evmos.org/protocol/modules/revenue#cli)

Find below a list of  `evmosd` commands added with the `x/revenue` module. You can obtain the full list by using the `evmosd -h` command. A CLI command can look like this:

```bash
evmosd query revenue params
```



#### Queries[](https://docs.evmos.org/protocol/modules/revenue#queries)

| Command           | Subcommand             | Description                            |
| ----------------- | ---------------------- | -------------------------------------- |
| `query` `revenue` | `params`               | Get revenue params                     |
| `query` `revenue` | `contract`             | Get the revenue for a given contract   |
| `query` `revenue` | `contracts`            | Get all revenues                       |
| `query` `revenue` | `deployer-contracts`   | Get all revenues of a given deployer   |
| `query` `revenue` | `withdrawer-contracts` | Get all revenues of a given withdrawer |

#### Transactions[](https://docs.evmos.org/protocol/modules/revenue#transactions-1)

| Command        | Subcommand | Description                                |
| -------------- | ---------- | ------------------------------------------ |
| `tx` `revenue` | `register` | Register a contract for receiving revenue  |
| `tx` `revenue` | `update`   | Update the withdraw address for a contract |
| `tx` `revenue` | `cancel`   | Remove the revenue for a contract          |