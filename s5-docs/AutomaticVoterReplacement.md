# Automatic voter replacement after sustained unreachability

**Status: proposed design; MicroRaft demotion support implemented, S5 planner not implemented.** This
document records the motivation, alternatives, and selected direction for automatically restoring
voter redundancy when a voter stays offline. The MicroRaft membership support (`DEMOTE_TO_LEARNER`)
exists on the `s5-branch` fork; the S5 policy and planner work that uses it is still to do.

## Motivation and context

S5 targets three voters per Raft group. Three voters have a quorum of two and tolerate one voter
failure. A fourth cluster node normally joins each group as a learner: it receives the replicated
state but does not vote. When a voter becomes unreachable, the other two voters can continue serving
and can commit writes, but the group has lost its ability to tolerate another failure. The learner
does not restore that lost voting redundancy by itself.

The current policy leaves a three-voter group unchanged during ordinary unreachability. That avoids
reshaping membership for a temporary network interruption and lets the returning voter resume its
role. The policy is safe for service availability, but it can leave the cluster degraded for an
unknown length of time. “Temporary” has no useful fixed meaning: a node may be unreachable for ten
seconds, ten minutes, or a day, and the cluster cannot know in advance whether it will return.

This became concrete during native-cluster voter replacement and restoration. Mac101 was removed and
Mac89 took its voter seat. Mac101 later returned as a newly identified learner and caught up; Mac89
was then removed and rejoined as a learner. The cluster recovered three voters and retained both
machines as learners, but this required operator-driven removal and fresh node identities. The work
and observed outcome are recorded in
[`NativeClusterVoterReplacement-2026-10-10.md`](../qualification/NativeClusterVoterReplacement-2026-10-10.md).
That exercise showed the replacement sequence can restore voting capacity, while also exposing that
the current membership API cannot simply turn an existing voter into a learner and later let it
resume as that same member.

## Alternatives discussed

### Keep the voter set unchanged until the node returns

This is the current behavior for ordinary outages. The remaining two voters keep quorum, and the
original voter can resume voting when it reconnects.

This avoids unnecessary membership changes when an outage is brief. Its cost is that the cluster
stays one failure away from losing quorum for as long as the node is absent. The policy has no bound
on that degraded period and does not use an available, synchronized learner to restore redundancy.

### Require an operator to declare permanent loss and remove the node

An operator can explicitly remove a node, after which voter planning replaces it with a reachable
learner. This distinguishes planned decommissioning from ordinary network failure and avoids
automatically acting on a short interruption.

The problem is that permanent loss cannot be known with certainty either. The operator must choose
when an outage has lasted long enough, and recovery currently requires a new Raft identity and clean
membership state for the returning machine. This is workable as a recovery procedure, but it does
not automatically restore redundancy on a predictable schedule.

### Replace after a configured offline deadline, then demote the returning voter

The selected direction is to treat sustained unreachability as a membership decision after a
configured deadline. The cluster need not classify the node as permanently lost. It only decides
that the node has been absent long enough that its voter seat should be used by an available learner.
If the old voter returns later, it catches up as a learner and stays a learner until normal policy
promotes it again.

This restores redundancy without waiting to learn whether the outage is permanent. It also turns a
long outage into predictable behavior, at the cost of a membership transition and a period in which
the cluster has less tolerance while that transition is committed.

## Proposed behavior

For each Raft group, when a voter has been continuously unreachable for the configured deadline
(initial proposal: **60 seconds**):

1. Confirm the group still has a quorum and identify a registered learner that is reachable and
   sufficiently caught up to take a voter seat.
2. Promote that learner through a committed Raft membership change.
3. Demote the offline voter to learner through a second committed membership change, restoring the
   target of three voters.
4. Report the transition and resulting voter/learner roles in cluster status and logs.
5. When the old voter reconnects, let it receive and apply the committed membership state, catch up
   as a learner, and remain a learner.

With three voters A, B, and X, and learner L, if X is offline while A and B remain healthy, promoting
L temporarily creates four voters and requires three votes. A, B, and L provide those three votes.
Demoting X then leaves A, B, and L as three voters with quorum two. Each change must be committed
before the next is proposed. If the learner is unavailable or not caught up, or if the group cannot
commit the transition safely, the group remains degraded and reports why; it must not force a
membership change.

The 60-second value is a starting policy, not a claim that every deployment has the same network or
sleep behavior. It should be configurable. Status should show the last successful contact, time
remaining before the deadline, the pending replacement candidate, and any blocked condition. A
recovered voter before the deadline keeps its voter role; once a replacement transition commits,
recovery does not silently take the seat back.

## MicroRaft work

### Implemented: `DEMOTE_TO_LEARNER`

The fork adds `MembershipChangeMode.DEMOTE_TO_LEARNER`. It commits a membership entry that keeps the
member in `members` and removes it only from `votingMembers`, so the member keeps its Raft identity and
log history and keeps receiving replicated entries. It differs from `REMOVE_MEMBER`, which terminates
the removed node and prevents the same identity from rejoining.

Behavior:

- The target must be a current voting member. A leader cannot demote itself (transfer leadership
  first). A demotion that would create a third learner (`MAX_LEARNER_COUNT = 2`) is refused.
- A node that applies an entry removing its own vote switches to `LEARNER` and persists
  `voting=false`, so it stops pre-vote and election timeouts. If the entry is truncated before
  commit, the node is promoted back to its committed role.
- The same identity can later be promoted again with `ADD_OR_PROMOTE_TO_FOLLOWER`.
- Only one membership change can be pending at a time, and a new leader must commit an entry in its own
  term before it can change membership. Callers pass the expected group-members commit index.

Covered by unit tests (`MembershipChangeTest`, `PersistenceTest`): demote then promote back, leader and
learner demotion refused, restart of a demoted follower (restores `LEARNER`, does not campaign),
demotion of a partitioned follower followed by healing (no term change), and truncation of an
uncommitted demotion on the demoted follower (reverts to voter).

### Not implemented in MicroRaft

- **Atomic promote-and-demote is deliberately not provided.** Replacing `{A,B,X}` with `{A,B,L}` in
  one entry changes two voters at once, and the old and new majorities (`{A,X}` and `{B,L}`) can be
  disjoint. Single-server membership change is only safe one server per entry; a swap would need
  joint consensus, which MicroRaft does not have. The two-step sequence below is the safe form.
- **Vote-request guard.** Vote and pre-vote requests from a known non-voter are not rejected. A
  receiver with a stale config could otherwise refuse a legitimately promoted node. A demoted node
  does not campaign because it is a learner.
- **Further tests:** restart between the promotion and the demotion, snapshot install on a node being
  demoted, and the SQLite store persisting `voting=false`.
- **Liveness edge case:** if a leader appends a demotion that only the demoted follower receives and
  then dies, that follower is already a learner and refuses votes, so the remaining voters may not be
  able to elect a leader until the old leader returns. This needs a test and a decision.
- **Rolling upgrade:** an older node that receives the new mode value is not handled.
- **Learner readiness:** MicroRaft exposes no "caught up" flag. The planner must compare the learner's
  match index with the leader's commit index from `RaftNodeReport`.

## S5 planner work

Sequencing is owned by the S5 planner, not MicroRaft. The planner must be written as an idempotent
"converge to three voters" loop, not as a remembered two-step transaction, because leadership can
change between the steps.

- **Order:** promote the learner first, wait for the commit, then demote the offline voter. Promote
  first keeps the learner holding the entries before the vote is withdrawn. Demote first would leave
  two voters with no failure tolerance.
- **Window:** while four voters exist with one offline, an election needs A, B, and L all reachable. A
  leader crash in this window blocks elections until a voter returns. Committing entries needs fewer
  acks (see `RaftState.logReplicationQuorumSize`), but the membership entries themselves are committed
  with the full quorum.
- **Resume after failover:** a new leader that sees four voters, one unreachable, finishes the
  demotion.
- **Concurrency:** pass the current committed members index on each call. On
  `MismatchingRaftGroupMembersCommitIndexException`, re-read and re-plan. `CannotReplicateException`
  on a fresh leader is retryable.
- **Leader-only policy:** only the leader evaluates the deadline, so two nodes never propose competing
  changes. A local timeout alone must never alter voter membership outside consensus.

S5 should add planner and runtime tests for the deadline, candidate readiness, committed transition
order, recovery before and after the deadline, leader change between steps, and blocked replacement.

## Scope and non-goals

This proposal replaces only the policy for sustained voter unreachability when a safe learner
replacement is available. It does not convert every brief packet loss into a reconfiguration, and it
does not claim a node is permanently destroyed. It does not change blob replication or placement; those
have their own repair and safety rules. It does not permit a group that lacks quorum to reconfigure
itself. Quorum-lost recovery remains a separate operator-controlled procedure.

## Decision

Adopt a configurable offline deadline, initially 60 seconds, after which a reachable and caught-up
learner may be promoted and the unreachable voter demoted to learner through two committed Raft
membership changes. A returning node catches up as a learner. The MicroRaft demotion mode exists;
the S5 planner and its tests must land before this behavior is enabled.

Until then, S5's current behavior remains in effect: ordinary unreachability does not change voter
membership; explicit node removal can trigger replacement using the existing remove-based path.
