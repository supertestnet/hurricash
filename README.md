# Hurricash
A bitcoin channel factory and almost-coinpool that works without a soft fork

# Advantages of Hurricash

- it doesn’t need a soft fork
- you can put your money in it without worrying that your utxos will expire (though when you *receive* a payment you *do* have a time limit to sweep it or risk your counterparty broadcasting a transaction that pays you less money)
- it scales linearly with the number of users – O(n) instead of O(n**2)
- users can exit in any order, at any time, without coordinating with other users
- it supports invisible channels (meaning you can open a channel without it appearing on the blockchain)
- it enables cheaper channel opens (this is a consequence of channels being invisible -- a batch channel open today would have many outputs, this scheme reduces the number of outputs to one, though there'd be some more if there's change)
- it enables fewer channel closure transactions (in the happy path, everyone in the multisig exits via lightning, leaving the routing node or nodes with all the keys to the multisig, so they can sweep the funds from it cooperatively)

# How it works

## Pooling the funds

### Coinjoin

Have n people prepare and sign a coinjoin that funds an n of n multisig with Q sats apiece. It is important that Q be the same for every user and that every user has one of the keys to the multisig. In the "happy path," the money will stay in this multisig until every user is done using it and they perform a cooperative exit together, described below in the section called `"Happy paths" and "sad paths"`. Everything else on this page (starting with Rounds) is part of the "sad path" and is only used if someone wants to exit unilaterally (i.e. without needing to cooperate with any other user).

### Rounds

Before the users share their coinjoin sigs with one another, they generate n presigned transactions, and each user gets a copy of each transaction. Each presigned transaction defines a “round” (e.g. round 1, round 2...round n) containing two transactions. The first transaction uses the pooled funds to create two outputs: one is a utxo worth Q sats, and the remainder, if any, returns to the multisig in a second output, ready for use in the next round, if any. The second transaction is described below in the section `Midstate` and creates the withdrawing user's final state.

### Midstate

In each round, the Q-sat utxo created in that round is locked to a “midstate address” with n script paths. Each script path lets a different user spend that Q-sat utxo, but only with n-of-n signatures (i.e. a signature from every other party; these signatures are also generated before anyone deposits money into the multisig). This mechanism allows any user to withdraw from any midstate, in any order, in any round.

### Bonds

To withdraw from the midstate address, a user has to supply the n-of-n signatures for their script path and reveal a “withdrawal secret” – a different one in each round for each user. The n-of-n sigs are signed with sighash_all | anyone_can_pay and they force the user to do two things: (1) the user must move the money to an address they chose in advance, before anyone deposited money into the multisig, and (2) the user must fund a “fidelity bond” that is worth 2 times the value of Q.

### Penalty mechanism

The fidelity bond’s script has two spend paths. One is timelocked for 2016 blocks. After that period elapses, anyone can spend the withdrawer’s bond, but only if they know *two* of the withdrawer’s withdrawal secrets. If the withdrawer tries to steal from the pool by withdrawing twice (i.e. in two different rounds), they must disclose a different withdrawal secret each time, and thus they will almost certainly lose Q sats as long as anyone is paying attention to the theft attempt. Miners will presumably get the funds, because someone will prepare a transaction during the 2016 block period that spends the entire value of the bond as fees.

This mechanism disincentivizes theft attempts game-theoretically: the thief stands to *lose* Q instead of *gaining* Q, so they will probably not do it. Assuming the user has *not* tried to withdraw twice, no one could use the first script path to spend his or her funds, so the user can use the other spend path. It stipulates that after a period of 2026 blocks, the user can collect his or her bond and all is well.

## Adding channels to the pool

### Enabling a one-time off-chain transfer

A user can transfer an entire pooled utxo to one other person (selected in advance) if he makes the withdrawal address an htlc to which he knows the preimage. If he *doesn’t* disclose the preimage, he gets his money back after 2 weeks. If he *does* disclose the preimage, his preselected recipient can take the money.

### Adding channels

The protocol described above has everyone presign *one* transaction for each user allowing them to sweep their money from the midstate of any round. That "sweep" transaction has two outputs: one puts all of that user's money in an address the user chose in advance, the other is a bond that will burn double that amount if the user tries to withdraw from the pool multiple times. To enable users to have a "channel," have every user pick a channel counterparty in advance, and have everyone in the multisig presign k transactions, each of which has *three* outputs: one gives the user some percentage of his funds, the next gives his channel counterparty the remainder, and the third is the bond.

### Enabling multiple off-chain transfers

Each of the k transactions uses a different tapleaf in the midstate, and each one distributes the funds differently to the user and his counterparty (e.g. 90/10, 80/20, 70/30, etc.). Each of these transactions is also timelocked, and the timelock gets lower and lower in each transaction that pays the user's counterparty a larger amount. A user pays their counterparty by giving him a 32 byte preimage to a hash that "guards" the tapleaf that pays him the amount the user wants him to have. With that preimage, the user's counterparty can spend from the midstate and thus pay himself the amount desired by the user. As long as each “larger” amount has a smaller timelock, the channel counterparty can broadcast whichever one gives him the most money, so long as the user gave him the preimage that "unlocks" that tapleaf.

### Connecting it to lightning (send only)

To any user's counterparty, any preimage that unlocks a tapleaf that pays him a "bigger" amount than he is currently recieving is worth the additional amount it pays him. As a result, if any user wants to pay someone over lightning, he should have his recipient create a lightning invoice locked to the preimage that pays the user's counterparty the amount closest to whatever amount the recipient wants. Then the recipient can give this invoice to the user, who gives it to their counterparty. The counterparty then knows that if he pays that invoice, he will get the preimage, and thus be able to unlock the tapleaf that pays him that amount. This does require some way for the recipient to make an invoice locked to a specific preimage, but that's fine: Blixt and Zeus both let you do this, and other wallets can add support too.

### Adding support for receiving via lightning

In each channel created via this pooling scheme, one party is always the sender and the other party is always the recipient. Which might seem to mean they are only good for sending, not receiving. But it is easy to enable receiving for *both* parties: simply have each party create *two* channels in the pool. In one, they are the sender, and in the other, they are the recipient. And that makes it work both ways. When a user wishes to receive a payment via lightning, they pick the next tapleaf that pays them the amount they want, and ask their counterparty to create an invoice locked to the preimage that "guards" that tapleaf. The counterparty then gives this invoice to the user, who gives it to whoever wants to pay them. When it is paid, the user can get the preimage either from the person who paid it or from their channel counterparty. As soon as they have it, they can unlock that tapleaf and withdraw that amount of money without help from their counterparty, so they've effectively been paid.

# "Happy paths" and "sad paths"

The hurricash scheme, as described so far, allows for the creation of off-chain channels connected to the lightning network, and from which every user can exit without help from the other users. But the "unilateral exit" mechanism takes 2 transactions (multisig -> midstate, midstate -> final state) and, since each user has *two* channels (one for sending, one for receiving), it essentially takes 4 transactions to fully exit. This is inefficient, so I call the use of the unilateral exit mechanism the "sad path."

But I think there is at least sometimes a way to do a "happy path." If a user sends their entire balance out through lightning, so that all of their money goes to their counterparty and they receive an equivalent amount via lightning, then the user can give their counterparty their private key, because they don't need it anymore.

If every user "exits" via this much cheaper mechanism, then the routing nodes who serve as channel counterparties end up with all the keys to the multisig. And since routing nodes are expected to always be online, it is more reasonable to expect them to do an interactive protocol, such as this: create a transaction that disperses all funds in the multisig to the routing nodes, and sign it once all of their users have exited via the "happy path." Then none of the channel closure transactions end up on the blockchain, and a single transaction just gives the routing nodes the money they would have exited with anyway.
