# Automatic voter replacement after sustained unreachability

**Status: proposed design; S5 planner not implemented.** This document records the motivation,
alternatives, and selected direction for automatically restoring voter redundancy when a voter stays
offline. The first version needs no MicroRaft change: it uses the existing promote and remove
operations. An optional second version uses `DEMOTE_TO_LEARNER`, which is implemented on the
`s5-branch` fork, so a returning voter can rejoin as the same member.

## Motivation and context

S5 targets three voters per Raft group. Three voters have a quorum of two and tolerate one voter
failure. A fourth cluster node normally joins each group as a learner: it receives the replicated
state but does not vote. When a voter becomes unreachable, the other two voters can continue serving
and can commit writes, but the group has lost its ability to tolerate another failure. The learner
is never promoted by itself, so even with a healthy, synchronized learner available the group keeps
the same voter set and stays at risk. If the offline voter never returns, it stays that way forever.

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
That exercise showed the replacement sequence restores voting capacity using the existing membership
operations. What it lacked was an automatic trigger, and a way for a returning node to rejoin without
operator action and a new identity.

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
when an outage has lasted long enough, and the cluster stays degraded until they act.

### Replace automatically after a configured offline deadline (selected)

The selected direction is to treat sustained unreachability as a membership decision after a
configured deadline. The cluster need not classify the node as permanently lost. It only decides
that the node has been absent long enough that its voter seat should be used by an available learner.

This restores redundancy without waiting to learn whether the outage is permanent. It also turns a
long outage into predictable behavior, at the cost of a membership transition and a period in which
the cluster has less tolerance while that transition is committed.

There are two ways to retire the offline voter once its replacement is a voter:

- **Remove it (`REMOVE_MEMBER`).** Available today. The node leaves the group. If it returns, it is
  no longer a member and must be wiped and re-added with a fresh Raft identity.
- **Demote it (`DEMOTE_TO_LEARNER`).** The node stays a member without a vote. If it returns, it
  catches up and settles as a learner by itself. If it never returns, it occupies one of the two
  learner slots until it is removed, so this needs a second, longer deadline for cleanup.

## Proposed behavior

For each Raft group, when a voter has been continuously unreachable for the configured deadline
(initial proposal: **60 seconds**):

1. Confirm the group still has a quorum and identify a registered learner that is reachable and
   sufficiently caught up to take a voter seat.
2. Promote that learner through a committed Raft membership change (`ADD_OR_PROMOTE_TO_FOLLOWER`).
3. Retire the offline voter through a second committed membership change, restoring the target of
   three voters:
   - **Version 1:** `REMOVE_MEMBER`.
   - **Version 2:** `DEMOTE_TO_LEARNER`, plus a longer deadline after which a demoted node that is
     still offline is removed.
4. Report the transition and resulting voter/learner roles in cluster status and logs.
5. When the old voter reconnects:
   - **Version 1:** it is a stale, removed node. An operator (or later tooling) wipes it and re-adds
     it as a new learner.
   - **Version 2:** it receives and applies the committed membership state, catches up as a learner,
     and remains a learner until normal policy promotes it again.

With three voters A, B, and X, and learner L, if X is offline while A and B remain healthy, promoting
L temporarily creates four voters and requires three votes. A, B, and L provide those three votes.
Retiring X then leaves A, B, and L as three voters with quorum two. Each change must be committed
before the next is proposed. If the learner is unavailable or not caught up, or if the group cannot
commit the transition safely, the group remains degraded and reports why; it must not force a
membership change.

The 60-second value is a starting policy, not a claim that every deployment has the same network or
sleep behavior. It should be configurable. Status should show the last successful contact, time
remaining before the deadline, the pending replacement candidate, and any blocked condition. A
recovered voter before the deadline keeps its voter role; once a replacement transition commits,
recovery does not silently take the seat back.

## MicroRaft work

### Version 1: no change needed

`ADD_OR_PROMOTE_TO_FOLLOWER` followed by `REMOVE_MEMBER` are existing operations. Each changes the
voter set by one server, so each is safe on its own and the intermediate four-voter state has
overlapping majorities. The work for version 1 is entirely in the S5 planner.

To verify before enabling: how a removed node that returns behaves. It is no longer in the member
list, so the leader stops contacting it, but it may still hold its old role and try pre-vote
rounds. Pre-vote and leader stickiness should keep it from disrupting the group; this needs a test.

### Version 2 (implemented): `DEMOTE_TO_LEARNER`

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

This is a single-server membership change, so it follows the same safety argument as add and remove.
Keeping a demoted node as a long-lived non-voting member extends the thesis's transient non-voting
member, but it does not affect quorums.

### Not implemented in MicroRaft

- **Atomic promote-and-retire is deliberately not provided.** Replacing `{A,B,X}` with `{A,B,L}` in
  one entry changes two voters at once, and the old and new majorities (`{A,X}` and `{B,L}`) can be
  disjoint. Single-server membership change is only safe one server per entry; a swap would need
  joint consensus, which MicroRaft does not have. The two-step sequence is the safe form.
- **Vote-request guard.** Vote and pre-vote requests from a known non-voter are not rejected. A
  receiver with a stale config could otherwise refuse a legitimately promoted node. A demoted node
  does not campaign because it is a learner.
- **Further demotion tests:** restart between the promotion and the demotion, snapshot install on a
  node being demoted, and the SQLite store persisting `voting=false`.
- **Liveness edge case (deliberate demotion of a healthy node only):** if a leader appends a demotion
  that only the demoted follower receives and then dies, that follower is already a learner and
  refuses votes, so the remaining voters may not be able to elect a leader until the old leader
  returns. Automatic replacement is not affected, because the target is offline and never applies the
  entry.
- **Rolling upgrade:** an older node that receives the new mode value is not handled (version 2 only).
- **Learner readiness:** MicroRaft exposes no "caught up" flag. The planner must compare the learner's
  match index with the leader's commit index from `RaftNodeReport`.

## S5 planner work

Sequencing is owned by the S5 planner, not MicroRaft. The planner must be written as an idempotent
"converge to three voters" loop, not as a remembered two-step transaction, because leadership can
change between the steps.

- **Order:** promote the learner first, wait for the commit, then retire the offline voter. Promote
  first keeps the learner holding the entries before the offline voter's seat is released. Retiring
  first would leave two voters with no failure tolerance.
- **Window:** while four voters exist with one offline, an election needs A, B, and L all reachable. A
  leader crash in this window blocks elections until a voter returns. Committing entries needs fewer
  acks (see `RaftState.logReplicationQuorumSize`), but the membership entries themselves are committed
  with the full quorum.
- **Resume after failover:** a new leader that sees four voters, one unreachable, finishes the
  retirement.
- **Concurrency:** pass the current committed members index on each call. On
  `MismatchingRaftGroupMembersCommitIndexException`, re-read and re-plan. `CannotReplicateException`
  on a fresh leader is retryable.
- **Leader-only policy:** only the leader evaluates the deadline, so two nodes never propose competing
  changes. A local timeout alone must never alter voter membership outside consensus.
- **Offline voter is the leader:** wait for a new leader, then evaluate. A leader cannot be removed or
  demoted by itself.
- **Version 2 cleanup:** remove demoted nodes that stay offline past the second deadline, so dead
  learners do not hold the learner slots.

S5 should add planner and runtime tests for the deadline, candidate readiness, committed transition
order, recovery before and after the deadline, leader change between steps, and blocked replacement.

## Unverified assumptions and open checks

These were assumed or read from code while writing this design and have not been confirmed by a test
or by running the system. Each should be checked before the behavior is enabled.

### MicroRaft behavior

- **Returning removed node (version 1).** A removed node that comes back may still hold its old role
  and try pre-vote rounds. Expected: pre-vote and leader stickiness keep it from disrupting the
  group, and the leader never contacts it. Not tested. Also check how long it keeps retrying and
  what it logs.
- **Removal of an offline voter.** `REMOVE_MEMBER` with an unreachable target should commit with the
  remaining voters. The existing tests were not checked for this exact case (target down, four
  voters, learner just promoted).
- **Full sequence under failover.** Promote, then retire, with a leader change between the two
  entries, has not been run end to end. Only each single-server step is covered.
- **Four-voter window.** The claims that an election needs A, B, and L, and that commits need fewer
  acks than elections (`RaftState.logReplicationQuorumSize`), come from reading the code. They are not
  tested for a four-voter group with one member down.
- **Demotion gaps (version 2).** Restart between promotion and demotion, snapshot install on a node
  being demoted, SQLite store persisting `voting=false`, and rolling upgrade with the new mode value.
- **Vote-request guard.** Whether a demoted node with a stale config could ever disturb elections
  was reasoned about, not tested.

### Interface to S5

- **Learner progress.** The planner needs the learner's match index and the leader's commit index.
  Whether `RaftNodeReport` exposes enough (per-follower match indexes, heartbeat timestamps) was
  not confirmed. If it does not, MicroRaft needs a small reporting addition.
- **Unreachability detection.** How S5 decides a voter is "continuously unreachable" for the
  deadline was not designed. MicroRaft heartbeat timestamps from the leader are the likely source;
  this needs checking, including what happens across leader changes where the new leader has no
  history.
- **Current S5 behavior.** The statement that explicit node removal triggers replacement through the
  existing path was taken from earlier notes, not verified in the S5 code, which is not in this
  repository.
- **Referenced qualification note.** `NativeClusterVoterReplacement-2026-10-10.md` was not read while
  updating this document, and the link may not resolve from this repository.

### Background

- **Raft thesis references.** Section numbers and the single-server-change safety argument were cited
  from memory and should be re-read against the thesis before this is used as a safety argument in
  review.
- **Learner slot cap.** The `MAX_LEARNER_COUNT = 2` interaction with removal and with the cleanup
  deadline was read from code, and only the refusal case is tested.

## Scope and non-goals

This proposal replaces only the policy for sustained voter unreachability when a safe learner
replacement is available. It does not convert every brief packet loss into a reconfiguration, and it
does not claim a node is permanently destroyed. It does not change blob replication or placement; those
have their own repair and safety rules. It does not permit a group that lacks quorum to reconfigure
itself. Quorum-lost recovery remains a separate operator-controlled procedure.

## Decision

Adopt a configurable offline deadline, initially 60 seconds, after which a reachable and caught-up
learner is promoted and the unreachable voter is retired through two committed Raft membership
changes.

Ship version 1 first: promote, then `REMOVE_MEMBER`. It fixes the at-risk-forever condition using
operations MicroRaft already has, and needs only the S5 planner and its tests. Adopt version 2 later
if returning nodes should rejoin without operator action: replace the removal with
`DEMOTE_TO_LEARNER` (implemented on the fork) and add the cleanup deadline.

Until the planner lands, S5's current behavior remains in effect: ordinary unreachability does not
change voter membership; explicit node removal can trigger replacement using the existing
remove-based path.
