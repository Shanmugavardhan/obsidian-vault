# Nonce in Blockchain Transactions

## Definition
**Nonce** = number of transactions already sent from an account.
## Purpose
- Guarantees each transaction is unique.
- Enforces strict ordering (transaction #5 must come after #4).
- Prevents replaying old transactions (you can’t reuse the same nonce).
## Behaviour
- A nonce is **consumed once the transaction is processed**, even if execution fails.
- If a transaction is **never included on-chain** (e.g., dropped from mempool), the nonce is **not consumed** and can be reused.
## Example
- Account has `nonce = 12`. 
- You send a transaction with `nonce = 13`.
- The transaction is included in a block but fails execution.
- The chain still records nonce `13` as attempted → next transaction must use `nonce = 14`.
- If nonce `13` was never included, you can still reuse it.
## Usage in `mx-sdk-py`
```python
account.nonce = entrypoint.recall_account_nonce(account.address)
nonce = account.get_nonce_then_increment()

