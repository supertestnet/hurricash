# Hurricash
It's almost like a coinpool except everyone has to have the same departing when they exit

# Video demo
https://github.com/user-attachments/assets/a9caef5e-23a5-42dc-9687-6c0bd5f5bce8

# Advantages of Hurricash

- it doesn’t need a soft fork
- as the number of users increases, the amount of cpu usage needed to create the pool grows with a low exponent -- O(n**3) -- rather than O(n!), which was the state of the art before this (see, for example, [Stupot](https://github.com/stutxo/op_ctv_payment_pool))
- users can exit in any order, at any time, without coordinating with other users
- unlike Ark, it does not need a central coordinator
- unlike Ark, pooled funds do not have a timelock on them -- users can stay in the pool indefinitely

# Disadvantages of Hurricash

- everyone has to deposit the same amount to the pool
- everyone has to exit with the same amount
- any payments made in between are deducted from *everyone's* balance -- so everyone pays
- it is an "interactive" coinpool -- meaning you can't pay anyone if any pool member is offline or unwilling to cosign your payment (though you *can* just leave and then make your payment with regular bitcoin)

# How it works

## Coinjoin

Have n people prepare and sign a coinjoin that funds an n of n multisig with Q sats apiece. It is important that Q be the same for every user and that every user has one of the keys to the multisig. In the "happy path," the money will stay in this multisig until every user is done using it and they perform a cooperative exit together. Everything else on this page (starting with Rounds) is part of the "sad path" and is only used if someone wants to exit unilaterally (i.e. without needing to cooperate with any other user).

## Rounds

Before the users share their coinjoin sigs with one another, they generate n presigned transactions, and each user gets a copy of each transaction. Each presigned transaction defines a “round” (e.g. round 1, round 2...round n) containing two transactions. The first transaction uses the pooled funds to create two outputs: one is a utxo worth Q sats, and the remainder, if any, returns to the multisig in a second output, ready for use in the next round, if any. The second transaction is described below in the section `Midstate` and creates the withdrawing user's final state.

## Midstate

In each round, the Q-sat utxo created in that round is locked to a “midstate address” with n script paths. Each script path lets a different user spend that Q-sat utxo, but only with n-of-n signatures (i.e. a signature from every other party; these signatures are also generated before anyone deposits money into the multisig). This mechanism allows any user to withdraw from any midstate, in any order, in any round.

## Bonds

To withdraw from the midstate address, a user has to supply the n-of-n signatures for their script path. The n-of-n sigs are signed with sighash_all | anyone_can_pay and they force the user to do two things: (1) the user must move the money to an address that they must withdraw it from within 1008 blocks in a transaction that reveals a “withdrawal secret” – a different one in each round for each user, otherwise anyone can spend it. (2) the user must fund a “fidelity bond” that is worth 2 times the value of Q.

## Penalty mechanism

The fidelity bond’s script has two spend paths. One is timelocked for 2016 blocks. After that period elapses, anyone can spend the withdrawer’s bond, but only if they know *two* of the withdrawer’s withdrawal secrets. If the withdrawer tries to steal from the pool by withdrawing twice (i.e. in two different rounds), they must disclose a different withdrawal secret each time, and thus they will almost certainly lose Q sats as long as anyone is paying attention to the theft attempt. Miners will presumably get the funds, because someone will prepare a transaction during the 2016 block period that spends the entire value of the bond as fees.

This mechanism disincentivizes theft attempts game-theoretically: the thief stands to *lose* Q instead of *gaining* Q, so they will probably not do it. Assuming the user has *not* tried to withdraw twice, no one could use the first script path to spend his or her funds, so the user can use the other spend path. It stipulates that after a period of 2026 blocks, the user can collect his or her bond and all is well.

