# SSB Playground
Create and test SSB restrictions -- bitcoin scripts that use the Sighash Single Bug to do introspection on a transaction's input count and output count and place restrictions on them

# What is this?
This is a website where you can play with the bitcoin bug commonly called the Sighash Single Bug. I've created several different types of transactions which exploit it to do a basic form of transaction introspection, and you can try out each one to see what it does.

# How can I try it?
Go here and follow the instructions: https://supertestnet.github.io/ssb_playground

# I'm confused, please explain
The Sighash Single Bug (SSB) is a bug in bitcoin's signature checking algorithm that affects certain legacy transactions, i.e. ones that don't use segwit. (In case you're wondering, "what about taproot?" taproot transactions are segwit transactions, so they are unaffected by this bug.)

The bug involves bitcoin's "sighash" function, which determines what message a user has to sign to create a valid signature for their transaction. All signatures in bitcoin are *supposed to* sign the hash -- or "sighash," as it is called -- of a truncated version of the transaction about to be broadcasted.

Typically, the truncated version includes a list of all the inputs to the transaction and all the outputs. However, wallets are *allowed* to modify what gets signed using things called "sighash flags." They have six options:

1. All inputs and all outputs (sighash_all, which is the default)
2. All inputs and one output (sighash_single)
3. All inputs and zero outputs (sighash_none)
4. One input and all outputs (sighash_all | anyone_can_pay)
5. One input and one output (sighash_single | anyone_can_pay)
6. One input and zero outputs (sighash_none | anyone_can_pay)

The Sighash Single Bug affects two of these sighash flags: sighash_single and sighash_single | anyone_can_pay. If a wallet uses either of those sighashes while signing a transaction, the wallet is *supposed to* sign one output. But which one? The answer is: it depends on the input. Every input has a field where you're supposed to place the signature for whatever bitcoin address is in that "from" position. The field you put your signature in determines the input you're signing, and the output to sign must be selected by running an algorithm that looks at the index of whatever *input* is being signed and finds a "matching output" at the same index.

So, for example, suppose you are sending "from" 3 bitcoin addresses and sending "to" 2 bitcoin addresses. The three "from" addresses are part of your transaction's "inputs," and you have to create a signature for each one. If you sign the "first" input with sighash_single, it will also sign one output: the first output. If you sign the "second" input with sighash_single, it will also sign one output: the second output.

The bug occurs if you sign the "third" input with sighash_single. It's *supposed to* sign a matching "third output," but there is none. The bug is, instead of causing the transaction to fail, a worse error happens: bitcoin expects your wallet to sign a sighash consisting of the number "01" followed by 62 zeroes. Here is the exact message ("sighash") it wants you to sign when this error happens:

```
0100000000000000000000000000000000000000000000000000000000000000
```

Weird, huh?

That strange bug causes several headaches. For example, if someone knows you have a bitcoin address containing a bunch of money, and they somehow convince you to sign that message with its private key, they can take your money. They have to manually create a bitcoin transaction that has your address in one of the inputs, e.g. in position 2, don't create an output in position 2, and add the signature they got from you in the signature field. Then they can use their own bitcoin addresses to fill any gaps in the input list, add one of their own bitcoin addresses as an output, and voila! Any money you had becomes theirs.

Wallet devs are therefore discouraged from letting users "accidentally" sign the above buggy sighash, as it can cause loss of funds. The segwit upgrade fixed this bug for all transactions that use segwit, but it's never been fixed for legacy transactions.

Recently, I realized that the bug can be exploited to do a basic form of transaction introspection. If you create a bitcoin script that "intentionally" exploits the sighash_single bug, it tells you information about the transaction that triggered it. A bitcoin script can require the user to pass in a signature that uses sighash_single and a pubkey, and check if the resulting signature is valid for the buggy sighash.

If the bug triggers, then the bitcoin script knows that the "input position" of the utxo being spent is greater than position 1, because position 1 (or position 0, if you prefer indexing from 0) *always* has a matching output -- it's a consensus rule in bitcoin that every transaction has to have at least 1 address in the outputs. So if the bug is exploited, it has to happen at an index position higher than that.

The same piece of information also tells the transaction that there is *at least* one more input than there are outputs. That is because the only way for the bug to trigger is if there is an input without a corresponding output -- so there must be more inputs than outputs.

I ended up coming up with a list of things your script can learn by checking if the exploit triggers, and here is that list:

- if the exploit triggers, then the "input position" of the utxo being spent is greater than position 1 (or 0, if you prefer indexing from 0)
- if the exploit triggers, then the number of inputs is at least 2
- if the exploit triggers, then the number of outputs is less than the number of inputs
- if the exploit does NOT trigger, then there is an output at the current input position

With that list in mind, I started playing with scripts that check if the bug triggers and either succeed or fail depending on the outcome, and figured out that I can create bitcoin addresses that use this information to place "restrictions" on the transactions that try to spend them -- like "this transaction must have more inputs than outputs" (an FTX restriction, also used in the Cleanup restriction) or "this transaction must have an output at index N" (a GTX restriction).

Then I figured out that I can combine an FTX restriction and a GTX restriction to create a *pair* of bitcoin addresses, fund them, make each one a multisig that requires *two* signatures, and publish a pair of signatures that commit to both. That way, they have to be spent together, or the published signatures are invalid.

The published signatures make both addresses "anyone_can_spend" addresses, but the FTX and GTX restrictions make it so that anyone who wants to *use* those signatures (and thus take the money) *must* create a transaction with exactly N outputs, where N is chosen by whoever originally published the signatures. I call this an ETX restriction. FTX = "Fewer than X," GTX = "Greater than X," and ETX = "Equal to X."

Is any of this useful? I'm not sure, it's certainly interesting to me, and that was enough for me to work on it. But I did think of one use case: an FTX restriction forces the spender's transaction to have more inputs than outputs, which means you can create and fund a bitcoin address that can only be spent from in a transaction that "cleans up" more utxos than it creates.

I worked on this a bit and expanded it so that you can generate an address that pays a future user only if they *create* 1 utxo but *consume* an arbitrary number, which devs can set. I call this the "cleanup restriction" and it seems like it might be useful for creating a kind of bounty hunting program that automatically pays people to clean up the utxo set. That sounds interesting to me, and maybe useful.

Anyway, that's the story behind why I made SSB playground. I hope you play around with it and have fun! Enjoy creating and testing restrictions that use the Sighash Single Bug in surprising, counterintuitive ways.
