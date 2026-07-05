<!-- *TODO*: incorporate more thinking process stuff and images. Add BDK
based data structure names (and also links) because that is what we want
to introduce the participants with. Mention the `ChangeSet`.--> Note:
The article represents a flow of thought and is therefore meant to be
read in a linear fashion. This guide is written for budding contributors
to the Bitcoin Tech ecosystem and curious individuals. If you want to
understand BDK from the user's perspective, [book-of-bdk] is  a greaaat
resource. Let’s get our expectations in order first. Take a pause and
think of what all our wallet should be able to do? 
1. Create Transactions
2. Generate Addresses
3. Sign Transactions (-- or should it?)
4. Look at the blockchain for relevant transactions.
5. Store keys ( -- wait a minute?)
6. Buy Bitcoin ( -- really?)
7. Fee estimation (--maybe not?)
8. Mine Bitcoins (-- definitely not)

Now let's go over each of these requirements and think about what all do
we need to implement. 

### Address Generation

Make it more clearer that we are just interested in public keys.

[Add a GIF for address generation here]

This is perhaps the first thing we do when we open our wallets. Assuming
we know how [addresses are
created](https://learnmeabitcoin.com/technical/keys/address/) , let's
think of how our wallet would generate addresses for us. It first needs
a public key(note: not private key!) and the address type. But we need
to be especially careful about address reuse! 

(Maybe a box on address - reuse?)
> Problems with Address Reuse Privacy Loss (due to Chain Analysts)
> Quantum Threat


But it would be so cumbersome, if for every new address we need to keep
track of a new public key! Enter [BIP32
wallets](https://learnmeabitcoin.com/technical/keys/hd-wallets/). Ok we
have A LOT (Corresponding to each path we have 2^31 and not 2^32 since
one half of the space is reserved for hardened keys ) keys here but what
about the actual script to use? How do we know that? Enter Output
Descriptors and Miniscript.

> Miniscript To refresh Miniscript is a language which compiles down to
> a subset Bitcoin Script. Miniscript helps us analyze if the script is
> valid (if it can be solved), whether its optimal etc. We use the
> policy language to specify the spending conditions. Policy is then
> converted to Miniscript language which performs optimization on the
> script for it to be solvable with the same conditions but the bytes we
> need to use to spend (satisfaction weight) becomes lesser etc. Read
> more

>Output Descriptors To refresh, descriptors consist of SCRIPT and KEY
>expressions and specify the scriptpubkey (in simple terms addresses)
>corresponding to our spending condition. In case of the newer
>descriptor types the SCRIPT can consist of policy. [Read
>more](https://bitcoin.stackexchange.com/a/99541/150114)

So with policy and Output Descriptors all our addresses can be generated
from one single string that practically doesn not exhaust!

Now  we have our addresses, except one thing! Remember address reuse? We
do need to make sure we remember which all indexes of our descriptor  we
have already looked at! So we would need an `Indexer` to remember the
last derived index of every keychain, let's call it `last_revealed`.
(Remember the pubkeys in Monitoring the blockchain section). Also since
our descriptors would generally have a wildpath (*), i.e., we would be
able to generate a lot of keys from it let's call this list of keys a
`keychain`.

Are we done? Hold on a bit! We talked a bit about entities which
infringe privacy by using our addresses to connect our transactions. A
simple way to fool them is to use two keychains instead of one! One of
the keychains will be `External` and the other would be `Internal`. And
when somebody asks you for an address you give them one generated on
your `External` keychain and you direct all your change to addresses on
the internal keychain. This way if the External (Public) keychain is
compromised (maybe because it was stored on a server), the analysts
still won't know half our addresses. Doesn't sound plausible? We'll come
to that in a minute! ( This should come when we talk about
multi-keychain: more:
[link](https://bitcoin.stackexchange.com/questions/123896/understanding-the-advantages-pitfalls-of-using-one-two-keychains-for-wallets)
(Mention the industry standard:
https://bitcoin.stackexchange.com/questions/123896/understanding-the-advantages-pitfalls-of-using-one-two-keychains-for-wallets
and only one use case of 2 desc and debunk it!) Finally done! Ready for
the next requirement?

- bdk_wallet and bdk_chain(miniscript and rust-bitcoin)
- silentpayments (talk about address reuse.)

### Monitoring the Blockchain 
So does our wallet need a node for this?
Not really (as long as we can TRUST it -- any blockchain source works,
it could be Bitcoin Core node, Electrum Server, Esplora Server and or
even a node speaking CBF(link to rob's article)). 

> Electrum and Esplora 
> Electrum Protocol is the one used to communicate
> with Electrum servers. These servers index the blockchain in order to
> achieve faster information retrieval. Not to be confused with the
> electrum wallet (the wallet speaks the electrum protocol though).
> Esplora is a modification of the Electrs ( Rust implementation of the
> Electrum Server) and to communicate with Esplora servers we use
> Esplora Protocol.


> Compact Block Filters (CBF) 
> This is the improved way of implementing a
> Simplified Verfication Client (`SPV`), basically a way for wallets to
> obtain transactions of interest without trusting the peers on the
> Bitcoin P2P network.

But in most cases our `Wallet` would only be interested in transactions
that either spend to us or from us and whether the transaction is in the
chain of most POW or not.

So we need to check if a given transaction spends an output we control
or pays to us. Remember the `Indexer`? We could have the `Indexer` also
remember the `scriptpubkey` we derived (along with which index it was
found on etc.) Now for every block or transaction the blockchain source
tells us about through say an `Update`, we can filter out the relevant
transactions.

But why do we need to maintain all this data, especially about the
chain? Can't we just query the blockchain to understand whether the
given block lies in the Most Pow chain (something we call Canonical).
The problem becomes more complicated when we consider `spk-based` chain
sources.

> spk-based Chain sources 
> These chain sources (contrary to block based
> sources) index information based on spk. 
> Hence we can query transactions related to an spk easily. 
> Electrum and Esplora are two examples.

From the performance perspective it is better if we could ask the
Blockchain source about all the blocks/txns and then do our wallet
operations offline! Note for this we would need to thus maintain what is the
current state of the blockchain (`LocalChain`). Also we would want to
store all transactions (especially those relevant to us) in order to
calculate fees etc. So (`TxGraph`). Actually it would better if we
combine the `Indexer` and `TxGraph` into the `IndexedTxGraph`, in order
to ensure these two are in sync (remember the `OutPoints` in the
`TxGraph`) (Perhaps explain more fields of the `TxGraph`.

Also we could remember the information that the source
tells us about the transaction, like when it was `first_seen`,
`last_seen` or `last_evicted` from the mempool or which block it was
found in. 
For each of the blocks we could remember its hash, height and
the block preceding it (in order to find a view of the chain considering
reorgs). 
We could also have our `Indexer` store the `Outpoints`
associated with each `scriptpubkey`.

Ok so now we know which funds we control -- time to spend (party time
graphics?) Wait a minute... We cannot just query the blockchain for all
the `spks`, there are so many of them!!! Ok so we need to decide how
many spks we would like to check. We should also persist the
`last_revealed` index of each keychain (more on that later!). Syncing vs
Full Scan (link to Thunder's article.)

- bdk_chain and chain-crates
- bdk_kyoto
- bdk_floresta

What do we need in an `Update`? 
Txs, Block Headers where txs are anchored or in mempool. Suppose rescanning so
would need revealed indices too. Block based update would have txs.

I mentioned getting data, but we need to request data in a way that BDK
understands and then process the data got, that is what the chain crates
do. Enter into the explanation of chain crates, SyncRequest , Response
etc. Construct an instance of the chain source and then create APIs to
call its APIs and process data in a way that is easy for the chain-stuff
[link to bdk_chain] to digest. What's the use of this modularization?
Suddenly the chain-stuff becomes independent of which chain_crate and
can be used by other applications too.

Comment on modularization of BDK here.

Mention that you do not need BDK wallet and can actually use the
primitives to do more interesting stuff(display the example or Evan's
PR.) BDK specific detail: how bdk_electrum_client is a wrapper on the
more general client. Do we want to explain that the structures are
monotone?

### Transaction Creation 
Aah, finally... 
Before we go into how our `Wallet` would create transactions
let's get another big idea: Partially
Signed Bitcoin Transactions (`PSBT`s)

>PSBTs
> This is an (extensible) standard to represent a transaction in order to
> increase interoperability between wallets and signers.
> Add link.
> Add Ava Chow's video link

Ok back to transaction building. Thankfully the last step told us what
all UTXOs(links) we can spend. So now we need to know what utxos the
user wants to spend for sure(`manually_selected`) and whom do we send
the bitcoins too (`recipient`), how much and what is the fee rate? Now
it is possible that the manually selected utxos are not sufficient given
the fee rate and the amount we want to send in that case (if the user
consents) we would like to use some other UTXOs too. But which ones?
Does it matter, you ask? Yeah!
Suppose the difference between the UTXOs we are spending and the ones we
are creating is very small, then we have the risk of creating dust
outputs as change!

> Dust outputs
> These are outputs which are uneconomical to spend,
> i.e., they cannot pay the fees for their relay.
> Transactions with such outputs are not relayed
> by Bitcoin Core nodes by default.

Keeping these things in mind, we need to choose coins carefully (called
`Coin Selection`).

Intro to Knapsack then Murch's work.

#### Creator and Updator
After choosing the coins, we will fill in the details of the `PSBT`
using all the information we have.

#### Signer
Next we need to `Sign` the `PSBT` . 
Mention the `plan` module here and `Miniscript`.


#### Finalizer
We finally finalize the transaction.

#### Extract
Extract the transaction and send it to the chain source!. Now our
wallet needs to know how to be able to create the correct
scriptsigs/witnesses for the inputs. How would it do that? (Talk about
policy and plan module). (What does Finalization do? Explain it here.)
Talk about signing and how it is being delegated to PSBT (maybe color
code it.)

<!-- Also need to talk about RBF. --> Ok, now we gave our transaction to
the blockchain source (to broadcast) but viola! there was a fee hike and
our tx is taking ages to confirm? one popular approach is RBF. (Talk
about RBF approach in Evan's PR.)

- bdk_tx and bdk_coinselect and bdk_wallet (policy) and plan(miniscript)
Thankfully we now know which all txns belong to our wallet. Let's go
over the process of txn creation for now.
- Users (may) select some utxos to spend.
- and send to someone for some particular amount
- with some fee. Now if the utxos are not enough we must *select* some
more UTXOs to fund the transaction. Which ones? The ones that are
unspent and abide to certain policies set by the user. 



### Persistence (Just when you thought the article is going to end :) ) 
The mainnet blockchain
is huge and scanning it everytime from the start whenever we start our
wallet program is a costly operation so we need to persist our public
descriptors, the transactions we saw, the `last_revealed`, the
`Local_Chain` etc. (Public Descriptors are the absolute essential)

So how should we persist all this data? Should we (monotoneness can come
up here, mention later on when talking about `BDK` that `LocalChain` is
not monotone and link to `BlockGraph`).
[ChangeSet] should come here ig.

- persistence (sqlite, redb, sqlx, the sqlite lib maintained by mammal
mention turso)


---

<!-- Explain why the other points need not be handled by the wallet -->

Now this is a lot of code that is common to almost all wallets (). So why not have a library (or a set of
libraries) which provides such components? That is exactly what BDK
is!!! Built on rust-bitocoin, miniscript. (Link to list of crates page
of book-of-bdk.) Touch upon modularity and independance.

Mention new things (removing signers, create_psbt, multi_keychain in
these subheadings itself.)

### FFI
Since a lot of awesome applications are written in languages
like Python, Java, React-Native (check these langs and link to ffi crate
for each) it is important to make BDK's Rust code available in these
languages (ffi therefore).

### BDK-CLI 
Playground.

--- 
Acknowledge:
inersha (learnmeabitcoin)

--- 
### Useful links 
https://github.com/bitcoindevkit/bdk/issues/1302
(bdk_chain's ChainOracle is not just for LocalChain)
