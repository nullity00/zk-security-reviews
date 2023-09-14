[Original Article](https://www.zksecurity.xyz/blog/posts/penumbra/)

Public report of auditing Penumbra's circuits on Aug 24, 2023

![penumbra audit](https://i.imgur.com/gUirXcc.png)

We have audited [Penumbra](https://penumbra.zone/)'s main circuits, and published a report of the findings. You can read about it on [Penumbra's blog post here](https://penumbra.zone/blog/2023-audits).

> Of those 8 issues found, the Penumbra Labs team considered the two highest impact bugs to be the "double spend" and "double vote" bugs (rated high by zkSecurity, see report items #0 and #1), each with clear paths to exploitation. As of time of publication, the Penumbra Labs team has resolved all issues higher than "informational" identified by either of the audit teams, and confirmed these fixes by subsequent follow-up review with both original audit performers. Details can be found in the audit reports, but we'd like to summarize the two critical findings and their resolution below.

As we noted in our report, the code was found to be thoroughly documented, rigorously tested, and well specified.

We are happy to see that the Penumbra team has been very responsive to our findings, and we are looking forward to seeing the project launch soon.

What follows is a copy/paste of the introduction of our report.

* * * * *

Penumbra is a [Cosmos zone](https://v1.cosmos.network/resources/faq) (an application-specific blockchain in the Cosmos ecosystem) where the main token is used to delegate power to validators and vote on proposals. The protocol allows the Penumbra token itself, as well as external tokens (connected to Penumbra via the [IBC protocol](https://ibcprotocol.org/)), to live in a shielded pool a la Zcash where transactions within the same token provide full privacy.

The protocol's main feature is a decentralized exchange (also called DEX) in which users can trade tokens in the clear (by moving their shielded tokens to open positions), or in a private way if they choose to trade at market price.

To enable the privacy features of Penumbra, [zero-knowledge proofs](https://www.zksecurity.xyz/blog/posts/helloworld/) are used. Specifically, the [Groth16](https://eprint.iacr.org/2016/260) proof system is used on top of the [BLS12-377](https://eips.ethereum.org/EIPS/eip-2539) elliptic curve.

In order to perform group operations within the circuit, the [Edwards curve introduced in the ZEXE paper](https://protocol.penumbra.zone/main/crypto/decaf377.html#curve-parameters) was chosen as the "inner curve". In addition, the curve is never used directly, but rather through the [decaf377](https://protocol.penumbra.zone/main/crypto/decaf377.html) abstraction/encoding.

In the next sections we introduce different aspects of the protocol, when relevant to the zero-knowledge proofs feature of the protocol.

### Statements in code

The zero-knowledge proofs in Penumbra can be observed in users' transactions, proving that specific statements are correct to the validators, and allowing the validators to safely perform state transitions in the consensus protocol.

Each transaction can contain a list of actions, defined in the [specification](https://protocol.penumbra.zone/main/concepts/transactions.html). Not all actions contain zero-knowledge proofs, as some of them will be performed in the clear. Interaction between actions that modify balances, specifically between private actions themselves, or between private and public actions, happen hidden within commitments.

For example, as part of a single transaction some actions might create positive balances for some tokens hidden in commitments, and some other actions will create negative balances for some tokens hidden in commitments. The result will be verified to be a commitment to a 0 balance, either in the clear or via a proof from the user (as commitments can be hiding). (The proof is also embedded in a user's signature via the use of [binding signatures](https://protocol.penumbra.zone/main/crypto/decaf377-rdsa.html?highlight=binding#binding-signatures).)

In the codebase, each component is cleanly separated from the rest and self-contained within a crate (Rust library). Actions are encoded in `action.rs` files under the different crates, and circuits are encoded in `proof.rs` files under the same crate. Some gadgets are implemented under `r1cs.rs` files.

Circuits are implemented using of the [arkworks r1cs-std](https://github.com/arkworks-rs/r1cs-std) library, and as such can easily be found as an implementation of a `ConstraintSynthesizer` trait on circuit structures:

```
pub trait ConstraintSynthesizer<F: Field> {
    fn generate_constraints(self, cs: ConstraintSystemRef<F>) -> crate::r1cs::Result<()>;
}

```

To form a transaction, users follow a "transaction plan". The implementation of a transaction always leads the logic to `plan.rs` files which contain the creation of the action, including the creation of a proof for private actions.

On the other hand, an `action_holder/` folder can always be found for each component, which will determine how validators need to handle actions. For private actions, this will lead to the validators verifying the proofs contained in a transaction's actions. Verifications are split into two parts, a stateful one that performs checks which need to read the state, and a stateless one that performs any other checks:

```
pub trait ActionHandler {
    type CheckStatelessContext: Clone + Send + Sync + 'static;
    async fn check_stateless(&self, context: Self::CheckStatelessContext) -> Result<()>;
    async fn check_stateful<S: StateRead + 'static>(&self, state: Arc<S>) -> Result<()>;
    async fn execute<S: StateWrite>(&self, state: S) -> Result<()>;
}

```

An `ActionHandler` is also implemented for a transaction, which will call the relevant action handlers for each of the actions contained in the transaction. Among others, it also sums up each action's *balance commitment* in order to ensure (using the signature binding scheme previously mentioned) that they all sum up to 0.

In the rest of the sections we review the different actions that are related to zero-knowledge proofs, while at the same time giving high-level pseudocode description of the circuits (not-including gadgets and building blocks).

### Multi-asset shielded pool

Similar to Zcash, values in penumbra are stored as commitments in a Merkle tree. A particularity of Penumbra's Merkle tree is that each node links to four children. It is also technically a hyper tree (a tree of tree) where a leaf tree contains all of the commitments that were created in a block, the middle tree contains all of the blocks created in an epoch, and the top tree routes to the different epochs.

In order to spend a commitment, a spend action (and its spend proof) must be present in a transaction. In effect, the spend proof verifies that the note exists, and that it has never been spent. Revealing the commitment itself could allow validators to check if a commitment has been spent before, it leads to poor privacy. For this reason, another value strongly tied to the commitment is derived (provably) called a *nullifier*. In a sense, if Bitcoin tracks all of the unspent outputs, Penumbra tracks all of the spent ones instead.

In pseudocode, the logic of the spend circuit is as follows:

```
def spend(private, public):
    # private inputs
    note = NoteVar(private.note)
    claimed_note_commitment = StateCommitmentVar(private.state_commitment_proof.commitment)

    position = PositionVar(private.state_commitment_proof.position)
    merkle_path = MerkleAuthPathVar(private.state_commitment_proof)

    v_blinding = uint8vec(private.v_blinding)

    spend_auth_randomizer = SpendAuthRandomizer(private.spend_auth_randomizer)
    ak_element = AuthorizationKeyVar(private.ak)
    nk = NullifierKeyVar(private.nk)

    # public inputs
    anchor = public.anchor
    claimed_balance_commitment = BalanceCommitmentVar(public.balance_commitment)
    claimed_nullifier = NullifierVar(public.nullifier)
    rk = RandomizedVerificationKey(public.rk)

    # dummy spends have amounts set to 0
    is_dummy = (note.amount == 0)
    is_not_dummy = not is_dummy

    # note commitment integrity
    note_commitment = note.commit()
    if is_not_dummy:
        assert(note_commitment == claimed_note_commitment)

    # nullifier integrity
    nullifier = NullifierVar.derive(nk, position, claimed_note_commitment)
    if is_not_dummy:
        assert(nullifier == claimed_nullifier)

    # merkle auth path verification against the provided anchor
    if is_not_dummy:
        merkle_path.verify(position, anchor, claimed_note_commitment)

    # check integrity of randomized verification key
    computed_rk = ak_element.randomize(spend_auth_randomizer)
    if is_not_dummy:
        assert(computed_rk == rk)

    # check integrity of diversified address
    ivk = IncomingViewingKey.derive(nk, ak_element)
    computed_transmission_key = ivk.diversified_public(note.diversified_generator)
    if is_not_dummy:
        assert(computed_transmission_key == note.transmission_key)

    # check integrity of balance commitment
    balance_commitment = note.value.commit(v_blinding)
    if is_not_dummy:
        assert(balance_commitment == claimed_balance_commitment)

    # check the diversified base is not identity
    if is_not_dummy:
        assert(decaf377.identity != note.diversified_generator)
        assert(decaf377.identity != ak_element)

```

A spend proof includes a committed balance in its public input, exposing a hidden version of the balance that was contained in the note. As notes can hold different types of assets, the commitments to notes are computed as Pedersen commitments that use different base points for different assets.

As said previously, validators will eventually check that all actions from a transaction have balance commitments that cancel out. One of the possible ways to decrease the now positive balance is to create a *spendable output* for someone else (or oneself). The action to do that is called an *output action*, which simply exposes a different kind of (committed) balance: a negative one, potentially negating the balance of a spend action, but of other actions as well. An output proof also exposes a commitment to that new note it just created, allowing validators to add it to the hyper tree.

The pseudocode for that circuit is as follows:

```
def output(private, public):
    # private inputs
    note = NoteVar(private.note)
    v_blinding = uint8vec(private.v_blinding)

    # public inputs
    claimed_note_commitment = StateCommitmentVar(public.note_commitment)
    claimed_balance_commitment = BalanceCommitmentVar(public.balance_commitment)

    # check the diversified base is not identity
    assert(decaf377.identity != note.diversified_generator)

    # check integrity of balance commitment
    balance_commitment = BalanceVar(note.value, negative).commit(v_blinding)
    assert(balance_commitment == claimed_balance_commitment)

    # note commitment integrity
    note_commitment = note.commit()
    assert(note_commitment == claimed_note_commitment)

```

### Governance

A delegator vote is a vote on a proposal from someone who has delegated some of their tokens to a validator. Their voting power is related to their share of the total tokens delegated to that validator. As such, a delegator vote proof simply shows that they have (or had, at the time the proposal got created) *X* amount of delegated tokens in the pool of validator *Y*, revealing both of these values. In addition, it also reveals the nullifier correlated with the then-delegated note, in order to prevent double voting.

In exchange for voting, an NFT is created and must be routed towards a private output in the shielded pool (using an *output action*).

The pseudocode for the delegator vote statement is as follows:

```
def delegator_vote(private, public):
    # private inputs
    note = NoteVar(private.note)
    claimed_note_commitment = StateCommitmentVar(private.state_commitment_proof.commitment)

    delegator_position = private.state_commitment_proof.delegator_position
    merkle_path = merkleAuthPathVar(private.state_commitment_proof)

    v_blinding = u8vec(private.v_blinding)

    spend_auth_randomizer = SpendAuthRandomizer(private.spend_auth_randomizer)
    ak_element = AuthorizationKeyVar(private.ak)
    nk = NullifierKeyVar(private.nk)

    # public inputs
    anchor = public.anchor
    claimed_balance_commitment = BalanceCommitmentVar(
        public.balance_commitment)
    claimed_nullifier = NullifierVar(public.nullifier)
    rk = RandomizedVerificationKey(public.rk)
    start_position = PositionVar(public.start_position)

    # note commitment integrity
    note_commitment = note.commit()
    assert(note_commitment == claimed_note_commitment)

    # nullifier integrity
    nullifier = NullifierVar.derive(
        nk, delegator_position, claimed_note_commitment)
    assert(nullifier == claimed_nullifier)

    # merkle auth path verification against the provided anchor
    merkle_path.verify(delegator_position.bits(),
                       anchor, claimed_note_commitment)

    # check integrity of randomized verification key
    # ak_element + [spend_auth_randomizer] * SPEND_AUTH_BASEPOINT
    computed_rk = ak_element.randomize(spend_auth_randomizer)
    assert(computed_rk == rk)

    # check integrity of diversified address
    ivk = IncomingViewingKey: : derive(nk, ak_element)
    computed_transmission_key = ivk.diversified_public(
        note.address.diversified_generator)
    assert(computed_transmission_key == note.transmission_key)

    # check integrity of balance commitment
    balance_commitment = note.value.commit(v_blinding)
    assert(balance_commitment == claimed_balance_commitment)

    # check elements were not identity
    assert(identity != note.address.diversified_generator)
    assert(identity != ak_element)

    # check that the merkle path to the proposal starts at the first commit of a block
    assert(start_position.commitment == 0)

    # ensure that the note appeared before the proposal was created
    assert(delegator_position.position < start_position.position)

```

### Decentralized exchange

The main feature of Penumbra (besides its multi-asset shielded pool) is a decentralized exchange (also called DEX). In the DEX, peers can swap different pairs of tokens in the open by creating positions.

Positions, created by *position actions*, create negative balances as non-hiding commitments, aiming at canceling out spendings created by spend actions. As positions are a *public* market-making feature of the DEX, allowing users to use different trading strategies (or provide liquidity to pairs of tokens) in the clear, we will not talk more about that in this document.

On the other side of the DEX are *Zswaps*, which allow users to trade tokens privately at "market price". A Zswap has two steps to it: a *swap*, and a *swap claim* (or *sweep*).

The swap action and proof takes a private *swap plaintext*, which describes the pair of tokens being exchanged, and what amount of tokens are being exchanged for each asset. The intention with having two amounts is to hide which is being traded. So in practice, only one amount will be set to non-zero.

The proof verifiably releases a commitment of the swap plaintext, which will allow the user to claim their tokens once the trade has been executed. The commitment to the swap plaintext is derived as a unique asset id, so as to be stored in the same multi-asset shielded pool as a commitment note of value 1 (i.e. as a non-fungible token).

In addition, the proof also releases a hidden commitment of a negative balance, subtracting both amounts from the user's balance. Validators will eventually verify that positive balances matching these negative balances are created in other actions of the same transaction.

(Note that a prepaid fee is also subtracted from the balance, which will allow the transaction to claim the result of that trade later on without having to spend notes to cover the transaction fee.)

The pseudocode of the swap statement is as follows:

```
def swap(private, public):
    # private inputs
    swap_plaintext = SwapPlaintextVar(private.swap_plaintext)
    fee_blinding = uint8vec(private.fee_blinding_bytes)

    # public inputs
    claimed_balance_commitment = BalanceCommitmentVar(
        public.balance_commitment)
    claimed_swap_commitment = StateCommitmentVar(public.swap_commitment)
    claimed_fee_commitment = BalanceCommitmentVar(public.fee_commitment)

    # swap commitment integrity check
    swap_commitment = swap_plaintext.commit()
    assert(swap_commitment == claimed_swap_commitment)

    # fee commitment integrity check
    fee_balance = BalanceVar.from_negative_value_var(swap_plaintext.claim_fee)
    fee_commitment = fee_balance.commit(fee_blinding)
    assert(fee_commitment == claimed_fee_commitment)

    # reconstruct swap action balance commitment
    balance_1 = BalanceVar.from_negative_value_var(swap_plaintext.delta_1)
    balance_2 = BalanceVar.from_negative_value_var(swap_plaintext.delta_2)
    balance_1_commit = balance_1.commit(0) # will be blinded by fee
    balance_2_commit = balance_2.commit(0) # will be blinded by fee
    transparent_balance_commitment = balance_1_commit + balance_2_commit
    total_balance_commitment = transparent_balance_commitment + fee_commitment

    # balance commitment integrity check
    assert(claimed_balance_commitment == total_balance_commitment)

```

Once a block has successfully processed the transaction containing this trade, the user can claim the result of the trade (i.e. the swapped tokens). To do that, a swap claim proof must be provided in which the user provides the path to the committed swap plaintext in the commitment tree, and exchanges it (or converts it) into two committed balances of the two traded tokens (which can then be routed to output notes using output actions within the same transaction).

The pseudocode for this circuit is the following:

```
def swap_claim(private, public):
    # private inputs
    swap_plaintext = SwapPlaintextVar(private.swap_plaintext)
    claimed_swap_commitment = StateCommitmentVar(
        private.state_commitment_proof.commitment)
    position_var = PositionVar(private.state_commitment_proof.position)
    position_bits = position.to_bits_le()
    merkle_path = MerkleAuthVar(private.state_commitment_proof)
    nk = NullifierKeyVar(private.nk)
    lambda_1_i = AmountVar(private.lambda_1_i)
    lambda_2_i = AmountVar(private.lambda_2_i)
    note_blinding_1 = private.note_blinding_1
    note_blinding_2 = private.note_blinding_2

    # public inputs
    anchor = public.anchor
    claimed_nullifier = NullifierVar(public.nullifier)
    claimed_fee = ValueVar(public.claim_fee)
    output_data = BatchSwapOutputDataVar(public.output_data)
    claimed_note_commitment_1 = StateCommitmentVar(public.note_commitment_1)
    claimed_note_commitment_2 = StateCommitmentVar(public.note_commitment_2)

    # swap commitment integrity check
    swap_commitment = swap_plaintext.commit()
    assert(swap_commitment == claimed_swap_commitment)

    # merkle path integrity. Ensure the provided note commitment is in the TCT
    merkle_path.verify(position_bits, anchor, claimed_swap_commitment)

    # nullifier integrity
    nullifier = NullifierVar.derive(nk, position_var, claimed_swap_commitment)
    assert(nullifier == claimed_nullifier)

    # fee consistency check
    assert(claimed_fee == swap_plaintext.claim_fee)

    # validate the swap commitment's height matches the output data's height (i.e. the clearing price height)
    block = position_var.block  # BooleanVar[16..32] as FqVar
    note_commitment_block_height = output_data.epoch_starting_height + block
    assert(output_data.height == note_commitment_block_height)

    # validate that the output data's trading pair matches the note commitment's trading pair
    assert(output_data.trading_pair == swap_plaintext.trading_pair)

    # output amounts integrity
    computed_lambda_1_i, computed_lambda_2_i = output_data.pro_rata_outputs(
        swap_plaintext.delta_1_i, swap_plaintext.delta_2_i)
    assert(computed_lambda_1_i == lambda_1_i)
    assert(computed_lambda_2_i == lambda_2_i)

    # output note integrity
    output_1_note = NoteVar(address=swap_plaintext.claim_address, amount=lambda_1_i,
                            asset_id=swap_plaintext.trading_pair.asset_1, note_blinding=note_blinding_1)
    output_1_commitment = output_1_note.commit()

    output_2_note = NoteVar(address=swap_plaintext.claim_address, amount=lambda_2_i,
                            asset_id=swap_plaintext.trading_pair.asset_2, note_blinding=note_blinding_2)
    output_2_commitment = output_2_note.commit()

    assert(output_1_commitment == claimed_note_commitment_1)
    assert(output_2_commitment == claimed_note_commitment_2)

```

### Staking

Delegation and undelegation to a validator's pool are both done "half in the clear". With the undelegation part being subject to a delay.

Delegation happens by providing spend actions and proofs that spend committed Penumbra notes in the multi-asset shielded pool. A public *delegation action* is then used in conjunction to subtract the number of Penumbra tokens from the transaction's balance and add a calculated amount of the validator's token to the transaction's balance. All of these extra balances are committed in a non-hiding way so that they can interact with the exposed committed balances of the private actions. Finally, output actions and proofs can use the positive validator token balance to produce new note commitments in the multi-asset shielded pool (by exposing an equivalent amount of negative validator token balance).

Undelegation happens in two steps. The first step is public, and converts (via a *undelegation action*) the delegated tokens into *unbounding tokens*. Unbounding tokens are tokens which asset ids are derived from the validator's unique identity and the epoch in which the tokens were unbounded. As such, an undelegation action simply creates a negative balance of delegated tokens (which the transaction provides via spend actions) and a positive balance of unbounding tokens (which the transaction can store back into the multi-asset shielded pool via output actions).

Once a proper delay has been observed (for some definition of proper), a user can submit a new transaction containing an *undelegate claim action* to the network. This action finally consumes the unbounding tokens created previously, converting them back into Penumbra tokens after a penalty has been applied (in cases where the validator might have misbehaved). The proof accompanying the action ensures that the released balance commitment correctly contains both a positive balance of Penumbra tokens (after correctly applying the penalty) and a negative balance of unbounding tokens. As such, such an action is accompanied by spend actions matching the balance of unbounding tokens and output actions matching the balance of Penumbra tokens.

What follows is the pseudocode for the undelegate claim circuit:

```
def undelegate_claim(private, public):
    # private inputs
    unbonding_amount = AmountVar(private.unbounding_amount)
    balance_blinding = Uint8vec(private.balance_blinding)

    # public inputs
    claimed_balance_commitment = BalanceCommitment(public.balance_commitment)
    unbonding_id = AssetVarId(public.unbonding_id)
    penalty = PenaltyVar(public.penalty)

    # verify balance commitment
    expected_balance = penalty.balance_for_claim(
        unbonding_id, unbonding_amount)
    expected_balance_commitment = expected_balance.commit(balance_blinding)
    assert(claimed_balance_commitment == expected_balance_commitment)

```
