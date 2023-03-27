Tornado Cash Nova

Tornado cash nova is the latest version of the protocol and is different from core in many ways.

Instead of hashing two random numbers in the commitment as in the Core version. The Nova upgrade uses three numbers: amount, public key and blinding. The blinding is a random number used instead of the secret in the previous version. 

The commitment is now computed by the hash of the three values mentioned above. Whereas the nullifier is the hash of commitment and the corresponding merkle path.


```js
/*
Utxo structure:
{
    amount,
    pubkey,
    blinding, // random number
}

commitment = hash(amount, pubKey, blinding)
nullifier = hash(commitment, merklePath, sign(privKey, commitment, merklePath))
*/
```

Unlike the core version, the user needs to generate a snark proof even while depositing some eth. The proof takes root, publicAmount and extDataHash as public inputs. extDataHash is 

UTXO contents' for both the input and output transaction is also kept private. Path indices and Path Elements are also included in the input data to specify the node path in the merkle tree.

Input Nullifier and Output Commitments are public inputs. By making them public, they are included in the transaction's proof, which is used to verify the transaction's validity.

```circom=
// Universal JoinSplit transaction with nIns inputs and 2 outputs
template Transaction(levels, nIns, nOuts, zeroLeaf) {
    signal input root;
    // extAmount = external amount used for deposits and withdrawals
    // correct extAmount range is enforced on the smart contract
    // publicAmount = extAmount - fee
    signal input publicAmount;
    signal input extDataHash;

    // data for transaction inputs
    signal         input inputNullifier[nIns];
    signal private input inAmount[nIns];
    signal private input inPrivateKey[nIns];
    signal private input inBlinding[nIns];
    signal private input inPathIndices[nIns];
    signal private input inPathElements[nIns][levels];

    // data for transaction outputs
    signal         input outputCommitment[nOuts];
    signal private input outAmount[nOuts];
    signal private input outPubkey[nOuts];
    signal private input outBlinding[nOuts];

    component inKeypair[nIns];
    component inSignature[nIns];
    component inCommitmentHasher[nIns];
    component inNullifierHasher[nIns];
    component inTree[nIns];
    component inCheckRoot[nIns];
    var sumIns = 0;
```

The proof follows the construction of the usual split joint transaction scheme. In the case of  Alice sending Bob some money, we need both their UTXO's as inputs. 

After the transaction, These leaves are marked spent in the merkle tree. This results in two different output UTXO's for Alice and Bob which are inserted into the merkle tree as new leaves.

For each input a commitment hash is computed using the UTXO. Then we check that the commitment is present in the merkle tree. However, when the input amount is 0 then we do not check the inclusion. 

This would happen when a user deposits some amount. In this case there are no inputs from the user, so contract generates two input UTXO's with zero amount and random numbers in pubkey and blinding. 

Additionally we verify the correct signature by the private key corresponding to the given public key in the UTXO.

Moreover, the nullifier hash is computed using the commitment and the merkle path. Once computed this emitted as events to be included in the list of spent nullifiers.

```circom=+
// verify correctness of transaction inputs
    for (var tx = 0; tx < nIns; tx++) {
        inKeypair[tx] = Keypair();
        inKeypair[tx].privateKey <== inPrivateKey[tx];

        inCommitmentHasher[tx] = Poseidon(3);
        inCommitmentHasher[tx].inputs[0] <== inAmount[tx];
        inCommitmentHasher[tx].inputs[1] <== inKeypair[tx].publicKey;
        inCommitmentHasher[tx].inputs[2] <== inBlinding[tx];

        inSignature[tx] = Signature();
        inSignature[tx].privateKey <== inPrivateKey[tx];
        inSignature[tx].commitment <== inCommitmentHasher[tx].out;
        inSignature[tx].merklePath <== inPathIndices[tx];

        inNullifierHasher[tx] = Poseidon(3);
        inNullifierHasher[tx].inputs[0] <== inCommitmentHasher[tx].out;
        inNullifierHasher[tx].inputs[1] <== inPathIndices[tx];
        inNullifierHasher[tx].inputs[2] <== inSignature[tx].out;
        inNullifierHasher[tx].out === inputNullifier[tx];

        inTree[tx] = MerkleProof(levels);
        inTree[tx].leaf <== inCommitmentHasher[tx].out;
        inTree[tx].pathIndices <== inPathIndices[tx];
        for (var i = 0; i < levels; i++) {
            inTree[tx].pathElements[i] <== inPathElements[tx][i];
        }

        // check merkle proof only if amount is non-zero
        inCheckRoot[tx] = ForceEqualIfEnabled();
        inCheckRoot[tx].in[0] <== root;
        inCheckRoot[tx].in[1] <== inTree[tx].root;
        inCheckRoot[tx].enabled <== inAmount[tx];

        // We don't need to range check input amounts, since all inputs are valid UTXOs that
        // were already checked as outputs in the previous transaction (or zero amount UTXOs that don't
        // need to be checked either).

        sumIns += inAmount[tx];
    }

    component outCommitmentHasher[nOuts];
    component outAmountCheck[nOuts];
    var sumOuts = 0;
```

For outputs, their commitment is checked and published as a public output. To prevent overflow, the output amount is checked to fit 248bits. Then the circuit checks for duplicates among the input nullifiers.

Let's take an example of depositing 3 eth and then withdrawing 0.5 eth. 

The user starts by deposits 3 eth as the first transaction. This generates two input UTXO's. A dummy input with zero amount and random numbers for the other two fields. The publicAmount, defined as a public input is set to 3 eth. This generates two outputs UTXO's. One of which has a zero amount and the random numbers while the other has amount equal to 3 eth and public key set to the user and a random blinding. 

The second transaction has a shielded input of 3 eth. The observer only notices some input being marked as "spent" in the merkle tree without gaining any more information. The other input is a dummy as before. 

The publicAmount will be set to -0.5 eth which is send to the withdraw address as the first output. The negative sign means that the contract pays the amount to some address. This is followed by another shielded output of 2.5 eth, which belongs to the public key of the withdrawer. Note that the latter amount is in the UTXO of th withdrawer on the Tornado cash network.

Finally, to correctly execute the transaction we need to check the following:
1) The sum of amount in inputs plus the publicAmount should be equal to the corresponding sum in the outputs. This is defined as amount variant in line 104.

2) Both the input amounts should be non-negative


```circom=+
 // verify correctness of transaction outputs
    for (var tx = 0; tx < nOuts; tx++) {
        outCommitmentHasher[tx] = Poseidon(3);
        outCommitmentHasher[tx].inputs[0] <== outAmount[tx];
        outCommitmentHasher[tx].inputs[1] <== outPubkey[tx];
        outCommitmentHasher[tx].inputs[2] <== outBlinding[tx];
        outCommitmentHasher[tx].out === outputCommitment[tx];

        // Check that amount fits into 248 bits to prevent overflow
        outAmountCheck[tx] = Num2Bits(248);
        outAmountCheck[tx].in <== outAmount[tx];

        sumOuts += outAmount[tx];
    }

    // check that there are no same nullifiers among all inputs
    component sameNullifiers[nIns * (nIns - 1) / 2];
    var index = 0;
    for (var i = 0; i < nIns - 1; i++) {
      for (var j = i + 1; j < nIns; j++) {
          sameNullifiers[index] = IsEqual();
          sameNullifiers[index].in[0] <== inputNullifier[i];
          sameNullifiers[index].in[1] <== inputNullifier[j];
          sameNullifiers[index].out === 0;
          index++;
      }
    }

    // verify amount invariant
    sumIns + publicAmount === sumOuts;

    // optional safety constraint to make sure extDataHash cannot be changed
    signal extDataSquare <== extDataHash * extDataHash;
}
```
