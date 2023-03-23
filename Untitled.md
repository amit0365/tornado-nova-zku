Tornado cash Nova

Torndao trees- merkle trees are computed offchain and then prove validity on chain 

Subtrees are of certain length as we need to specify the length of the proof

Contracts will check the old root and new root, both as public input.

To verify the snarks, each public input is used to do one EC addition and multiplication which costs around 6000 gwei. 

   function verifyProof(
        bytes memory proof,
        uint256[7] memory input
    ) public view returns (bool) {
        uint256[8] memory p = abi.decode(proof, (uint256[8]));
        for (uint8 i = 0; i < p.length; i++) {
            // Make sure that each element in the proof is less than the prime q
            require(p[i] < PRIME_Q, "verifier-proof-element-gte-prime-q");
        }
        Pairing.G1Point memory proofA = Pairing.G1Point(p[0], p[1]);
        Pairing.G2Point memory proofB = Pairing.G2Point([p[2], p[3]], [p[4], p[5]]);
        Pairing.G1Point memory proofC = Pairing.G1Point(p[6], p[7]);

        VerifyingKey memory vk = verifyingKey();
        // Compute the linear combination vkX
        Pairing.G1Point memory vkX = vk.IC[0];
        for (uint256 i = 0; i < input.length; i++) {
            // Make sure that every input is less than the snark scalar field
            require(input[i] < SNARK_SCALAR_FIELD, "verifier-input-gte-snark-scalar-field");
            vkX = Pairing.plus(vkX, Pairing.scalarMul(vk.IC[i + 1], input[i]));
        }

However, to optimise gas prices for larger subtrees, all inputs are hashed and given as a public input to the snark

Relayers are used by clients who dont have any eth balance to pay for gas fee. They take a cut from the transaction fee and withdraw funds by the customer. They cannot change the transaction.

Deposit are done from mainnet, L1 and the funds are directed to tornadocash pool on L2, xdai by the omnibridge which requires 20 confirmation.

Shield transaction, no external eth involved i.e. no deposits and no withdrawals. These transfer is done on L2, cheap.

While withdrawing funds from tornado pool, the bridge funds the amount back into mainnet.

Thus the user never has to change network from the mainnet.

In order to protect privacy, users are advised to deposit and withdraw standard amounts to protect their anonymity.