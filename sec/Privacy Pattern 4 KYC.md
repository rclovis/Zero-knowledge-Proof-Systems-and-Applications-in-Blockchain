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