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

