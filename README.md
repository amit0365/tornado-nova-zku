# Tornado Pool [![Build Status](https://github.com/tornadocash/tornado-pool/workflows/build/badge.svg)](https://github.com/tornadocash/tornado-pool/actions)

This an experimental version of tornado.cash that allows to deposit **arbitrary amounts** and make **internal(shielded) transfers**.

Other facts about this version:

1. It uses L2 (xdai). Xdai has a ETH(mainnet)<>WETH(xdai) bridge that will be used under hood.
2. Contracts will be upgradable by tornado-cash governance! xdai bridge supports transferring messages from L1 to L2 and vise versa, so community can always upgrade tornado-pool to a new version in case of an issue.
3. Since it's a beta version, deposits are limited by 1ETH. Governance can always increase the limit.
4. Withdrawal amount from pool to L1 has to be larger than 0.05 ETH to prevent spam attack on the bridge.
5. The code was [audited](./resources/Zeropool-Tornado.pool-audit.pdf) by Igor Gulamov from Zeropool.

This project was presented on LisCon 2021. [Slides](https://docs.google.com/presentation/d/1CbI6fiWvgwoD_1ahcSR62wD7V4TdSzkdL2RwAeMPagQ/edit#slide=id.gf731d8850e_0_133)


## My understanding

Torndao trees- merkle trees are computed offchain and then prove validity on chain 

Subtrees are of certain length as we need to specify the length of the proof

Contracts will check the old root and new root, both as public input.

To verify the snarks, each public input is used to do one EC addition and multiplication which costs around 6000 gwei. However, to optimise gas prices for larger subtrees, all inputs are hashed and given as a public input to the snark



Relayers are used by clients who dont have any eth balance to pay for gas fee. They take a cut from the transaction fee and withdraw funds by the customer. They cannot change the transaction.

Deposit are done from mainnet, L1 and the funds are directed to tornadocash pool on L2, xdai by the omnibridge which requires 20 confirmation.

Shield transaction, no external eth involved i.e. no deposits and no withdrawals. These transfer is done on L2, cheap.

While withdrawing funds from tornado pool, the bridge funds the amount back into mainnet.

Thus the user never has to change network from the mainnet.

In order to protect privacy, users are advised to deposit and withdraw standard amounts to protect their anonymity.


## Usage

```shell
yarn
yarn download
yarn build
yarn test
```

## Deploy

Check config.js for actual values.

With `salt` = `0x0000000000000000000000000000000000000000000000000000000047941987` addresses must be:

1. `L1Unwrapper` - `0x3F615bA21Bc6Cc5D4a6D798c5950cc5c42937fbd`
2. `TornadoPool` - `0x0CDD3705aF7979fBe80A64288Ebf8A9Fe1151cE1`

Check addresses with current config:

```shell
yarn compile
node -e 'require("./src/0_generateAddresses").generateWithLog()'
```

Deploy L1Unwrapper:

```shell
npx hardhat run scripts/deployL1Unwrapper.js --network mainnet
```

Deploy TornadoPool Upgrade:

```shell
npx hardhat run scripts/deployTornadoUpgrade.js --network xdai
```
