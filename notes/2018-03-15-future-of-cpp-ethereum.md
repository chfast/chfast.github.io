# Future of Ethereum C++

## Introduction

This document is intended to open up a discussion about the future of
cpp-ethereum codebase and resources needed for this job.

In the first section we try to identify all current users of the cpp-ethereum.

In the second section the vision of the modular Ethereum client implementation
is described.

The third section is the list of possible tasks for cpp-ethereum developers.


## Current Users

### testeth

The testeth is the Ethereum testing tool that generates all JSON integration
tests. There is the ongoing effort to decouple the testeth tool from the
C++ implementation of the Ethereum client. The future testeth is to use the
JSON-RPC requests to generate tests. Any implementation will be able to be used
as the generator providing it implements a to-be-defined set of JSON-RPC
methods.

Nevertheless, the C++ client is to be the first client to implement these
JSON-RPC methods. See https://github.com/ethereum/cpp-ethereum/pull/4859.

Dimitry and Yoichi know more about this subject.

### eWASM testnet

The eWASM team is to run the first eWASM test network using cpp-ethereum.
The eWASM VM uses EVM-C API so the C++ node can be replaced with any other
node implementation that implements Client-side of the EVM-C API.
I'm working on the EVM-C support in go-ethereum.

### Solidity testing

The Solidity project uses C++ Ethereum client for running integration tests
in Continuous Integration systems. This uses a subset of the same RPC methods
required by testeth.

### Smart Contract tools

There are some external tools that use cpp-ethereum code base to deploy,
execute and analyze bytecode of smart contracts. They use the VM tracing
callback function to inspect the execution of a smart contract. I'm working
on adding tracing support to EVM-C to allow the tools to depend on a stable
API. Example: https://github.com/aarlt/soltest.

### EIP implementations

Some first implementations of EIP are based on cpp-ethereum codebase
(e.g. [EIP145]).

In general, implementing EIPs selected for a hard fork in C++ adds valuable
feedback about EIPs' spec and possible edge cases.


# Ethereum Client Modular Design

The following diagram visualizes the bundle of software components that creates
a full featured Ethereum Client. Some of these components already exist, some
are in progress, some are only concepts.

![alt text][design]

## Ethereum node

The implementation of Ethereum protocol exposing JSON-RPC server endpoint.

## ethminer

The GPU mining software. This is the community-driven project living in
[ethminer]. It communicates with mining pool and parity by 2 variants of
stratum protocol and with cpp-ethereum and geth by legacy "getwork" protocol.
It uses ethash library for validating solutions coming from GPU.

## ethash

The library implementing Ethash Proof Of Work algorithm (CPU only). The legacy
code resides in [ethash] -- currently used directly only by ethereum-js.
cpp-ethereum has its own in-source copy of the library.

## Dopple: RPC Proxy

The RPC Proxy extends Node's RPC capabilities by providing additional transport
protocols for RPC messages. The Node should only implement single transport.
This is a good candidate for a small independent project. The prototype exists
as a Python 3 script in cpp-ethereum: [jsonrpcproxy.py]. It provides HTTP
transport over Unix Socket messages. This allowed removing HTTP server from
cpp-ethereum.

## EVM-C

The C API for Ethereum Virtual Machines implementations. Compatible with eWASM.
See [evm.h].

## Signer

The tool for signing transactions. See [signer].

## P2P Daemon

In the image called "P2P Service" because this name was used by libp2p team.
At the moment only a concept of extracting the base P2P protocol
(i.e. DevP2P or libp2p) to a dedicated process. This should replace the current
P2P network stack implemented as a library. The process can be shared
(in runtime) by multiple "applications" like Ethereum, Swarm, IPFS. Also the
implementation can be shared.


# Future Works

## In progress

1. Reimplement ethash library with lazy DAG generation. To be shared by
   cpp-ethereum, ethminer, ethereum-js. Should include full test coverage
   and testing on big-endian architectures.

2. Migrate C++ EVM Interpreter to use EVM-C API. https://github.com/ethereum/cpp-ethereum/issues/4882

3. Finalize EVM-C. Some minor tweaks are needed. VM tracing to be added.
   Move to dedicated repository. https://github.com/ethereum/evmjit/issues/82

4. Add support for EVM-C in geth. https://github.com/ethereum/go-ethereum/pull/3410

5. Create new JSON test generation tool base on JSON-RPC requrest, not using
   cpp-ethereum directly. [retesteth]

6. eWASM testnet powered by cpp-ethereum nodes.

7. Change the default database implementation from leveldb to rocksdb.
   This is mostly maintenance tweak: rocksdb have Windows support upstream.

8. Snapshot sync - using Parity warp snapshots to sync with the Mainnet
   or Ropsten.

## Recommended

1. Maintain cpp-ethereum to be used for new EIP implementations. Use it for
   testing incoming hard forks.

2. Add support for the Signer to increase use cases for the Signer and increase
   its test coverage. This also opens up the possibility to remove
   cryptographic functions related to key storage management.

3. Kickoff the RPC Proxy project. This can be considered a candidate for a
   development grant or be done in cooperation with Python and/or Go teams.

4. Start making regular releases of cpp-ethereum to stabilize testing for
   solidity, testeth and eWASM projects.

## Optional

1. Optimize the database design

   It is clear by now, that with current state size old-school approach to
   storing the State Trie without pruning and without GC is not feasible
   for public chains (Mainet and pulic testnets). We need to implement some of
   the state DB refactoring and optimization ideas of go-ethereum generational
   garbage collection, or TurboGeth's generating trie on-the-fly, or both.

2. Add support for Ethereum wire protocol v63 and fast-sync.

3. Add support for Clique PoA engine (Rinkeby testnet).

4. A cross-chain/cross-shard interoperability tool that could validate the
   transactions/blocks having only the needed state trie branches instead of
   the full trie (that is, the tool gets tx/block and a set of trie nodes as an
   input, returns modified trie nodes and trie root after execution).

5. Make it easy to run a private testnet with cpp-ethereum (update docs, clean
   up config options).

6. Implement Whisper protocol in C++ (suggested by Whisper team).

7. Implement Casper.



[design]: https://gist.githubusercontent.com/chfast/142193acb3d34e78017227ba4cb42dc0/raw/a561a53359634d919a4e6274dd4fe609b72bbaca/ethereum_client_modular_design.png
[EIP145]: https://github.com/ethereum/EIPs/blob/master/EIPS/eip-145.md
[ethash]: https://github.com/ethereum/ethash
[ethminer]: https://github.com/ethereum-mining/ethminer
[evm.h]: https://github.com/ethereum/evmjit/blob/develop/include/evm.h
[jsonrpcproxy.py]: https://github.com/ethereum/cpp-ethereum/blob/develop/scripts/jsonrpcproxy.py
[retesteth]: https://github.com/ethereum/retesteth
[signer]: https://github.com/ethereum/go-ethereum/pull/16154
