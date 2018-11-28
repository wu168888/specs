# The Filecoin Storage Market

### What is the Filecoin Storage Market

TODO: why do we even have markets? (motivate the necessity of markets vs the network allocating storage to willing miners)

The Filecoin `storage market` is the underlying system used to discover, negotiate and form `storage contracts` between clients and storage providers called `storage miners`. These contracts specify that a given `piece` will be stored for a given time duration. It is assumed that the client, or some delegate of the client, remains online to monitor the `storage miner` and `slash` it in the case that it fails to submit proofs for the data.

In the current design of the `storage market`, `storage miners` post `asks` indicating the price they are willing to accepts, and `clients` select (either manually, or via some locally run algorithm) a set of storage miners to store their data with. They then contact the `storage miners` who programmatically either accept or deny their `deal proposals`. In the future, we may allow miners to search for clients and propose deals to them, but for now, for simplicity, we stick with the model described above.

### Visualization of the Filecoin Storage Market

TODO: This is a high level overview of how the storage market interacts with components

### What the Filecoin storage market affects

- FIL Block chain via settled payments
- Exports the `power table` to the block-chain; where a miner's power can be determined in-situ (TODO: requires clarification).
- Settling up or posting the result of `payment channels` on-chain

### Dependencies

TODO: reading through this is confusing.

- `PoST` is generated fairly by `storage miners`
- Verification of `PoST` can be done by anyone in the Filecoin Storage Market
- Deals among `clients`, and `storage miners` are done through a `payment channel` but settled on the Filecoin blockchain
- Smart contracts can be evaluated deterministically

## The Market Interface

This interface, written using Go type notation, defines the set of methods that are callable on the storage market actor. The storage market actor is a built-in network actor. For more information about Actors, see TODO.

```go
type StorageMarket interface {
    // CreateStorageMiner creates a new storage miner with the given public key and a pledge
    // of the given size. The miners collateral is set by the value in the message.
    // The public key must match the private key used to sign off on blocks created
    // by this miner. This key is the 'worker' key for the miner.
    // The libp2p peer ID specified should reference the libp2p identity that the
    // miner is operating. This is the ID that clients will connect to to propose deals
    CreateStorageMiner(pubk PublicKey, pledge BytesAmount, pid libp2p.PeerID) Address

    // SlashConsensusFault is used to slash a misbehaving miner who submitted two different
    // blocks at the same block height. The signatures on each block are validated
    // and the offending miner has their entire collateral slashed, including the
    // invalidation of any any all storage they are providing. The caller is rewarded
    // a small amount to compensate for gas fees (TODO: maybe it should be more?)
    SlashConsensusFault(blk1, blk2 BlockHeader)

    // SlashStorageFaults slashes a storage miner for not submitting their PoSTs within
    // the correct time window
    SlashStorageFaults(miners []Address)
    
    // UpdateStorage is called by a miner to adjust the storage market actors
    // accounting of the total storage in the system.
    UpdateStorage(delta BytesAmount)
    
    // GetTotalStorage returns the total committed storage in the system. This number is
    // also used as the 'total power' in the system for the purposes of the power table
    GetTotalStorage() BytesAmount
}
```

## The Filecoin Storage Market Operation

The Filecoin storage market operates as follows. Miners providing storage submit ask orders, asking for a certain price for their available storage space, and clients with files to store select an ask they wish to use. Clients negotiate directly with the storage miner that owns that ask, off-chain. Storage is priced in terms of Filecoin per byte per block (note: we may change the units here).

### Market Datastructures

TODO: This storage-market.md doc should try to describe the high level interfaces of the storage market, details about storage and specific method behavior should go in a separate place where we talk about individual system actors.

The storage market contains the following data:

- StorageMiners - The storage market keeps track of the set of the addresses of all storage miners in the storage market. All miners referenced here were created by the storage market via the `CreateMiner` method.
- TotalComittedStorage - This is a tally of all the committed storage in the network. This is both a nice metric to see how much data is being stored by the filecoin network, and a critical piece of information used by mining routine to compute each miners storage ratio.

## Market Flow

This section describes the flow required to store a single piece with a single storage miner. Most use-cases will involve performing this process for multiple pieces, with different miners.

#### Before Deal

1. **Merkle Translation:** The client runs a 'Local Merkle Translation' to generate the pedersen commitment for their piece of data.
  - Storage miners reference data by its pedersen commitment, and not by its standard hash. This step is needed for the client to be able to trust their data was correctly included in the miners sector. See [Piece Confirmation](definitions.md#piece-confirmation)
2. **Miner Selection:** The client looks at asks on the network, and then selects a storage miner to store their data with.
   - Note: this is currently a manual process.
3. **Payment Channel Setup:** The client calls [`Payment.Setup`](#payments) with the piece and the funds they are going to pay the miner with.

#### Deal

1. **Storage Deal Staging:** The client now runs the ['make storage deal'](network-protocols.md#storage-deal) protocol, as follows:
  - The client sends a `StorageDealProposal` for the piece in question
    - This contains updates for the payment channel that the client may close at any time, unless the piece gets confirmed (see next section), in which case the miner is able to extend the channel.
  - The miner decides whether or not to accept the deal and sends back a `StorageDealResponse`
  - If the miner accepts, the client now sends the data to the miner
  - Once the miner receives the data:
    - They validate that the data matches the hashes claimed by the client
    - They stage it into a sector and set the deal state to `Staged`

2. **Storage Deal Start**: Clients makes sure data is in a sector
   - **PieceConfirmation:** Once the miner seals the sector, they update the PieceConfirmation in the deal state, which the client then gets the next time they query that state.
     - The PieceConfirmation proves that the piece in the deal is contained in a sector whose commitment is on chain. The 'Merkle Translation' hash from earlier is used here.
     - Note: a client that is not interested in staying online to wait for PieceConfirmation can leave immediately, however, they run the risk that their files don't actually get stored (but if their data is not stored, the miner will not be able to claim payment for it).
     - Note: In order to provide the piece confirmation, the miner needs to fill the sector. This may take some time. So there is a wait between the time the data is transferred to the miner, and when the piece confirmation becomes available.
   - **Mining**: Miner posts `seal commitment` and proof on chain and starts running `proofs of spacetime`

3. **Storage Deal Abort:** If the miner doesn't provide the PieceConfirmation, the client can invalidate the payment channel.
   - This is done by invoking the 'close' method on the channel on-chain. This process starts a timer that, on finishing, will release the funds to the client. 
   - If a client attempts to abort a deal that they have actually made with a miner, the miner can submit a payment channel update to force the channel to stay open for the length of the agreement.

4. **Storage Deal Complete:** The client periodically queries the miner for the deals status until the deal is 'complete', at which point the client knows that the data is properly replicated.
   - The client should store the returned 'PieceConfirmation' for later validation.

5. **Income Withdrawal**: When the miner wishes to withdraw funds, they call `Payment.RedeemVoucher`.

## The Power Table

The `power table` is exported by the storage market for use by consensus. There isn't actually a concrete object that is the power table, instead, there is a 'total power' exported by the storage market actor, and each individual miner reports its power through their actor.

### Power Updates

A miners power is incremented every time they commit a new seal to the chain. (TODO: this should actually change to being updated once the first PoSt for that data is completed).

Power is deducted when miners remove sectors, either through direct action (calling `RemoveSector`) or by reporting the sector missing in a PoSt.

## Payments

The storage market expects a payments system to allow clients to pay miners for storage. Any payments system that has the following capabilities may be used:

```go
type Payments interface {
    // Setup sets up a payment from the caller to the target address. The payment
    // MUST be contingent on the miner being able to prove that they have the data
    // referenced by 'piece'. The total amount of Filecoin that may be transfered by
    // this payment is specified by 'value'
    Setup(target Address, piece Cid, value TokenAmount) ID

    // MakeVouchers creates a set of vouchers redeemable by the target of the
    // previously created payment. It creates 'count' vouchers, each of which is
    // redeemable only after an certain block height, evenly spaced out between
    // start and end. Each voucher should be redeemable for proportionally more
    // Filecoin, up to the total amount specified during the payment setup.
    MakeVouchers(id ID, start, end BlockHeight, count int) []Voucher

    // Redeem voucher is called by the target of a given payment to claim the
    // funds represented by it. The voucher can only be redeemed after the block
    // height that is attributed to the voucher, and also only if the proof given
    // proves that the target is correctly storing the piece referenced in the
    // payment setup.
    RedeemVoucher(v Voucher, proof Proof)
}
```



## Future Protocol Improvements

- Slashable Commitments
  - When miners initially receive the data for a deal with a client, that signed response statement can be used to slash the miner in the event that they never include that data in a sector.

# Open questions

- Storage time should likely be designated in terms of proving period. Where a proving period is the number of blocks in which every miner must submit a proof for their sectors. Not doing this makes accounting hard: "when exactly did this sector fail?"