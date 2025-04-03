## Privacy Pattern 1 : Zeto Anon
### Overview
The `Zeto_Anon` contract is a privacy-focused fungible token implementation using zero-knowledge proofs (ZKPs) to anonymize transactions. It leverages the **UTXO (Unspent Transaction Output)** model (similar to Bitcoin) with cryptographic commitments and ZK-SNARKs (Groth16) for privacy.

---
### Experience
**Alice** wants to send 10 zeto tokens to **Bob** 
#### 1. Setup (Pre-Transaction)
- **Alice** and **Bob** generate **BabyJubjub** key pairs:
  - Alice: `(privKey_A, pubKey_A)`
  - Bob: `(privKey_B, pubKey_B)`
- **Alice** has existing UTXOs (e.g., a UTXO worth **15 tokens**) stored as a *commitment*:
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
## Privacy Pattern 2 : Zeto Enc
### Overview
The `Zeto_AnonEnc` contract extends the privacy features of the previous `Zeto_Anon` by adding **encryption** to protect UTXO data for receivers. It combines **zero-knowledge proofs (ZKPs)** with **Elliptic Curve Diffie-Hellman (ECDH)** encryption.

--- 
### Experience
**Alice** wants to send 10 zeto tokens to **Bob** 
#### 1. Setup
- **Alice** (sender) and **Bob** (receiver) generate **BabyJubjub key pairs**:
	- Alice: `(privKey_A, pubKey_A)`
	- Bob: `(privKey_B, pubKey_B)`
- **Alice** has an input UTXO: `commitment_A = hash(15, salt_A, pubKey_A)`.
#### 2. Transaction Preparation
- **Alice** creates:
	- **Outputs**:
		- **Bob’s UTXO**: `commitment_B = hash(10, salt_B, pubKey_B)`.
		- **Change UTXO**: `commitment_A_change = hash(5, salt_new, pubKey_A)`.
- **Encrypted Data**:
	- Uses **ECDH** to compute a shared secret: `shared_secret = ECDH(privKey_A, pubKey_B)`.
	- Encrypts `(value=10, salt_B)` with `shared_secret` and a `nonce`.
	- Generates `encryptedValues` (e.g., `[encrypted_value, encrypted_salt, parity_bit]`).
- **ZKP**:
	- Proves:
		1. Valid input/output commitments.
		2. Correct encryption of `commitment_B` using `pubKey_B`.
		3. Mass conservation (`15 = 10 + 5`).
#### 3. Transaction Submission
Alice calls `transfer()` with:
```solidity
inputs = [commitment_A],
outputs = [commitment_B, commitment_A_change],
encryptionNonce = <random_nonce>,
ecdhPublicKey = pubKey_A,
encryptedValues = [encrypted_data_for_B],
proof = <Groth16_ZKP>,
```
#### 4. On-Chain Verification
1. **Contract Checks**:
	- Input UTXOs are unspent.
	- Outputs are new.
2. **Proof Verification**:
	- Verifies:
		- Correct encryption via `ecdhPublicKey` and `encryptedValues`.
		- ZKP conditions (mass conservation, valid UTXOs).

![[Anon enc circuit.canvas|Anon Enc circuit]]

3. **State Update**:
	- Marks `commitment_A` as spent.
	- Adds `commitment_B` (Bob’s) and `commitment_A_change` (Alice’s) to the ledger.
#### 5. Bob Decrypts His UTXO
- **Bob** uses his `privKey_B` and Alice’s `ecdhPublicKey` to compute `shared_secret`.
- Decrypts `encryptedValues` using `shared_secret` and `nonce`.
- Now knows `value=10` and `salt_B` to spend `commitment_B` later.

---
### Specificity

1. **Receiver Privacy**:
	- Even if `commitment_B` is public, only Bob can decrypt its value/salt.
2. **Data Availability**:
	- Encrypted data is stored on-chain, ensuring Bob can always decrypt.
3. **Replay Protection**:
	- Unique `nonce` per transaction prevents reusing encrypted data.

---

## Privacy Pattern 3 : Zeto Enc Nullifiers
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
## Privacy Pattern 4 : KYC
### Overview
The `Zeto_AnonEncNullifierKyc` contract extends the privacy-preserving UTXO model with **KYC (Know Your Customer) compliance** while maintaining anonymity. It combines:
- **Nullifiers & Sparse Merkle Trees (SMTs)**: For double-spending prevention.
- **ECDH Encryption**: For receiver confidentiality.
- **KYC Registry**: A whitelist of approved public keys stored in a Merkle tree.
- **ZK-SNARKs**: Prove transaction validity without revealing sensitive data.
---
### Transaction Flow (Alice → Bob with KYC)
#### 1. Setup
- **KYC Registration**:
	- Contract owner calls `register(pubKey_A)` and `register(pubKey_B)`.
	- Updates the `identitiesRoot` Merkle root.
- **Alice’s UTXO**:
	- `commitment_A = hash(15, salt_A, pubKey_A)`.
	- `nullifier_A = hash(commitment_A, privKey_A)`.
#### 2. Transaction Preparation
- **Alice** creates:
	- **Outputs**:
		- `commitment_B = hash(10, salt_B, pubKey_B)` (Bob’s UTXO).
		- `commitment_A_change = hash(5, salt_new, pubKey_A)` (change).
- **ZKP**:
	- Proves:
		1. `pubKey_A` and `pubKey_B` are in the KYC registry (`identitiesRoot`).
		2. `nullifier_A` is in the SMT (`root`).
		3. Valid encryption of `commitment_B`.
		4. `15 = 10 + 5`.
#### 3. Transaction Submission
Alice calls `transfer()` with:
```solidity
nullifiers = [nullifier_A],
outputs = [commitment_B, commitment_A_change],
root = current_SMT_root,
encryptedValues = [encrypted_data_for_B],
proof = <Groth16_ZKP>,
...
```
#### 4. On-Chain Verification
1. Contract Checks:
	- Validates the ZKP includes the correct `identitiesRoot` and SMT `root`.
2. Proof Validation:
	- Verifies all ZKP conditions (KYC, nullifiers, encryption, etc.).
3. State Update:
	- Adds `nullifier_A` to the SMT.
	- Adds new UTXOs to the ledger.
#### 5. Bob Decrypts His UTXO
- Uses `privKey_B` to decrypt `encryptedValues`.
---
### Security & Compliance Mechanisms
1. **KYC Enforcement**:
	- Only registered public keys can send/receive UTXOs in `transfer()`.
	- **Deposit/Withdraw**: No KYC checks (gas optimization), but non-KYC users cannot transfer tokens further.
2. **Privacy Preservation**:
	- **Anonymity**: No link between real identities and public keys.
	- **Confidentiality**: UTXO values encrypted for receivers.
3. **Integrity**:
	- ZK-SNARKs ensure all rules (mass conservation, valid SMT roots, KYC) are followed.
