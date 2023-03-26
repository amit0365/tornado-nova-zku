Demo



Deposit are done from mainnet, L1 and the funds are directed to tornado cash instance on xdai, a layer 2 Blockchain, by the omnibridge which requires 20 confirmation. While deposition, users are automatically registered on the Tornado cash network.

In a shielded transaction, no external eth involved i.e. no deposits and no withdrawals. This transfer is done on L2, which makes them cheap. However, this requires both the users to be registered on Tornado cash. Note that shielded here means that both the recipient and the amount are kept private from an external observer.

While withdrawing funds from tornado pool, the bridge funds the amount back into mainnet. The inputs here like the balance and address are private. However, the outputs being on the ethereum mainnet are public. This is why users are advised to withdraw standard amounts to help them blend among other transactions and thus preserve their privacy. Note xdai sponsors withdrawal for 0.05 eth.

The user design is elegant in the sense that the client never has to change network from the mainnet.

The proof generation takes place in the browser. Tornado cash has a relatively smaller circuit with 30k constraints which takes about 5-10 seconds to compile using Web Assembly. This wasm file is embedded in the static UI of the Tornado cash smart contract.

The transaction data gets fetched from the ethereum node. This downloads all the events linked to the smart contract, builds the merkle tree. The generated transaction is then sent to the relayer.

Relayers are used by clients who don't have any eth balance to pay for gas fee. The nodes in th relayer take a cut from the transaction fee and withdraw funds for the customer, preserving their anonymity. They cannot change the transaction.

```js
#show code for alice bob test
``` 

