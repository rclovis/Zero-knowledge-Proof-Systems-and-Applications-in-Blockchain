### Overview
The `Zeto_AnonEncNullifier` contract enhances privacy and security by introducing **nullifiers** and **Sparse Merkle Trees (SMTs)** to the previous encrypted UTXO model.

---
### Experience
**Alice** wants to send 10 zeto tokens to **Bob** 
#### 1. Setup
- **Alice** (sender) and **Bob** (receiver) generate **BabyJubjub key pairs**:
	- Alice: `(privKey_A, pubKey_A)`
	- Bob: `(privKey_B, pubKey_B)`
- **Alice** has a UTXO represented by `commitment_A = hash(15, salt_A, pubKey_A)`.
- **Nullifier**: `nullifier_A = hash(commitment_A, privKey_A)` is stored in the SMT.
#### 2. Transaction Preparation
- **Alice** creates:
	- **Outputs**:
		- **Bob’s UTXO**: `commitment_B = hash(10, salt_B, pubKey_B)`.
		- **Change UTXO**: `commitment_A_change = hash(5, salt_new, pubKey_A)`.
- **Nullifiers**: `[nullifier_A]` to mark `commitment_A` as spent.
- **Encrypted Data**:
	- Uses **ECDH** to compute a shared secret: `shared_secret = ECDH(privKey_A, pubKey_B)`.
	- Encrypts `(value=10, salt_B)` with `shared_secret` and a `nonce`.
	- Generates `encryptedValues` (e.g., `[encrypted_value, encrypted_salt, parity_bit]`).
- **ZKP**:
	- Proves:
		1. `nullifier_A` exists in the SMT with root `root`.
		2. `15 = 10 + 5` (mass conservation).
		3. Valid encryption of `commitment_B`.
#### 3. Transaction Submission
Alice calls `transfer()` with:
```solidity
nullifiers = [nullifier_A],
outputs = [commitment_B, commitment_A_change],
root = current_SMT_root,
encryptionNonce = <random_nonce>,
ecdhPublicKey = pubKey_A,
encryptedValues = [encrypted_data_for_B],
proof = <Groth16_ZKP>,
```
#### 4. On-Chain Verification
1. **Contract Checks**:
	- Verifies the ZKP includes the correct SMT `root`.
2. **Proof Validation**:
	- Checks ZKP conditions (nullifiers in SMT, encryption, mass conservation)

![[Anon enc nullifiers circuit.canvas|Anon enc nullifiers circuit]]

3. **State Update**:
	- Adds `nullifier_A` to the SMT (updating the root).
	- Adds `commitment_B` and `commitment_A_change` to the ledger.
#### 5. Bob Decrypts His UTXO
- **Bob** uses his `privKey_B` and Alice’s `ecdhPublicKey` to compute `shared_secret`.
- Decrypts `encryptedValues` using `shared_secret` and `nonce`.
- Now knows `value=10` and `salt_B` to spend `commitment_B` later.

---
### Specificity
1. **Double-Spending Prevention**:
	- Nullifiers are added to the SMT; reuse is detected via Merkle inclusion proofs.
2. **Data Confidentiality**:
	- UTXO values/salts encrypted with ECDH (only receivers can decrypt).
3. **Anonymity**:
	- SMT hides which nullifier corresponds to which UTXO.
4. **Integrity**:
	- ZK-SNARKs ensure all rules (mass conservation, valid SMT root) are followed.