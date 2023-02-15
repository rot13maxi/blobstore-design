# blobstore-design

This is a design sketch for a non-consensus, optional (and opt-in) blobstore that can run next to bitcoin nodes. It is incentive compatible for miners, requires no consensus changes, has no effect on transaction or block validity, and has enough of the useful properties of bitcoin witness data storage (inscriptions, etc) to make it an attractive alternative for people who want to store data in bitcoin transaction witnesses.

Note to the reader: As of 2023-02-15, none of this software exists, so when you read things that sound like they assume software is written, imagine those statements taking place in the future, after the software has been written.

Terminology note: `blob` is used liberally below to mean "a string of arbitrary bytes that has some application-specific semantic or encoding, but is opaque to the system as a whole".

## Design Constraints
- Don't add new consensus rules to Bitcoin
- Don't disadvantage miners who can't/won't run a parallel blobstore
- Don't affect transaction or block validity
- Don't create an unbounded blobspace, this will make it so that very few people will run it, which makes it less useful and makes storing data in transaction witnesses more valuable. 
- We want something that has data availability that is close enough to witness data that it serves as a viable alternative for usecases like storing encrypted miniscript policies, NFTs, etc. 

## Design Overview
There is a new daemon called `blobspaced` that users can optionally run next to their `bitcoind`. `blobspaced` stores arbitrary bytestrings, indexed by bitcoin transaction IDs. In other words, you can imagine the core functionality of `blobspaced` as being a HashMap that maps TransactionID -> Vector<u8>. `blobspaced` instances connect to peers that also run `blobspaced` in the same was as Bitcoin nodes, and relay blobs to each other over a BIP324 encrypted transport. `blobspaced` nodes have their own mempool where they hold blobs that have been sent to them by peers, but are not yet committed to the blobspace. 

When a user wants to add a blob to `blobspace`, they construct a Bitcoin transaction that when spent puts a `blob-reference` into the witness of an input being spent. The `blob-reference` is encoded as follows:

```
OP_FALSE
OP_IF
  OP_PUSH "blob"
  OP_1
  32 [sha256 hash of the blob]
  OP_0
  4 [size of the blob in bytes]
OP_ENDIF
```

They broadcast a transaction with that encoding in transaction witness data to the bitcoin network, and they publsh the blob to their `blobspaced` peers. `blobspaced` is notified by `bitcoind` when a block is added to the chain (either via zmq, rpc polling, or some other mechanism). When a bitcoin block is added to the chain, `blobspaced` reads the `blob-reference` transactions from the block, sorts them by fee-rate, and then adds up to 4MB of blobs referenced in the `blob-reference` transaction set from its blobspace mempool to the committed blobspace. In other words, those blobs have been "confirmed". Once the 4MB limit of blobs per block has been reached, any additional blobs referenced by transactions in the blobspace mempool can be pruned, or may be cached for another few blocks incase users try to confirm them in a subsequent block.

## Nice things about this design
Blobspace is constrained. There is a maximum of 4MB that can be added to blobspace per Bitcoin Node. That means three things:
1. Blobspace is still scarce and valuable, which makes it an attractive place to put scarce collectibles
2. Blobspace does not grow unbounded. If there was no cap on blobspace, then very few people would run `blobspaced`, which means that people won't pick it over IPFS, Bittorrent, centralized file hosting, etc. People who wish to run unpruned `bitcoind` AND unpruned `blobspaced` are signing up for at max, ~420GB of incremental space per year. That is within the realm of plausible for a broadly distributed system.
3. If blobspace is in demand, users have to bid up their blob-reference transactions to get their blobs confirmed. This means that moving from data-in-witnesses to data-in-blobspace doesn't hurt miner revenue.

The blob reference in the transaction witness can be validated by any node that isn't running `blobspaced`. Furthermore, because the size of the blob is included in that reference, a small change to `getblocktemplate` can construct bitcoin blocks that include only the highest-paying 4MB of blobs into blobspace, without needing to run `blobspaced`. That means that miners can include the most profitable blobs, without including blob-references that get excluded from blobspace (in other words, without breaking the consensus rules of the blobspace), and they can do it without they themselves needing to run `blobspaced`. This is incredibly important because it means that this design doens't add new operational costs to miners, and also doesn't disadvantage miners who have better or worse exposure to blob relay. Because blobspace fees are being paid on the bitcoin-side, miners will have an incentive to not break blobspace. Being able to mine blobspace-optimal transactions without actually needing to run `blobspaced` is an important feature of this design.

This design does not rely on any aggregators or centralized relay points for blob storage or merklization. Anyone should be able to run `blobspaced` next to `bitcoind` and have the same access to blobspace that they have to bitcoin, provided enough nodes on the network are relaying blobs. 

