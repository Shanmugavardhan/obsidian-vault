# MultiversX Smart Contract Lifecycle

A comprehensive analysis of the smart contract lifecycle from development to execution on the MultiversX blockchain.

## Overview of the Contract Lifecycle

The journey of a smart contract WASM binary follows a strict pipeline:
1. **Build Phase (`mx-sdk-rs`)**: Rust source code is compiled into optimized WASM and packed into a deployable format.
2. **Deployment Phase (`mxpy`/`mx-sdk-js`)**: The WASM bytes are embedded into the `data` field of a transaction and broadcast to the network.
3. **Storage Phase (`mx-chain-go`)**: The node receives the transaction, calculates the contract's new address, and stores the WASM bytes in the **Patricia-Merkle Trie**, keyed by the bytecode's `codeHash`.
4. **Execution Phase (`mx-chain-vm-go`)**: Upon a contract call, the Virtual Machine (VM) retrieves the WASM from the state trie via a blockchain hook and executes it using the Wasmer engine.

---

## 1. Build Phase: From `.rs` to `.wasm`

**Repository**: `mx-sdk-rs`

- **Entry Point**: `sc-meta all build` command
- **Compilation**: Rust source code compiled to WASM using `cargo build --target=wasm32-unknown-unknown --release`
- **Optimization**: `wasm-opt` applies size and performance optimizations
- **Packaging**: Creates `.mxsc.json` file containing hex-encoded WASM bytecode
- **Output**: Deployable contract artifact in `output/` directory

## 2. Deployment Phase: Transaction Delivery

**Repository**: `mx-sdk-py-cli`, `mx-sdk-js`

- **Entry Point**: `mxpy contract deploy` or SDK deployment functions
- **Transaction Creation**: WASM bytecode embedded in transaction `data` field
- **Arguments**: Constructor arguments appended to bytecode
- **Signing**: Transaction signed with deployer's private key
- **Broadcasting**: Sent to network via `/transaction/send` REST endpoint
- **Address Calculation**: Contract address computed deterministically from deployer address and nonce

## 3. Ingestion & Storage: The State Trie

**Repository**: `mx-chain-go`

- **Entry Point**: Smart contract processor handles deployment transactions
- **WASM Extraction**: Transaction data parsed to extract contract bytecode
- **VM Validation**: `init` function executed to validate deployment
- **Code Storage**: WASM bytecode stored in Patricia-Merkle Trie
- **Code Hash**: SHA256 hash of bytecode used as storage key
- **Account Update**: Contract account stores only the 32-byte `codeHash` reference
- **State Persistence**: Account state committed to blockchain state

## 4. Execution Phase: VM Runtime

**Repository**: `mx-chain-vm-go`

- **Entry Point**: Smart contract call transaction processed
- **VM Handover**: Control transferred to VM for execution
- **Code Retrieval**: VM uses `BlockchainHook.GetCode()` to fetch WASM from state trie
- **Compilation**: WASM compiled to native machine code via Wasmer engine
- **Execution**: Target function executed with provided arguments
- **State Access**: VM can read/write contract storage via blockchain hooks
- **Gas Metering**: Execution monitored for gas consumption
- **Result**: VM output returned to blockchain for state updates

## 5. Integration: Repository Interconnections

### Core Repository Dependencies

**Development Stack:**
- `mx-sdk-rs` → Provides smart contract framework and build tools
- `mx-sdk-py-cli` → CLI tool for deployment and interaction
- `mx-sdk-js` → JavaScript SDK for web applications

**Blockchain Infrastructure:**
- `mx-chain-go` → Main blockchain node implementation
- `mx-chain-vm-go` → WebAssembly virtual machine
- `mx-chain-core-go` → Core blockchain components

### Integration Flow

1. **Development to Deployment**:
   - `mx-sdk-rs` builds contracts → `.mxsc.json` artifacts
   - `mx-sdk-py-cli` reads artifacts → creates deployment transactions
   - SDKs broadcast transactions → `mx-chain-go` nodes

2. **Node Processing**:
   - `mx-chain-go` receives transactions → validates and processes
   - Smart contract processor → extracts WASM bytecode
   - State management → stores code in Patricia-Merkle Trie

3. **Execution Integration**:
   - `mx-chain-go` → delegates execution to `mx-chain-vm-go`
   - `BlockchainHook` interface → bridges node state with VM
   - VM retrieves code → executes WASM → returns results

### Key Integration Points

**Blockchain Hook Interface**:
- `GetCode()` - Retrieves contract bytecode from state
- `GetStorageData()` - Accesses contract storage
- `GetUserAccount()` - Fetches account information
- `ProcessBuiltInFunction()` - Executes built-in functions

**Transaction Processing Pipeline**:
1. SDK creates transaction with WASM data
2. Node validates transaction format and signature
3. Smart contract processor extracts and validates bytecode
4. VM executes init/constructor function
5. State updated with contract account and code storage

## Data Transformations Summary

| Phase | Data Format | Location | Key Component |
|-------|-------------|----------|---------------|
| **Build** | `.wasm` Binary | File System | `sc-meta` |
| **Package** | Hex String | `.mxsc.json` | Build system |
| **Transit** | Transaction Data | Network | SDKs |
| **Storage** | Binary Code | Patricia-Merkle Trie | `mx-chain-go` |
| **Execution** | Machine Code | VM Memory | `mx-chain-vm-go` |

**Key Insight**: No `.wasm` files exist on blockchain nodes. The WASM bytecode is stored as state data in the Patricia-Merkle Trie and retrieved by the VM only during contract execution.