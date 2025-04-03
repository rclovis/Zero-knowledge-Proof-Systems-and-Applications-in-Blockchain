### Overview
The `Zeto_Anon` contract is a privacy-focused fungible token implementation using zero-knowledge proofs (ZKPs) to anonymize transactions. It leverages the **UTXO (Unspent Transaction Output)** model (similar to Bitcoin) with cryptographic commitments and ZK-SNARKs (Groth16) for privacy.

---
### Experience
**Alice** wants to send 10 zeto tokens to **Bob** 
#### 1. Setup (Pre-Transaction)
- **Alice** and **Bob** generate **BabyJubjub** key pairs:
	- Alice: `(privKey_A, pubKey_A)`
	- Bob: `(privKey_B, pubKey_B)`
- **Alice** has existing UTXOs (UTXO worth 15 tokens) stored as a commitment:
	- `commitment_A = hash(value=15, salt_A, pubKey_A)`
#### 2. Alice Prepares the Transaction
Alice creates:
- **Inputs**: Spends her UTXO (`commitment_A`).
- **Outputs**:
	- **Bob’s UTXO**: `commitment_B = hash(value=10, salt_B, pubKey_B)` (sent to Bob).
	- **Change UTXO**: `commitment_A_change = hash(value=5, salt_A_new, pubKey_A)` (returned to herself).
- **Zero-Knowledge Proof (ZKP)**:
	- Proves:
	    1. She knows `(value=15, salt_A, privKey_A)` for `commitment_A`.
	    2. **Mass conservation**: `15 (input) = 10 (to Bob) + 5 (change)`.
	    3. Outputs are valid (range-checked, correct hashes).
#### 3. Alice Submits the Transaction
- Calls `transfer()` on `Zeto_Anon` with:
```solidity
inputs = [commitment_A],
outputs = [commitment_B, commitment_A_change],
proof = <Groth16 ZKP>,
```
#### 4. On-Chain Verification
1. Contract Checks:
	- Input UTXO (`commitment_A`) exists and is unspent.
	- Output UTXOs (`commitment_B`, `commitment_A_change`) are new.
2. Proof Verification:
	- `Groth16Verifier_Anon` validates the ZKP:
		- Correctness of commitments.
		- No double-spending or inflation.

![[Anon circuit.canvas|Anon circuit]]

3. State Update:
	- Marks `commitment_A` as spent.
	- Adds `commitment_B` (Bob’s UTXO) and `commitment_A_change` (Alice’s change) to the ledger.
#### 5. Bob Accesses His Funds
- **Bob** sees no on-chain link to Alice (only `commitment_B` exists).
- To spend his UTXO later, Bob:
- Uses his `privKey_B` to generate a ZKP proving ownership of `commitment_B`.

---
### Specificity
1. Anonymity:
	- No on-chain link between Alice and Bob.
	- External observers only see opaque commitments (e.g., `hash(10, salt_B, pubKey_B)`).
2. No Traceability:
	- Even if Bob’s `pubKey_B` is known, the salt (`salt_B`) hides the UTXO’s uniqueness.
3. Cryptographic Guarantees:
	- **Soundness**: ZKP ensures Alice can’t cheat (e.g., create tokens from nothing).
	- **Unlinkability**: Cannot correlate `commitment_A` and `commitment_B`.