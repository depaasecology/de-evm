<!--
parent:
  order: false
-->

<div align="center">
  <h1> DE Evmos </h1>
</div>

<div align="center">
  <a href="https://github.com/depaasecology/de-evm/blob/main/LICENSE">
    <img alt="License: Apache-2.0" src="https://img.shields.io/github/license/tharsis/evmos.svg" />
  </a>
  <a href="https://go.dev/">
    <img alt="GoDoc" src="https://godoc.org/github.com/evmos/evmos?status.svg" />
  </a>
  <a href="https://goreportcard.com/">
    <img alt="Go report card" src="https://goreportcard.com/badge/github.com/evmos/evmos"/>
  </a>
  <a href="https://bestpractices.coreinfrastructure.org/projects/5018">
    <img alt="Lines of code" src="https://img.shields.io/tokei/lines/github/tharsis/evmos">
  </a>
</div>
<div align="center">
  <a href="https://github.com/depaasecology/de-evm">
    <img alt="Lint Status" src="https://github.com/evmos/evmos/actions/workflows/lint.yml/badge.svg?branch=main" />
  </a>
  <a href="http://depaas.de/#/index">
    <img alt="Code Coverage" src="https://codecov.io/gh/evmos/evmos/branch/main/graph/badge.svg" />
  </a>
  </a>
</div>

Evmos is a scalable, high-throughput Proof-of-Stake blockchain
that is fully compatible and interoperable with Ethereum.
It's built using the [Cosmos SDK](https://github.com/cosmos/cosmos-sdk/)
which runs on top of the [Tendermint Core](https://github.com/tendermint/tendermint) consensus engine.

## Quick Start

To learn how Evmos works from a high-level perspective,
go to the [Protocol Overview](/docs/) section of the documentation.
You can also check the instructions to [Run a Node](/docs/cli/README.md).

## Documentation

Our documentation is hosted in a [separate repository](/docs/) and can be found at [depaas](http://depaas.de/#/index).
Head over there and check it out.

## Installation

For prerequisites and detailed build instructions
please read the [Installation](/docs/cli/single-node.md) instructions.
Once the dependencies are installed, run:

```bash
make install
```

Or check out the latest [release](https://github.com/depaasecology/de-evm).

## Licensing

Starting from April 21st, 2023, the Evmos repository will update its License
from GNU Lesser General Public License v3.0 (LGPLv3) to Evmos Non-Commercial
License 1.0 (ENCL-1.0). This license applies to all software released from Evmos
version 13 or later, except for specific files, as follows, which will continue
to be licensed under LGPLv3:

- `x/revenue/v1/` (all files in this folder)
- `x/claims/genesis.go`
- `x/erc20/keeper/proposals.go`
- `x/erc20/types/utils.go`

LGPLv3 will continue to apply to older versions (<v13.0.0) of the Evmos
repository. For more information see LICENSE.

### SPDX Identifier

The following header including a license identifier in SPDX short form has been added to all ENCL-1.0 files:

```go
// Copyright Tharsis Labs Ltd.(Evmos)
// SPDX-License-Identifier:ENCL-1.0(https://github.com/depaasecology/de-evm)
```

Exempted files contain the following SPDX ID:

```go
// Copyright Tharsis Labs Ltd.(Evmos)
// SPDX-License-Identifier:LGPL-3.0-only
```

### License FAQ

Find below an overview of the Permissions and Limitations of the Evmos Non-Commercial License 1.0.
For more information, check out the full ENCL-1.0 FAQ [here](/LICENSE_FAQ.md).

| Permissions                                                                                                                                                                  | Prohibited                                                                 |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| - Private Use, including distribution and modification<br />- Commercial use on designated blockchains<br />- Commercial use with DE EVM permit (to be separately negotiated) | - Commercial use, other than on designated blockchains, without DE EVM permit |
