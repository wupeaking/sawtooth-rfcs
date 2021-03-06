- Feature Name: pbft_regular_view_changes
- Start Date: 2018-10-11
- RFC PR: https://github.com/hyperledger/sawtooth-rfcs/pull/29
- Sawtooth Issue: N/A

# Summary
[summary]: #summary

This RFC describes an extension to the PBFT implementation to mitigate the
effect of compromised leader nodes. The RFC takes into account leaders unfairly
ordering batches within blocks (the "unfair ordering" problem) and leaders
intentionally not producing new blocks to stall the network (the "silent
leader" problem).

# Motivation
[motivation]: #motivation

This RFC proposes a solution to the "unfair ordering" problem for Sawtooth
PBFT. In general, fair ordering means that a Byzantine node can not order
transactions in a way that unfairly benefits that node. This problem is
important in voting-type algorithms with a long-term leader, such as many PBFT
variants. A malicious leader can, depending on the protocol, participate in the
protocol perfectly but manipulate the ordering to its benefit by:

1. Manipulating the order of a given set of transactions
2. Generating and inserting new transactions into the ordering
3. Withholding submitted transactions from the ordering

One thing that makes this problem hard is that, without application-specific
knowledge from the transactions, it is difficult to tell if an ordering
actually benefits the leader or if "random noise" caused the ordering to be
substantially different than some expected or fair ordering.

The current implementation of PBFT does not prevent a leader from unfairly
ordering transactions. This is a problem when a leader node has an incentive to
do so. This RFC aims to mitigate this problem in the interest of making PBFT
more resilient to bad actors.

This RFC also proposes a mitigating solution to the "silent leader" problem.
Sawtooth PBFT checks for leader liveness by starting a timer when a leader
proposes a new block and endorses it with a pre-prepare message. If the timer
expires before the new block is committed, the leader is suspected of being
faulty and a view change is initiated. However, if a leader never proposes a
block or doesn't endorse a block with a pre-prepare message, no timer is
started. This means that a faulty leader can "remain silent" and stall the
network without the other nodes being able to determine whether the silence is
because no new batches are arriving, or the leader is intentionally ignoring
them.

                PrePreparing  <----------- Finished
    (start timer) |                           ^
                  v                           | (stop timer)
                Preparing -------------> Committing

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

In order to mitigate the effects of the "unfair ordering" and "silent leader"
problems, Sawtooth PBFT will perform view changes in two new cases.

In the first case, regular view changes are "forced" based on the number of
committed blocks. This mitigates the "unfair ordering" problem by giving every
node a chance to be unfair for a period of blocks. This is inspired by how the
Tendermint consensus algorithm handles the problem and how lottery-style
algorithms handle the problem, where a new leader can be elected for every
block.

In the second case, nodes propose a standard view change when the network is
idle for too long (the primary does not produce a block and a pre-prepare
message). This mitigates the "silent leader" problem by limiting the amount of
time a faulty leader can stall the network.

The setting `sawtooth.consensus.pbft.forced_view_change_period` determines how
often, measured in blocks, a view change should be forced. After transitioning
from the Finished to PrePreparing phase, a node will check whether a view change
should be forced and, if so, immediately change views. Specifically, the node
will not perform a view change by proposing a view change and collecting votes.

The setting `sawtooth.consensus.pbft.idle_timeout` is set to an integer and
represents the maximum time (in seconds) to wait in between committing a block
and starting a round of consensus on a new block before starting a view change.
After transitioning to the PrePreparing phase, the node will start a timer based
on this setting and, if the timer expires before a new block is proposed and
endorsed with a pre-prepare from the primary, will vote for a view change.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Let n_force := the configured period in blocks between view changes as read
from `sawtooth.consensus.pbft.forced_view_change_period` and t_idle := the time
to wait between a block being committed and a new block being proposed/endorsed
before forcing a view change as read from
`sawtooth.consensus.pbft.idle_timeout`.

Regular, forced view changes occur only when the validator transitions from the
Finished to the PrePreparing phase. Let n_seq := the sequence number after
transitioning to PrePreparing. (Note that the sequence number is incremented at
the same time as the block height and are identical.) Immediately after
transitioning to PrePreparing, the node will check if

    `n_seq % n_force == 0`

and if true, it will immediately change view.

Whenever a node transitions to the PrePreparing phase, a timer is started which
will expire after t_idle has passed. If this timer expires without a BlockNew
update and matching PrePreparing arriving, the node will propose a view change.
A separate timer is used here instead of moving where the existing timer is
started for two reasons. First, an idle network should not affect how long a
network has to commit a block after proposing one. Second, it is desirable that
the time a network has to commit a block and the time a network can remain idle
before proposing a view change be independently configurable.

# Drawbacks
[drawbacks]: #drawbacks

There are two drawbacks to these changes:

1. They do not completely solve the unfair ordering and silent leader problems
2. They rely on false positives being okay, which means more view changes than
   necessary are performed

# Rationale and alternatives
[alternatives]: #alternatives

A couple alternatives to the unfair ordering problem were considered. The first
was to have followers compute a heuristic or "probability that the ordering is
fair" and to start a new view change if they determine the leader appears to be
manipulating the order. The second was inspired by the proposed method for
handling non-determinism in the original PBFT paper by Castro and Liskov.

The following is an example of the type of heuristic that could be used to
prevent excluding batches indefinitely.

1. When a batch arrives, add it to a queue and include a "skip count" and the
   queue size when it was added
2. When a block is committed, remove batches from the front of the queue until
   all batches in the block have been removed. If a batch is not in the block,
   increase its skip count and save it. Push all batches that were skipped back
   onto the front of the queue, preserving the original order.
3. While pushing batches back onto the queue, check that the batch isn't being
   intentionally skipped by the validator:

    a. Check if the skip count (S) is greater than the size of the queue when
       it was added (q_add) divided by the average throughput (T_avg)

          S > (q_add / T_avg)

    b. T_avg = sum((batches / block) for last N blocks) / N

4. If a batch is being intentionally skipped, consider the leader faulty and
   propose a view change.

Note: This doesn't prevent a bad leader from re-ordering batches, but does
require that it include them eventually, even on a busy network.

For the second alternative, the relevant portion of the PBFT paper is section
4.6 on non-determinism. (Paper found here:
http://pmg.csail.mit.edu/papers/osdi99.pdf) If we treat the batch ordering as
non-deterministic (because of network latency, dictionary iteration order,
etc.), then we can use some application of what is described in 4.6. An example
would be to have all followers vote on the batch ordering for the next block
before it is published and to require that the leader decide which batches to
include by some deterministic computation over 2f+1 of the votes. This creates
a smaller new problem to deal with, which is that malicious followers can vote
for bogus batch ids that must be solved.

It was determined that a complete solution to the problems described was beyond
the scope of this implementation of PBFT and a complete solution should come in
the form of an implementation of a new algorithm based on PBFT.

# Prior art
[prior-art]: #prior-art

The Tendermint consensus protocol also solves the unfair ordering problem by
electing a new leader for every block. This can be viewed as the same solution
as described above, with the number of blocks between view changes configured
to 1.

# Unresolved questions
[unresolved]: #unresolved-questions

- What should the default values for the new settings be?
- Should forced view changes be enabled by default? Should it be possible to
  turn them off?
