<!-- *TODO*: incorporate more thinking process stuff and images. Add BDK
based data structure names (and also links) because that is what we want
to introduce the participants with. Mention the `ChangeSet`.-->

<!-- Rather than BIP links use better articles on the topic -->
filled with links.
Here is a guide filled with links for creating a `Wallet`. If that interests
you, take a deep breath and dive in!
Note:
The article represents a flow of thought and is therefore meant to be
read in a linear fashion. This guide is written for budding contributors
to the Bitcoin Tech ecosystem and curious individuals. If you want to
understand BDK from the user's perspective, [book-of-bdk] is  a greaaat
resource.
Let’s get our expectations in order first. Take a pause and
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

> Address Reuse
> There are various entities (like [ChainAnalysis]) which actively
> try to associate Bitcoin addresses to individuals using hereustics.
> Address Reuse makes it easier for these entities to identify these
> associations and hence infringe privacy which in turn leads to 
> censorship and targeted advertising.
> Also the Address Reuse becomes critical blow to security if ECDSA
> gets broken by advancements in Quantum Computing.
> [Read More here](https://en.bitcoin.it/wiki/Address_reuse)


But it would be so cumbersome, if for every new address we need to keep
track of a new public key! Enter BIP32
wallets.

> BIP32 (Heirarchical Deterministic Wallets)
> These wallets use a master key and a derivation path to deterministically 
> generate keys.
> [Read More](https://learnmeabitcoin.com/technical/keys/hd-wallets/)

Ok we have A LOT (corresponding to each path we have 2^31 and not 2^32 since
one half of the space is reserved for hardened keys ) keys here but what
about the actual script to use? How do we know that? Enter Output
Descriptors and Miniscript.

> Miniscript 
> To refresh Miniscript is a language which compiles down to
> a subset Bitcoin Script. Miniscript helps us analyze if the script is
> valid (if it can be solved), whether its optimal etc. We use the
> policy language to specify the spending conditions. Policy is then
> converted to Miniscript language which performs optimization on the
> script for it to be solvable with the same conditions but the bytes we
> need to use to spend (satisfaction weight) becomes lesser etc. 
> Read more

>Output Descriptors 
> To refresh, descriptors consist of SCRIPT and KEY
>expressions and specify the scriptpubkey (in simple terms addresses)
>corresponding to our spending condition. In case of the newer
>descriptor types the SCRIPT can consist of policy. [Read
>more](https://bitcoin.stackexchange.com/a/99541/150114)

So with policy and Output Descriptors all our addresses can be generated
from one single string that practically does not exhaust!

Now  we have our addresses, except one thing! Remember address reuse? We
do need to make sure we remember which all indexes of our descriptor  we
have already looked at! So we would need an `Indexer` to remember the
last derived index of every keychain, let's call it `last_revealed`. Also since
our descriptors would generally have a wildpath (*), i.e., we would be
able to generate a lot of keys from it, let's call this list of keys a
`keychain`.

Are we done? Hold on a bit! We talked a bit about entities which
infringe privacy by using our addresses. 
A simple way to trick them is to use two keychains instead of one! One of
the keychains will be `External` and the other would be `Internal`. And
when somebody asks you for an address you give them one generated on
your `External` keychain and you direct all your change to addresses on
the internal keychain. This way if the External (Public) keychain is
compromised (maybe because it was stored on a server), the analysts
still won't know half our addresses. Doesn't sound plausible? We'll come
to that in a minute! 

Finally done! Ready for the next requirement?

- bdk_wallet and bdk_chain(miniscript and rust-bitcoin)
- silentpayments (talk about address reuse.)

### Monitoring the Blockchain 
So does our wallet need a node for this?
Not really (as long as we can TRUST it -- any blockchain source works,
it could be Bitcoin Core node, Electrum Server, Esplora Server and or
even a node speaking CBF).

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
> Read More

In most cases our `Wallet` would only be interested in transactions
that either spend to us or from us and whether the transaction is in the
chain of most POW or not. 
We would want to remember the `UTXO`s that we
have to be able to spend further.

So we need to check if a given transaction spends an output we control
or pays to us. Remember the `Indexer`? We could have the `Indexer` also
remember the `scriptpubkey` we derived (along with which index it was
found on etc.) The `Indexer` can take the responsibility of remembering
the transactions spending to us.

Also a lot of times, we would want to remember the transactions that spend to us
or those we created. Perhaps to understand the conditions under which we can spend
our outputs or to do a fee-bump later on. Let us store the transactions in a
`TxGraph`, which maintains ancestry relationships between the transactions as
well. Since we are storing the whole transactions anyways, our `Indexer` could
be made lighter by just holding on to the `Outpoint`s (`txid`, `vout` pairs for
each output.) 
Actually it would better if we combine the `Indexer` and `TxGraph` into the `IndexedTxGraph`, in order to ensure these two are in sync.

But the way we query a chain source for information depends on its type.

> spk-based chain sources 
> These chain sources (contrary to block based
> sources) index information based on spk. 
> Hence we can query transactions related to an spk easily. 
> Electrum and Esplora are two examples.

> Block based Chain sources
> These chain sources provide whole blocks and mempool transactions
> when queried.

For an spk-based chain source we could provide the chain source the
scriptpubkeys we are interested in.

For a block based source we could just query for new blocks or new transactions
in mempool.

Now for every block or transaction the blockchain source
tells us about through (say an `Update`), we can update our `IndexedTxGraph`.

Just when we thought that life is easy, Bitcoin's distributed consensus reminds
us of its presence. I mean that all this is good if there were no reorgs! In
case of a reorg, the transactions we once thought were confirmed could get
unconfirmed!

Ok so every time we query our balance we need to ask the chain source whether each of the `UTXO`s actually exist in the current chain. And note we would need to do this for every `UTXO`, no matter what the reorg depth was. Crazy! Since in many applications, network speed is a bottleneck this design might cost us on performance. Also we might want our wallet to connect to the network as less as possible maybe because of low availability of the network. 

What if instead we could query the chain source for new blockheaders and try to
build up a view of the blockchain, say `LocalChain` ourselves offline? 

For this every time we query the chain source, we can also ask it for the
required block-headers. Also since not every block is of interest to us the
`LocalChain` could be sparse, i.e., we can maintain `CheckPoint`s (which contains
the height, the hash of the block at this height and the previous `CheckPoint`).

But we started with the goal of finding if the transaction exists in the chain
of most POW. For this we associate to each transaction an `Anchor` (pair of
blockhash and block height) which *anchors* it in some chain. Now we just need
to check if the `Anchor` corresponds to a block in the most POW chain 
(let's call it the `Canonical` chain).

Also we could remember the information that the source
tells us about the transaction, like when it was `first_seen`,
`last_seen` from the mempool if its unconfirmed.
For each of the blocks we could remember its hash, height and
the block preceding it (in order to find a view of the chain considering
reorgs).

Ok so now we know which funds we control -- time to spend (party time
graphics?)
Wait a minute... For spk-based chain sources, we cannot just query 
the blockchain for all the `spks`, 
there are so many of them!!! Ok so we need to decide how
many spks we would like to check.
This brings us to two forms of querying:- Sync and Full Scan.

>> Full Scan and Sync
>> In case we have been querying the chain source frequently we can
>> just query for `scriptpubkey`s that have been revealed, this is called sync.
>> In case we have just created our wallet or are starting up after a
>> long time, the wallet could check some extra (`Lookahead` many)
>> `scriptpubkeys` along with those which have been revealed this is called
>> a full scan.
>> Read more.

In case of a block based source we could just store some extra (`Lookahead`
many) `scriptpubkey`s apart from those revealed, when we make the `Indexer` keep
info about a descriptor or whenever we reveal a new address.

- bdk_chain and chain-crates
- bdk_kyoto
- bdk_floresta

What do we need in an `Update`?
Txs, Block Headers where txs are anchored or in mempool. Suppose rescanning so
would need revealed indices too. Block based update would have txs.

I mentioned getting data, but we need to request data in a way that BDK
understands and then process the data got, that is what the chain crates
do.

The chain crates construct an instance of the chain source and 
then call APIs to call the chain source's APIs and process data 
in a way that is easy for the chain-stuff [link to bdk_chain] to digest. 

Depending on the type of chain crate, the chain crates take in a
`SyncRequest`, `FullScanRequest` and return a `SyncResponse`, `FullScanResponse` or `BlockEvent`, `MempoolEvent`. What should each of the types be?

What's the use of this modularization?
Suddenly the chain-stuff becomes independent of which chain_crate and
can be used by other applications too.

Comment on modularization of BDK here.

Mention that you do not need BDK wallet and can actually use the
primitives to do more interesting stuff(display the example or Evan's
PR.) BDK specific detail: how `bdk_electrum_client` is a wrapper on the
more general client.

### Transaction Creation 
`TxGraph` maintaining a map of `Outpoint`s to `txid` of
the transaction that spend the `UTXO`.
Aah, finally...
Before we go into how our `Wallet` would create transactions
there is another big idea we need to discuss: Partially
Signed Bitcoin Transactions (`PSBT`s)

>PSBTs
> This is an (extensible) standard to represent a transaction in order to
> increase interoperability between wallets and signers.
> Every entity that interacts with `PSBT` has a role: `Creator`, `Updator`,
> `Signer`, `Finalizer` and `Extractor`.
> Add link.
> Add Ava Chow's video link

#### Creator and Updator
Ok back to transaction building. Thankfully the last step told us what
all UTXOs we can spend. So now we need to know what utxos the
user wants to spend for sure(`manually_selected`), whom do we send
the bitcoins too (`recipient`), the `amount` we want to spend
and what is the fee rate? 
Now it is possible that the manually selected utxos are not sufficient given
the fee rate and the amount we want to send in that case
we would like to use some other UTXOs too, if the user consents. But which ones?
Does it matter, you ask? Yeah!
Suppose the difference between the UTXOs we are spending and the ones we
are creating after adjusting for fees is so small that the change output becomes
a dust UTXO?

> Dust outputs
> These are outputs which are uneconomical to spend,
> i.e., they cannot pay the fees for their relay.
> Transactions with such outputs are not relayed
> by Bitcoin Core nodes by default.

Also the user needs to pay lesser fees if the transaction size is less so it is
necessary to optimize for the size.

Keeping these things in mind, we need to choose our coins carefully (called
`Coin Selection`).

Ok so we need a `CoinSelector` which given few `Candidate`s (groups of inputs
that need to be spent together) and `target` can decide (according to an algorithm) which `Candidate`s to select.

What should the alogrithm be?
It could be as simple as selecting randomly from the available `UTXO`s till we
reach the desired target (called `Single Random Draw`). Or it could as
complicated as `Branch and Bound` algorithm as described in Murch's thesis(add
link). Or the `LowestFee` algorithm which optimizes for fees.

But in order to calculate the weight, we need to know the number of vbytes
required to spend each output (`satisfaction_weight`) we are trying to spend.
How do we do that?
Remember this was exactly one of the goals of `Miniscript` !!! So we could
simply ask our `Miniscript` library to generate a spending `Plan` for us when
told about the signatures, hash preimages (`Assets`) we have! And we could query
the `Plan` for the `satisfaction_weight`.
Mention later on that BDK uses its own policy module but about to be 
phased out.

After choosing the coins, using the `Plan` we will fill in the details of the
`PSBT` that are needed for optimal spending.

#### Signer
Now the `PSBT` could be sent to signers to sign. Why doesn't our wallet do the
signing? Because wouldn't it be better if our precious keys stay away from our
applications as much as possible? They can instead stay safe inside H/W wallets
or Cryobrick link and we could get our created `PSBT`s signed by these external
signers!

#### Finalizer
We finally finalize the transaction, which means that we use the `Wallet`s
descriptors to populate the `final_scriptsig` and the `final_witness_script`
fields of the psbt.

#### Extract
We can now extract the transaction and send it to the chain source!. 

Ok, now we gave our transaction to the blockchain source (to broadcast) 
but viola! there was a fee hike and our tx is taking ages to confirm? 
one popular approach is RBF.

- bdk_tx and bdk_coinselect and bdk_wallet (policy) and plan(miniscript)

### Persistence (Just when you thought the article is going to end :) ) 
The mainnet blockchain
is huge and scanning it everytime from the start whenever we start our
wallet program is a costly operation so we need to persist our public
descriptors, the transactions we saw, the `last_revealed`, the
`Local_Chain` etc. (Public Descriptors are the absolute essential)
`last_evicted` should come when talking about monotone.

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

### MultiKeychain
Remember we talked about using 2 keychains? 
( This should come when we talk about
multi-keychain: more:
[link](https://bitcoin.stackexchange.com/questions/123896/understanding-the-advantages-pitfalls-of-using-one-two-keychains-for-wallets)
(Mention the industry standard:
https://bitcoin.stackexchange.com/questions/123896/understanding-the-advantages-pitfalls-of-using-one-two-keychains-for-wallets
and only one use case of 2 desc and debunk it!) 

This also something I contribute to in BDK.

--- 

Acknowledge: inersha (learnmeabitcoin)

--- 
### Useful links 
https://github.com/bitcoindevkit/bdk/issues/1302(bdk_chain's ChainOracle is not just for LocalChain)
