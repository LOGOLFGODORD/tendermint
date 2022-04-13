# RFC 017: ABCI++ Vote Extension Propagation

## Changelog

- 11-Apr-2022: Initial draft (@sergio-mena).

## Abstract

According to the
[ABCI++ specification](https://github.com/tendermint/tendermint/blob/4743a7ad0/spec/abci%2B%2B/README.md)
(as of 11-Apr-2022), a validator MUST provide a signed vote extension for each non-`nil` precommit vote
of height *h* that it uses to propose a block in height *h+1*. When a validator is up to
date, this is easy to do, but when a validator needs to catch up this is far from trivial as this data
cannot be retrieved from the blockchain.

This RFC presents and compares the different options to address this problem, which have been proposed
in several discussions by the Tendermint Core team.

## Document Structure

The RFC is structured as follows. In the [Background](#background) section,
subsections [Problem Description](#problem-description) and [Cases to Address](#cases-to-address)
explain the problem at hand from a high level perspective, i.e., abstracting away from the current
Tendermint implementation. In contrast, subsection
[Current Catch-up Mechanisms](#current-catch-up-mechanisms) delves into the details of the current
Tendermint code.

In the [Discussion](#discussion) section, subsection [Solutions Proposed](#solutions-proposed) is also
worded abstracting away from implementation details, whilst subsections
[Feasibility of the Proposed Solutions](#feasibility-of-the-proposed-solutions) and
[Current Limitations and Possible Implementations](#current-limitations-and-possible-implementations)
analize the viability of one of the proposed solutions in the context of Tendermint's architecture
based on reactors. Finally, [Formalization Work](#formalization-work) brifely discusses the work
still needed demonstrate the correctness of the chosen solution.

## Background

This document assumes that all validators have equal voting power for the sake of simplicity. This is done
without loss of generality.

### Basic Definitions

TODO: Precommit votes, precommit messages

TODO: Re-read "commits do not contain nil votes", catchup messages

TODO: Mention somewhere that statesync is not really relevant for this problem... only for the first block after statesync'ing.

### Problem Description

In the version of [ABCI](https://github.com/tendermint/spec/blob/4fb99af/spec/abci/README.md) present up to
v0.35, for any height *h*, a validator *v* MUST have the decided block *b* and a *commit* for height *h* in
order to decide at height *h*, and propose at height *h+1* in the rounds of *h+1* where *v* is a proposer.
This *commit* consists of more than *2n/3* precommit votes voting for *b*, where *n* is the size of the
validator set at height *h*.

In [ABCI++](https://github.com/tendermint/tendermint/blob/4743a7ad0/spec/abci%2B%2B/README.md),
the information that a validator *v* MUST have to be able to decide in height *h* is the same:
the decided block *b* and a *commit* for height *h*. In contrast, *v* MUST have *b* and an
*extended commit* in order to propose at height *h+1*.
The *extended commit* is a *commit* where a mandatory vote extension is attached to every
non-`nil` vote in the *commit*. This mandatory vote extension can be empty, but needs to be signed.
The signing is carried out with the sending validator's private key (the same as the vote it extends),
but is independent from the vote, so a vote can be separated from its extension.
The reason for vote extensions to be mandatory is that, otherwise, a (malicious) node can omit a vote
extension while still providing/forwarding/sending the corresponding precommit vote.

When a validator takes an active part in consensus at height *h*, it has all the data it needs in memory,
in its consensus state, to decide on *h* and propose in *h+1*. Things are not so easy in the cases when
*v* cannot take part in consensus because it is late (e.g., it falls behind, it crashes
and recovers, or it just starts after the others). If *v* does not take part, it cannot actively
gather precommit messages (which include vote extensions) in order to decide.
Before ABCI++, this wasn't a problem: full nodes are supposed to persist past blocks in the block store,
so other nodes would realise that *v* is late and send it the missing decided block at height *h* and
the corresponding commit (kept in block *h+1*) so that *v* can catch up.
However, we cannot apply this catch-up technique for ABCI++, as the vote extensions, which are part
of the needed *extended commit* are not part of the blockchain.

### Cases to Address

Before we tackle the description of the possible cases we need to address, let us describe the following
incremental improvement to the ABCI++ logic. Upon decision, a full node persists (e.g., in the block
store) the extended commit that allowed the node to decide. For the moment, let us assume the node only
needs to keep its *most recent* extended commit, and MAY remove any older extended commits from persistent
storage.
This improvement is so obvious that all solutions descibed in the [Discussion](#discussion) section use
it as a building block. Moreover, it completely addresses by itself some of the cases described in this
subsection.

We now describe the cases (i.e. possible *runs* of the system) that have been raised in different
discussions and need to be addressed. They are (roughly) ordered from easiest to hardest to deal with.

- **(a)** *Happy path: all validators advance together, no crash*.

    This case is included for completeness. All validators have taken part in height *h*.
    Even if some of them did not manage to send a precommit message for the decided block, they all
    receive enough precommit messages to be able to decide. As vote extensions are mandatory in
    precommit messages, every validator *v* trivially has all the information, namely the decided block
    and the extended commit, needed to propose in height *h+1* for the rounds in which *v* is the
    proposer.

    No problem to solve here.

- **(b)** *Lagging majority*.

    Also included for completeness. This case is not possible by the nature of the Tendermint algorithm,
    which requires more than *2n/3* prevote and precommit votes in order to make progress. So only up to
    *n/3* validators can lag behind.

    No problem to solve here.

- **(c)** *All validators advance together, then all crash at the same height*.

    This case has been raised in some discussions, the main concern being whether the vote extensions
    for the previous height would be lost across the network. With the improvement described above,
    namely persisting the latest extended commit at decision time, this case is solved.
    When a crashed validator recovers, it recovers the last extended commit from persistent storage
    and handshakes with the Application.
    If need be, it also reconstructs messages for the unfinished height
    (including all precommits received) from the WAL.
    Then, the validator can resume where it was at the time of the crash. Thus, as extensions are
    persisted, either in the WAL (in the form of received precommit messages), or in the latest
    extended commit, the only way that vote extensions needed to start the next height could be lost
    forever would be if all validators crashed and never recovered (e.g. disk corruption).
    Since a *correct* node MUST eventually recover, this violates Tendermint's assumption of more than
    *2n/3* correct validators.

    No problem to solve here.

- **(d)** *Enough validators crash to block the rest*.

    In this case, blockchain progress halts, i.e. surviving full nodes keep increasing rounds
    indefinitely, until some of the crashed validators are able to recover.
    Those validators that recover first will handshake with the Application and recover at the height
    they crashed, which is still the same the nodes that did not crash are stuck in, so they don't need
    to catch up.
    Further, they had persisted the extended commit for the previous height. Nothing to solve.

    For those validators recovering later, we are in case (g) below.

- **(e)** *Some validators crash, but not enough to block progress*.

    When the correct processes that crashed recover, they handshake with the Application and resume at
    the height they were at when they crashed. As the blockchain did not stop making progress, the
    recovered processes are likely to have fallen behind with respect to the progressing majority.

    At this point, the recovered processes are in case (g) below.

- **(f)** *A new full node starts*.

    The reasoning here also applies to the case when more than one full node are starting.
    When the full node starts from scratch, it has no state (its current height is 0). Ignoring
    statesync for the time being, the node just needs to catch up by applying past blocks one by one
    (after verifying them).

    Thus, the node is in case (g) below.

- **(g)** *Advancing majority, lagging minority*

    In this case, a set of full nodes, of which *n/3* or less are validators, fall behind an arbitrary
    number of heights (e.g., temporary disconnection or network partition, memory thrashing, crashes,
    new nodes).
    These lagging nodes then need to catch up. They have to obtain the information they need to make
    progress from other nodes. For each height *h*, this includes the decided block for *h*, and the
    precommit votes also for *deciding h* (which can be extracted from the block at height *h+1*).
    At a given height, say *h_c*, a node will consider itself *caught up*, based on the (maybe out of
    date) information it is getting from its peers. Then, the node needs to be ready to propose
    at height *h_c+1* which entails having received the vote extensions for *h_c*.

    As the vote extensions are *not* stored in the blockchain, and it is difficult to have strong
    guarantees on *when* a late node considers itself caught up, providing the late note with the right
    vote extensions for the right height poses a problem.

At this point, we have described and compared all cases raised in discussions leading up to this
RFC. The list above aims at being exhaustive. The analysis of each case included above makes all of
them converge into case (g).

### Current Catch-up Mechanisms

We now briefly describe the current catch-up mechanisms in the reactors concerned in Tendermint.

#### Blocksync

TODO

#### Consensus Reactor

TODO

> TODO: I have considered *inserting* statesync somehow in this RFC. Do folks think this is needed?

## Discussion

### Solutions Proposed

These are the solutions proposed in discussions leading up to this RFC.

- **0.** *Vote extensions are made **best effort** in the specification*.

    This is the simplest solution, considered as a way to provide vote extensions in a simple enough
    way so that it can be part of v0.36.
    It consists in changing the specification so as to not *require* that precommit votes used upon
    `PrepareProposal` contain their corresponding vote extensions. In other words, we render vote
    extensions optional.
    There are strong implications stemming from such a relaxation of the original specification.

    - As a vote extension is signed *separately* from the vote it is extending, an intermediate node
      can now remove (i.e., censor) vote extensions from precommit messages at will.
    - Further, there is no point anymore in the spec requiring the Application to accept a vote extension
      passed via `VerifyVoteExtension` to consider a precommit message valid in its entirety. Remember
      this behavior of `VerifyVoteExtension` is adding a constraint to Tendermint's conditions for
      liveness.
      In this situation, it is better and simpler to just drop the vote extension rejected by the
      Application via `VerifyVoteExtension`, but still consider the vote itself valid as long as its
      signature verifies.

- **1.** *Include vote extensions in the blockchain*.

    Another obvious solution, which has somehow been considered in the past, is to include the vote
    extensions and their signatures in the blockchain.
    The blockchain would thus include the extended commit, rather than a regular commit, as the structure
    to be canonicalized in the next block.
    With this solution, the current mechanisms implemented both in the blocksync and consensus reactors
    would still be correct, as all the information a node needs to catch up, and to start proposing when
    it considers it is caught up, can now be recovered from past blocks.

    This solution has two main drawbacks.

    - It changes the block format, in particular, the way light blocks are verified; thus breaking
      backwards compatiblity in features such as light clients and IBC (the latter relying on the former).
    - The extra information (i.e., the vote extensions) that is now kept in the blockchain is not really
        needed *at every height* for a late node to catch up.
        - This information is only needed to be able to *propose* at the height the validator considers
          itself as caught-up. If a validator is indeed late for height *h*, it is useless (although
          correct) for it to call `PrepareProposal`, or `ExtendVote`, since the block is already decided.
        - Moreover, some uses cases require pretty sizeable vote extensions, which would result in an
          important waste of space in the blockchain.

- **2.** *Skip* propose *step in Tendermint algorithm*.

    This solution consists in modifying the Tendermint algorithm to skip the *send proposal* step in
    heights where the node does not have the required vote extensions to populate the call to
    `PrepareProposal`. The main idea behind is that this should only happen when the validator is late
    and, therefore, up-to-date validators have already proposed (and decided) for that height.
    A small variation of this solution is, rather than skipping the *send proposal* step, the validator
    sends a special *empty* or *bottom* proposal to signal other nodes that it is not ready to propose
    at (any round of) the current height.

    The appeal of this solution is its simplicity. A possible implementation does not need to extend
    the data structures, or change the current catch-up mechanisms implemented in the blocksync or
    in the consensus reactor. When we lack the needed information (vote extensions), we simply rely
    on another correct validator to propose a valid block in other rounds of the current height.

    However, this solution can be attacked by a byzantine node in the network in the following way.
    Let us consider the following scenario:

    - all validators send out precommit messages, with vote extensions, for height *h*, round *r*,
      roughly at the same time,
    - all validators then wait until they gather enough precommit messages for height *h*, round *r*,
      in order to decide in height *h*,
    - all those precommit messages (and any further message for height *h*) get delayed indefinitely,
    - an intermediate (malicious) node *m* manages to gather more than *2n/3* of the precommit messages
      for height *h*, round *r*, as well as the proposed block the precommit messages refer to,
    - node *m* uses the precommit messages to build a commit (not extended commit), which it then sends
      to *all* validators in order to convince them they are late,
    - all validators receive the *catch-up* message from *m*, decide on height *h*, and proceed to
      height *h+1*.

    At this point, *all* validators have advanced to *h+1* beleiving they are late, and so, expecting
    the *supposed* leading majority of validators to propose for *h+1*. As a result, the blockhain progress
    grinds to a halt. Furthermore, if the malicious node decides to *disappear* (together with the
    precommit messages), we have a situation we cannot recover from if the *p2p* layer is not able to,
    eventually, relay more than *2n/3* precommit messages sent originally for height *h*, round *r*.

- **3.** *Require extended commits to be available at switching time*.

    This is more involved than all previous solutions, and builds on an idea used in solution 2:
    vote extensions are actually not needed for Tendermint to make progress as long as the
    validator is *certain* it is late.

    We define two modes. The first is denoted *catch-up mode*, and Tendermint only calls
    `FinalizeBlock` for each height when in this mode. The second is denoted *consensus mode*, in
    which the validator considers itself up to date and calls `PrepareProposal`, `ExtendVote`, and `VerifyVoteExtension`, before calling `FinalizeBlock`, as prescribed by the
    [ABCI++ specification](https://github.com/tendermint/tendermint/blob/4743a7ad0/spec/abci%2B%2B/README.md).

    The catch-up mode does not need vote extension information to make progress, as all it needs is the
    decided block at each height to call `FinalizeBlock` and keep the state-machine replication making
    progress. The consensus mode, on the other hand, does need vote extension information at every height.

    When a validator falls behind for whatever reason, e.g. cases (d), (e), (f), and (g) above, it
    follows the catch-up mode as long as it considers itself out of date. The key safety property we need
    to provide is the following:

    - for every height *h*, a node in catch-up mode does not accept a (gossipped) block for height
        *h+1* **until**:
        - it has received and (light-client) verified all the blocks leading up to a new height
          *h' > h*
        - it has received an extended commit for *h'* and has verified:
            - the precommit vote signatures in the extended commit
            - the vote extension signatures in the extended commit: each is signed with the same
              key as the precommit vote it extends

    This invariant ensures that the node in catch-up mode:

    - cannot be attacked by being fed a block without extensions in a similar way as explained for
      solution 2.
    - at any moment, there exists a future height *h'* for which the node holds the (light-client
      verified) block and *a* corresponding (verified) extended commit.

    The second bullet above allows a node to have the *flexibility* to decide when to consider itself
    as caught up (independently of any network conditions) and switch to consensus mode by using the
    extended commit for *h'* to build a proposal in the rounds of *h' + 1* where it is the proposer.

    Of course, the node may fall behind later on. In that case, it rejects switching back to consensus
    mode until it receives a sequence of blocks and an extended commit that abides by the invariant
    specified above.

### Feasibility of the Proposed Solutions

Solution 0, besides the drawbacks described in the previous section, provides guarantees that are
weaker than the rest. The Application does not have the assurance that more than *2n/3* vote
extensions will *always* be available when calling `PrepareProposal`.
This level of guarantees is probably not strong enough for vote extensions to be useful for some of
the use cases that motivated them in the first place, e.g., encrypted mempool transactions.

Solution 1, while being simple in that the changes needed in the current Tendermint codebase would
be rather small, is changing the block format, and would therefore require all blockchains using
Tendermint v0.35 or earlier to hard-fork when upgrading to v0.36.

Since Solution 2 can be attacked, one might prefer Solution 3, even if it is more involved
to implement. Further, we must elaborate on how we can turn Solution 3, described in abstract
terms in the previous section, into a concrete implementation compatible with the current
Tendermint codebase.

### Current Limitations and Possible Implementations

The main limitations affecting the current version of Tendermint are the following.

- The current version of the blocksync reactor does not use the full
  [light client verification](https://github.com/tendermint/tendermint/blob/4743a7ad0/spec/light-client/README.md) algorithm to validate blocks coming from other peers.
- The code being structured into the blocksync and consensus reactors, only switching from the
  blocksync reactor to the consensus reactor is supported; switching in the opposite direction is
  not supported. Alternatively, the consensus reactor could have a mechanism allowing a late node
  to catch up by skipping calls to `PrepareProposal`/`ProcessProposal`, and
  `ExtendVote`/`VerifyVoteExtension` and only calling `FinalizeBlock` for each height.
  Such a mechanism does not exist at the time of writing this RFC.

The blocksync reactor featuring light client verification is being actively worked on (tentatively
for v0.37). So it is best if this RFC does not try to solve that problem, but just makes sure
the outcomes are compatible with that effort (TODO: check this paragraph with Callum/Jasmina).

In subsection [Cases to Address](#cases-to-address), we concluded that we can focus on
solving case (g) in theortical terms.
However, as the current Tendermint version does not yet support switching back to blocksync once a
node has switched to consensus, we need to split case (g) into two cases. When a full node needs to
catch up...

- **(g.1)** ... it has not switched yet from the blocksync reactor to the consensus reactor, or

- **(g.2)** ... it has already switched to the consensus reactor.

This is important in order to discuss the different possible implementations.

#### Base Implementation: Persist and Propagate Extended Commit History

In order to circumvent the fact that we cannot switch from the consensus reactor back to blocksync,
rather than just keeping the few (TODO: 1?, 2?) most recent extended commits, nodes will need to keep
and gossip a backlog of extended commits so that the consensus reactor can still propose and decide
in out-of-date heights.

The base implementation (for which an experimental patch exists) consists in the conservative
approach of persisting in the block store *all* extended commits for which we have also stored
the full block. Currently, when statesync is run at startup, saves light blocks (incomplete block
containing the header and commit, but, e.g., no transactions). This implementation does not seek
to receive or persist those light blocks as they would not be of any use.

Then, we modify the blocksync reactor so that peers *always* send requested full blocks together
with the corresponding extended commit in the `BlockResponse` messages. This guarantees that the
block store being reconstructed by blocksync has the same information as that of peers that are
up to date.
Thus, blocksync has all the data it requires to switch to the consensus reactor, as long as:

- The node is still at height 0 (where no commit or extended commit is needed)
- The node has processed at least 1 block in blocksync

As a side note, a chain might be started at a height *h_i > 0*, all other heights less than
*h_i* being non-existent. In this case, the chain is still considered to be at height 0 before block
*h_i* is applied (so the condition above is still correct for this case).

Additionally, when a validator falls behind while having already switched to the consensus reactor,
peer nodes have enough information in their block store to reconstruct the extended votes and send
them to the validator falling behind.

This solution requires a few changes to the consensus reactor:

- upon saving the block for a given height in the block store at decision time, save the
  corresponding extended commit as well
- in the catch-up mechanism, when a node realizes that another peer is more than 2 heights
  behind, it uses the extended commit (rather than the canoncial commit as done previously) to
  reconstruct the votes with their corresponding extensions

The changes to the blocksync reactor are more substantial:

- the `BlockResponse` message is extended to include the extended commit of the same height as
  the block included in the response (just as they are stored in the block store)
- structure `bpRequester` is likewise extended to hold the received extended commits coming in
  `BlockResponse` messages
- method `PeekTwoBlocks` is modified to also return the extended commit corresponding to the first block
- when successfully verifying a received block, the reactor saves its corresponding extended commit in
  the block store

The two main drawbacks of this base implementation are:

- the increased size taken by the block store, in particular with big extensions
- the increased bandwith taken by the new format of `BlockResponse`

#### Possible Optimization: Pruning the Extended Commit History

If we cannot switch from the consensus reactor back to the blocksync reactor, and we do not have
the catch-up mechanism in the consensus reactor explained above, we cannot prune the extended commit
backlog in the block store without sacrificing the implementation's correctness. The asynchronous
nature of our distributed system model allows a process to fall behing an arbitrary number of
heights, and thus all extended commits need to be kept *just in case* a node that late had
previously switched to the consensus reactor.

However, there is a possibility to optimize the base implementation. We could prune from the block
store all extended commits that are more than *d* heights in the past. Then, we need to handle two
new situations, roughly equivalent to cases (g.1) and (g.2) described above.

- (g.1) A node starts from scratch or recovers after a crash, and (for simplicity of explanation)
  it does not run statesync.
  In this case, the blocksync reactor needs to be improved to make sure it cannot switch to consensus
  until it has received a valid extended commit for a height in its future. This extended commit
  will allow the node to switch from the blocksync reactor to the consensus reactor and immediately
  act as a proposer if required.
- (g.2) A node already running the consensus reactor falls behind beyond *d* heights. In principle,
  the node will be stuck as no other node can provide the vote extensions it needs to make progress.
  However we can have the node crash and recover as a workaround. This effectively converts this case
  into the previous one.

### Formalization Work

A formalization work to show or prove the correctness of the different use cases and solutions
presented here (and any other that may be found) needs to be carried out.
A question that needs a precise answer is how many extended commits (one?, two?) a node needs
to keep in persistent memory when implementing Solution 3 described above without Tendermint's
current limitations.
Another important invariant we need to prove formally is that the set of vote extensions
required to make progress will always be held somewhere in the network.

## References

- [ABCI++ specification](https://github.com/tendermint/tendermint/blob/4743a7ad0/spec/abci%2B%2B/README.md)
- [ABCI as of v0.35](https://github.com/tendermint/spec/blob/4fb99af/spec/abci/README.md)
- [Vote extensions issue](https://github.com/tendermint/tendermint/issues/8174)
- [Light client verification](https://github.com/tendermint/tendermint/blob/4743a7ad0/spec/light-client/README.md)