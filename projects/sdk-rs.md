# Rust SDK
mx-sdk-rs has 6 core execution flows
```
1. General Transaction Execution Flow (SC execution)
2. Built-in Function Execution Flow (protocol operations)
3. Smart Contract Deployment Flow
4. Smart Contract Query (Read-only VM) Flow
5. Simulation / Scenario Execution Flow
6. Async Call / Callback Execution Flow
```

## 1. GENERAL TRANSACTION EXECUTION FLOW

The **General Transaction Execution Flow** is the **State Transition Function** of the blockchain.
It is the **only path** through which the blockchain's official record (who owns what) can be mutated. Every other feature in the SDK—from wallets to explorers—is simply a variation or a tool built to support this one, core flow.
### The Heart of the Blockchain
In a blockchain, nothing happens by accident. Every time you send tokens, buy an NFT, or interact with an app, you are triggering a **Transaction**. But a transaction isn't just a message; it is a request for the blockchain to change its "state" (its official record of who owns what).

This guide explains the **General Transaction Execution Flow**—the precise sequence of events that turns your request into a permanent part of history.

1. **perform_sc_call_lambda()** 
    
    ![](vscode-file://vscode-app/usr/share/antigravity/resources/app/extensions/theme-symbols/src/icons/files/rust.svg)
    
    `framework/scenario/src/scenario/run_vm/sc_call.rs:25` └─ **tx_input_from_call()** [CONSTRUCT TX INPUT] 
    
    ![](vscode-file://vscode-app/usr/share/antigravity/resources/app/extensions/theme-symbols/src/icons/files/rust.svg)
    
    `sc_call.rs:61` ↓
2. **commit_call_with_async_and_callback()** 
    
    ![](vscode-file://vscode-app/usr/share/antigravity/resources/app/extensions/theme-symbols/src/icons/files/rust.svg)
    
    `chain/vm/src/host/execution/exec_call.rs:53` ↓
3. **commit_call()** 
    
    ![](vscode-file://vscode-app/usr/share/antigravity/resources/app/extensions/theme-symbols/src/icons/files/rust.svg)
    
    `chain/vm/src/host/execution/exec_call.rs:25` ├─ **state.subtract_tx_gas()** [PRE-PAY GAS] │ 
    
    ![](vscode-file://vscode-app/usr/share/antigravity/resources/app/extensions/theme-symbols/src/icons/files/rust.svg)
    
    `exec_call.rs:34` └─ **execute_builtin_function_or_default()** (ROUTER) 
    
    ![](vscode-file://vscode-app/usr/share/antigravity/resources/app/extensions/theme-symbols/src/icons/files/rust.svg)
    
    `chain/vm/src/host/execution/exec_general_tx.rs:15` ↓
4. **execute_default()** 
    
    ![](vscode-file://vscode-app/usr/share/antigravity/resources/app/extensions/theme-symbols/src/icons/files/rust.svg)
    
    `chain/vm/src/host/execution/exec_general_tx.rs:103` ├─ **tx_cache.transfer_egld_balance()** [VALUE TRANSFER] │ 
    
    ![](vscode-file://vscode-app/usr/share/antigravity/resources/app/extensions/theme-symbols/src/icons/files/rust.svg)
    
    `exec_general_tx.rs:113` ├─ **TxContext::new()** [WORKSPACE PREP] │ 
    
    ![](vscode-file://vscode-app/usr/share/antigravity/resources/app/extensions/theme-symbols/src/icons/files/rust.svg)
    
    `exec_general_tx.rs:141` └─ **runtime.execute()** 
    
    ![](vscode-file://vscode-app/usr/share/antigravity/resources/app/extensions/theme-symbols/src/icons/files/rust.svg)
    
    `chain/vm/src/host/runtime.rs:107` ↓
5. **Runtime::execute() [WASM Handover]** 
    
    ![](vscode-file://vscode-app/usr/share/antigravity/resources/app/extensions/theme-symbols/src/icons/files/rust.svg)
    
    `runtime.rs:107` ├─ **self.set_executor_context()** [HOT-SWAP CONTEXT] │ 
    
    ![](vscode-file://vscode-app/usr/share/antigravity/resources/app/extensions/theme-symbols/src/icons/files/rust.svg)
    
    `runtime.rs:117` ├─ **self.executor.new_instance()** [LOAD WASM] │ 
    
    ![](vscode-file://vscode-app/usr/share/antigravity/resources/app/extensions/theme-symbols/src/icons/files/rust.svg)
    
    `runtime.rs:128` └─ **call_lambda.call()** [CONTRACT LOGIC RUNS] 
    
    ![](vscode-file://vscode-app/usr/share/antigravity/resources/app/extensions/theme-symbols/src/icons/files/rust.svg)
    
    `runtime.rs:133` └─ **VM Hooks Interface** (Storage, Math, Crypto) 
    
    ![](vscode-file://vscode-app/usr/share/antigravity/resources/app/extensions/theme-symbols/src/icons/files/rust.svg)
    
    `chain/vm/src/host/vm_hooks/vh_context.rs:15` ↓
6. **tx_context.into_results()** 
    
    ![](vscode-file://vscode-app/usr/share/antigravity/resources/app/extensions/theme-symbols/src/icons/files/rust.svg)
    
    `tx_context.rs:184` [EXTRACT (TxResult, BlockchainUpdate)] ↓
7. **blockchain_updates.apply(state)** 
    
    ![](vscode-file://vscode-app/usr/share/antigravity/resources/app/extensions/theme-symbols/src/icons/files/rust.svg)
    
    `exec_call.rs:41` [WRITE TO STORAGE - BLOCKCHAIN STATE UPDATED] ↓
8. **Return TxResult** 
    
    ![](vscode-file://vscode-app/usr/share/antigravity/resources/app/extensions/theme-symbols/src/icons/files/rust.svg)
    
    `exec_call.rs:47`
    
### Step 1: The Request (**Transaction Input**)

**Purpose:** Defining what you want to do.

Think of this as filling out a digital form. You aren't just saying "do something," you are providing specifics: who is sending the request, which contract you want to talk to, how much money you're attaching, and which specific "function" (recipe) you want to run.

- **Behind the scenes:** The SDK gathers your intent into a structured object called `TxInput`. You can find this definition in `chain/vm/src/host/context/tx_input.rs`.
- **The Code:**
```rust
// A simplified look at what the SDK prepares
pub struct TxInput {
    pub from: Address,
    pub to: Address,
    pub egld_value: BigUint,
    pub func_name: TxFunctionName,
    pub args: Vec<Vec<u8>>,
}
```

### Step 2: The Post Office (**Execution Router**)

**Purpose:** Directing your request to the right department.

Once your "form" (`TxInput`) is ready, it enters the **Execution Router**. The router's job is to look at the destination address and decide who should handle it. Is this a common, simple task that the blockchain already knows how to do (like a simple transfer)? Or is it a custom program written by a developer?

- **Behind the scenes:** It uses a "router" file to branch the logic.
- **The Function:** `execute_builtin_function_or_default` in `chain/vm/src/host/execution/exec_general_tx.rs:15`.

---

### Step 3: The Big Decision (**Builtin OR Contract?**)

**Purpose:** Choosing the right tool for the job.

If you are performing a standard action (like moving an ESDT token), the blockchain uses a "Builtin" function—a pre-optimized shortcut. If you are calling a custom Smart Contract, it needs to prepare a "Virtual Machine" to run that specific code.

- **Behind the scenes:** The router checks if the function name matches a known "Builtin" shortcut.
- **The Code:**
```rust
// exec_general_tx.rs:27
runtime.vm_ref.builtin_functions.execute_builtin_function_or_else(...)
```

### Step 4: Setting the Stage (**Create TxContext**)

**Purpose:** Gathering all the tools and workspace needed.

Before a program can run, it needs a workspace. We create a `TxContext`. This is like a "sandbox" that contains:

1. **TxCache:** A temporary copy of the blockchain accounts so the program can "check" balances without slowing down the real network.
2. **Managed Types:** Special high-speed memory for the program to use.
3. **Result Collector:** A blank notebook where the program will write down everything that happened.

- **The Function:** `TxContext::new(runtime, tx_input, tx_cache)` in `exec_general_tx.rs:141`.

---

### Step 5: Starting the Motor (**Runtime.execute**)

**Purpose:** Handing the request to the Virtual Machine (VM).

Now the "Engine" starts. The `runtime.execute` function hands the `TxContext` and the actual contract code (WASM) to the Virtual Machine. The VM is a safe, isolated environment where the code can run without being able to crash the rest of the blockchain.

- **The Code:**
```rust
// exec_general_tx.rs:143
let tx_context = runtime.execute(tx_context, f);
```

### Step 6: The Connection Windows (**VM Hooks**)

**Purpose:** Letting the program "talk" to the blockchain.

A smart contract is like a prisoner in a room—it can't see the outside world. **VM Hooks** are special "windows" in that room. If the contract needs to know how much money a user has, it calls a Hook. If it wants to save data, it calls a Hook.

- **Behind the scenes:** These hooks (storage, crypto, math) are defined in `chain/vm/src/host/vm_hooks/vh_context.rs`.
- **Plain English:** The contract says, "Hook, tell me the current block number," and the hook provides the answer safely.

---

### Step 7: The Logic Runs (**Contract Execution**)

**Purpose:** Running the actual instructions.

This is where the actual "math" happens. The contract code executes its logic: "If User A has 10 tokens, give them an NFT and subtract 10 tokens." It doesn't change the real blockchain yet; it just calculates what _should_ happen.

---
### Step 8: The Receipt (**TxResult**)

**Purpose:** Summarizing the outcome.

When the program finishes, it produces a `TxResult`. This is the final report. It says: "Success! I generated 2 logs, returned the value '5', and used this much gas."

- **The Code:**
```rust
// exec_general_tx.rs:145
tx_context.into_results()
```
---
### Step 9: Writing into Stone (**BlockchainUpdate**)

**Purpose:** Making the changes permanent.

Up until this final second, nothing has actually changed on the official record. The `BlockchainUpdate` contains all the _proposed_ changes. In the final step, the system takes those changes and "applies" them to the real blockchain state.

- **The Function:** `blockchain_updates.apply(state)` in `exec_call.rs:41`.

## 2. BUILT-IN FUNCTION EXECUTION FLOW
**No WASM execution happens here.**

Built-in functions are **Protocol Logic**, not **Contract Logic**. They represent the hardcoded rules of the blockchain that exist for performance and security. This is the "Express Lane" that powers 90% of simple asset transfers on the network.

In our journey through the blockchain SDK, we previously saw how Smart Contracts run like custom apps on a computer. But some tasks are so essential—like moving money or changing an owner—that the blockchain team built them directly into the "operating system" itself. These are **Built-in Functions**.

Think of them as the **Express Lane** of the blockchain: they skip the heavy "Virtual Machine" and run at the speed of the protocol.

Viewed tx_cache.rs:1-141
Ran command: `grep -r "fn transfer_esdt_balance" .`
Viewed tx_cache_balance_util.rs:1-128

### Built-in Function Execution Call Stack (ESDTTransfer)

**ESDTTransfer Call Stack**

**perform_sc_call_lambda()**
[`framework/scenario/src/scenario/run_vm/sc_call.rs:25`](file:///home/dharitri/Documents/WIP/RWA/WIP/mx-sdk-rs/framework/scenario/src/scenario/run_vm/sc_call.rs#L25)
↓

**execute_builtin_function_or_default()** (Router)
[`chain/vm/src/host/execution/exec_general_tx.rs:15`](file:///home/dharitri/Documents/WIP/RWA/WIP/mx-sdk-rs/chain/vm/src/host/execution/exec_general_tx.rs#L15)
↓

**BuiltinFunctionCall::execute_or_else()** (Detector)
[`chain/vm/src/builtin_functions/builtin_func_container.rs:85`](file:///home/dharitri/Documents/WIP/RWA/WIP/mx-sdk-rs/chain/vm/src/builtin_functions/builtin_func_container.rs#L85)
↓

**ESDTTransfer::execute()** (Built‑in Function)
[`chain/vm/src/builtin_functions/transfer/esdt_transfer_mock.rs:32`](file:///home/dharitri/Documents/WIP/RWA/WIP/mx-sdk-rs/chain/vm/src/builtin_functions/transfer/esdt_transfer_mock.rs#L32)
├─ **try_parse_input()**
│  [`esdt_transfer_mock.rs:77`](file:///home/dharitri/Documents/WIP/RWA/WIP/mx-sdk-rs/chain/vm/src/builtin_functions/transfer/esdt_transfer_mock.rs#L77)
├─ **build_log()** [GATHER DATA FOR RECEIPT]
│  [`esdt_transfer_mock.rs:55`](file:///home/dharitri/Documents/WIP/RWA/WIP/mx-sdk-rs/chain/vm/src/builtin_functions/transfer/esdt_transfer_mock.rs#L55)
└─ **execute_transfer_builtin_func()**
   [`chain/vm/src/builtin_functions/transfer/transfer_common.rs:60`](file:///home/dharitri/Documents/WIP/RWA/WIP/mx-sdk-rs/chain/vm/src/builtin_functions/transfer/transfer_common.rs#L60)
↓

**TxCache::transfer_esdt_balance()** (Protocol Logic)
[`chain/vm/src/host/context/tx_cache_balance_util.rs:105`](file:///home/dharitri/Documents/WIP/RWA/WIP/mx-sdk-rs/chain/vm/src/host/context/tx_cache_balance_util.rs#L105)
↓

**Detailed Sub‑Step: SENDER Balance Update**
[`tx_cache_balance_util.rs:43`](file:///home/dharitri/Documents/WIP/RWA/WIP/mx-sdk-rs/chain/vm/src/host/context/tx_cache_balance_util.rs#L43)
├─ **with_account_mut()** [LOAD SENDER FROM STORAGE]
│  [`tx_cache.rs:76`](file:///home/dharitri/Documents/WIP/RWA/WIP/mx-sdk-rs/chain/vm/src/host/context/tx_cache.rs#L76)
├─ **Validation Function** (Check Insufficient Funds)
│  [`tx_cache_balance_util.rs:62`](file:///home/dharitri/Documents/WIP/RWA/WIP/mx-sdk-rs/chain/vm/src/host/context/tx_cache_balance_util.rs#L62)
├─ **Computation (Subtract)**
│  [Subtracts transfer value from sender's ESDT instance balance]
└─ **Storage Write** (Cache updated)
   [`tx_cache_balance_util.rs:66`](file:///home/dharitri/Documents/WIP/RWA/WIP/mx-sdk-rs/chain/vm/src/host/context/tx_cache_balance_util.rs#L66)
↓

**Detailed Sub‑Step: RECEIVER Balance Update**
[`tx_cache_balance_util.rs:72`](file:///home/dharitri/Documents/WIP/RWA/WIP/mx-sdk-rs/chain/vm/src/host/context/tx_cache_balance_util.rs#L72)
├─ **with_account_mut()** [LOAD RECEIVER FROM STORAGE]
│  [`tx_cache.rs:76`](file:///home/dharitri/Documents/WIP/RWA/WIP/mx-sdk-rs/chain/vm/src/host/context/tx_cache.rs#L76)
├─ **Computation (Add)**
│  [Adds transfer value to receiver's ESDT instance balance]
└─ **Storage Write** (Cache updated)
   [`tx_cache_balance_util.rs:81`](file:///home/dharitri/Documents/WIP/RWA/WIP/mx-sdk-rs/chain/vm/src/host/context/tx_cache_balance_util.rs#L81)
↓

**Return TxResult with logs**
[`transfer_common.rs:90`](file:///home/dharitri/Documents/WIP/RWA/WIP/mx-sdk-rs/chain/vm/src/builtin_functions/transfer/transfer_common.rs#L90)

---

### Step 1: The Request (**Transaction Input**)

**Purpose:** Handing your ticket to the system.

Just like any other action, we start with a `TxInput`. This object holds your intent, but instead of calling a complex custom function, you use a "magic word" that the protocol recognizes, like `ESDTTransfer` or `ChangeOwner`.

- **The Code:** `TxInput` in `chain/vm/src/host/context/tx_input.rs`.

---

### Step 2: The Router (**exec_general_tx**)

**Purpose:** Deciding between the "Express Lane" or the "Local Lane."

The request hits the router (`execute_builtin_function_or_default`). Its primary job is to look at your request and ask: "Is this one of our native shortcuts?"

- **The Function:** `execute_builtin_function_or_default` in `chain/vm/src/host/execution/exec_general_tx.rs:15`.

---

### Step 3: The Detector (**Check: Builtin OR Contract?**)

**Purpose:** Identifying the shortcut.

The system looks up your function name in a Catalog. If it sees a name it knows (like `ChangeOwner`), it diverts the request away from the Smart Contract engine.

- **Behind the scenes:** The "Detector" uses a simple `match` statement to compare your function name against a list of protocol-defined constants.
- **The Code:**
```rust
// builtin_func_container.rs:119
CHANGE_OWNER_BUILTIN_FUNC_NAME => self.execute_bf(ChangeOwner, f),
```

### Step 4: The Workspace (**Create TxContext**)

**Purpose:** Preparing the playground, even for shortcuts.

Even though we are skipping the complex stuff, we still need a `TxContext`. This holds the `TxCache` (the mirror of the blockchain) so we can modify it safely.

- **The Function:** `TxContext::new(...)` in `exec_general_tx.rs:141`.

---

### Step 5: Skipping the Engine (**Bypassing Runtime.execute**)

**Purpose:** Efficiency by omission.

Here is where the Built-in flow differs drastically from Smart Contracts. **Neither `runtime.execute()` nor the WASM Virtual Machine are ever started.**

Instead of loading a "box" (Virtual Machine) to run your code, the system just runs a plain Rust function. This is why it's so much faster and cheaper.

- **Plain English:** While a Smart Contract has to "start a computer" to run its math, a Built-in function just "does the math" immediately.

---

### Step 6: Native Logic (**Replacing Contract Execution**)

**Purpose:** Executing the protocol's own rules.

The "logic" here isn't written in a contract; it is written in the SDK itself. For example, the `ChangeOwner` builtin simply updates a field in the account data.

- **The Code:**
```rust
// change_owner_mock.rs:35
tx_cache.with_account_mut(&tx_input.to, |account| {
    account.contract_owner = Some(new_owner_address);
});
```
### Step 7: The Receipt (**TxResult**)

**Purpose:** Summarizing the success.

Once the native logic finishes, it creates a `TxResult`. It looks the same as a Smart Contract result, so the rest of the blockchain doesn't have to worry about how it was made.

- **The Code:** `tx_context.into_results()` in `exec_general_tx.rs:145`.

---

### Step 8: The Final Seal (**BlockchainUpdate**)

**Purpose:** Making the changes "Law."

Finally, the updates made to the cache are committed. Whether it was a complex contract or a fast Built-in, the changes are applied to the real blockchain state the same way.

- **The Function:** `blockchain_updates.apply(state)` in `exec_call.rs:41`.


## 3. SMART CONTRACT DEPLOYMENT FLOW
In the world of blockchain, a Smart Contract starts its life as a file on a developer's computer. But to make it "live" on the network, it must go through a process called **Deployment**. This isn't just uploading a file; it is the act of creating a new digital identity and running its "setup" instructions for the very first time.

**Deployment = execution + storage initialization.**

You aren't just saving code to the blockchain; you are running that code once to "seed" the contract's initial state. A contract cannot exist on the blockchain without being "born" through this initialization flow.

---
### Step 1: The Blueprint (**Deployment Input**)

**Purpose:** Packaging the code and instructions.

Instead of sending a message to a person, you send a transaction that contains the **WASM bytecode** (the binary blueprint of your contract) and any "starting settings" (arguments) it needs.

- **Behind the scenes:** The SDK creates a `TxInput` where the "To" address is effectively blank (zero), signaling that this transaction is a request to create something new.
- **The Code:** `tx_input_from_deploy` in `framework/scenario/src/scenario/run_vm/sc_deploy.rs:58`.

---

### Step 2: The Router (**exec_create**)

**Purpose:** Handing the request to the "Creation Office."

The system identifies that this is a deployment request. Instead of the standard router, it goes to a specialized logic for building new accounts.

- **The Function:** `commit_deploy` in `chain/vm/src/host/execution/exec_create.rs:9`.

---

### Step 3: Determining the Home (**Check: Builtin OR Contract?**)

**Purpose:** Finding the new contract's address.

The blockchain uses your address and a counter (your "nonce") to mathematically calculate a unique address for the new contract. It's like the blockchain assigning a house number before the building even exists.

- **The Function:** `tx_cache.get_new_address(&tx_input.from)` in `exec_create.rs:58`.

---

### Step 4: The Workspace (**Create TxContext**)

**Purpose:** Preparing the environment for initialization.

Just like a regular call, we need a `TxContext`. However, this context is special: it’s the first time this specific address has ever been used in the system.

- **The Function:** `TxContext::new(...)` in `exec_create.rs:60`.

---

### Step 5: Building the Vault (**Create New Account**)

**Purpose:** Setting up the digital identity.

The system officially creates a new account in the `TxCache`. It establishes a place for the contract's money, its storage (data), and its code.

- **Behind the scenes:** At this split second, the account is created, and the WASM bytecode (the "blueprint") is saved into the account's record.
- **The Code:**
```rust
// tx_context.rs:172
contract_path: Some(contract_path),
```

### Step 6: The Grand Opening (**Runtime.execute**)

**Purpose:** Running the setup manual (`init`).

Every contract has an `init()` function (the "Constructor"). Now that the account exists, the Virtual Machine runs this function exactly once. It sets up the very first pieces of data—like who the "Owner" is or the starting price of a token.

- **The Function:** `runtime.execute(tx_context, f)` in `exec_create.rs:83`.

---

### Step 7: Connecting to State (**VM Hooks**)

**Purpose:** Letting the "Setup" code write data.

During the `init()` execution, the code uses **Hooks** to write the initial state to the blockchain. For example, it might save "Owner = Alice" into the contract's permanent storage.

---

### Step 8: Final Review (**TxResult**)

**Purpose:** Checking if the "birth" was successful.

If the `init()` function crashes or fails, the whole deployment is cancelled, and the account is never created. If it succeeds, we get a `TxResult` confirming the contract is ready for business.

- **The Code:** `tx_context.into_results()` in `exec_create.rs:85`.

---

### Step 9: The Final Seal (**BlockchainUpdate**)

**Purpose:** Making the new contract "Law."

The final step merges the new account, its saved code, and the data created during `init()` into the official blockchain record.

- **The Function:** `blockchain_updates.apply(state)` in `exec_create.rs:37`.