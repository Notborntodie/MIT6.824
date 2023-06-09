6.5840 2023 Lecture 6: Raft (2)

Last lecture: election safety
  a single leader per term
  as long as the leader stays up:
    clients only interact with the leader
    clients don't see follower states or logs

Today: replicating, persisting, and compacting log

*** topic: the Raft log (Lab 2B)

Challenge: log divergence
  a leader crashes before sending AppendEntries to all
    S1: 3
    S2: 3 3
    S3: 3 3
  (the 3s are the term number in the log entry)
  worse: logs might have different commands in same entry!
    after a series of leader crashes, e.g.
        10 11 12 13  <- log entry #
    S1:  3
    S2:  3  3  4
    S3:  3  3  5

  How could this happen?
    S2 is leader in term 3
      appends 10 to S1, S2, and S3
      appends 11 to S2 and S3 (S1 crashed)
    S3 crashes, reboots quickly, and leader in term 4
      appends 12 to its log, and crashes.
    S3 becomes leader in term 5 (with help of S1)
      appends a different entry for 12 to its log
    
what do we want to ensure?
  if any server executes a given command in a log entry,
    then no server executes something else for that log entry
  (Figure 3's State Machine Safety)
  why? if the servers disagree on the operations, then a
    change of leader might change the client-visible state,
    which violates our goal of mimicing a single server.
  example:
    S1: put(k1,1) | put(k1,2) 
    S2: put(k1,1) | put(k2,3) 
    can't allow both to execute their 2nd log entries!

Raft forces agreement by having followers adopt new leader's log
  example:
  S3 is chosen as new leader for term 6
  S3 wants to append a new log entry at index 13
  S3 sends an AppendEntries RPC to all
     prevLogIndex=12
     prevLogTerm=5
  S2 replies false (AppendEntries step 2)
  S3 decrements nextIndex[S2] to 12
  S3 sends AppendEntries w/ entries 12+13, prevLogIndex=11, prevLogTerm=3
  S2 deletes its entry 12 (AppendEntries step 3)
    and appends new entries 12+13
  similar story for S1, but S3 has to back up one farther

the result of roll-back:
  each live follower deletes tail of log that differs from leader
  then each live follower accepts leader's entries after that point
  now followers' logs are identical to leader's log

Q: why was it OK to forget about S2's index=12 term=4 entry?

could new leader roll back *committed* entries from end of previous term?
  i.e. could a committed entry be missing from the new leader's log?
  this would be a disaster -- old leader might have already said "yes" to a client
  so: Raft needs to ensure elected leader has all committed log entries

why not elect the server with the longest log as leader?
  in the hope of guaranteeing that a committed entry is never rolled back
  example:
    S1: 5 6 7
    S2: 5 8
    S3: 5 8
  first, could this scenario happen? how?
    S1 leader in term 6; crash+reboot; leader in term 7; crash and stay down
      both times it crashed after only appending to its own log
    Q: after S1 crashes in term 7, why won't S2/S3 choose 6 as next term?
    next term will be 8, since at least one of S2/S3 learned of 7 while voting
    S2 leader in term 8, only S2+S3 alive, then crash
  all peers reboot
  who should be next leader?
    S1 has longest log, but entry 8 could have committed !!!
    so new leader can only be one of S2 or S3
    so the rule cannot be simply "longest log"

end of 5.4.1 explains the "election restriction"
  RequestVote handler only votes for candidate who is "at least as up to date":
    candidate has higher term in last log entry, or
    candidate has same last term and same length or longer log
  so:
    S2 and S3 won't vote for S1
    S2 and S3 will vote for each other
  so only S2 or S3 can be leader, will force S1 to discard 6,7
    ok since 6,7 not on majority -> not committed -> reply never sent to clients
    -> clients will resend the discarded commands

the point:
  "at least as up to date" rule ensures new leader's log contains
    all potentially committed entries
  so new leader won't roll back any committed operation

The Question (from last lecture)
  figure 7, top server is dead; which can be elected?

who could become leader in figure 7? (with top server dead)
  need 4 votes to become leader
  a: yes -- a, b, e, f
  b: no -- b, f
    e has same last term, but its log is longer
  c: yes -- a, b, c, e, f
  d: yes -- a, b, c, d, e, f
  e: no -- b, f
  f: no -- f

why won't d prevent a from becoming leader?
  after all, d's log has higher term than a's log
  a does not need d's vote in order to get a majority
  a does not even need to wait for d's vote

why is Figure 7 analysis important?
  choice of leader determines which entries are preserved vs discarded
  critical: if service responded positively to a client,
    it is promising not to forget!
  must be conservative: if client *could* have seen a "yes",
    leader change *must* preserve that log entry.
    Election Restriction does this via majority intersection.
  why OK to discard e's last 4,4?
  why OK to (perhaps) preserve c's last 6?
    could client have seen a "yes" for them?

"a committed operation" has two meanings in 6.5840:
  1) the op cannot be lost, even due to (allowable) failures.
     in Raft: when a majority of servers persist it in their logs.
     this is the "commit point" (though see Figure 8).
  2) the system knows the op is committed.
     in Raft: leader saw a majority.

again:
  we cannot reply "yes" to client before commit.
  we cannot forget an operation that may have been committed.
      
how to roll back quickly
  the Figure 2 design backs up one entry per RPC -- slow!
  lab tester may require faster roll-back
  paper outlines a scheme towards end of Section 5.3
    no details; here's my guess; better schemes are possible
      Case 1      Case 2       Case 3
  S1: 4 5 5       4 4 4        4
  S2: 4 6 6 6 or  4 6 6 6  or  4 6 6 6
  S2 is leader for term 6, S1 comes back to life, S2 sends AE for last 6
    AE has prevLogTerm=6
  rejection from S1 includes:
    XTerm:  term in the conflicting entry (if any)
    XIndex: index of first entry with that term (if any)
    XLen:   log length
  Case 1 (leader doesn't have XTerm):
    nextIndex = XIndex
  Case 2 (leader has XTerm):
    nextIndex = leader's last entry for XTerm
  Case 3 (follower's log is too short):
    nextIndex = XLen

*** topic: persistence (Lab 2C)

what would we like to happen after a server crashes?
  Raft can continue with one missing server
    but failed server must be repaired soon to avoid dipping below a majority
  two repair strategies:
  * replace with a fresh (empty) server
    requires transfer of entire log (or snapshot) to new server (slow)
    we must support this, in case failure is permanent
  * or reboot crashed server, re-join with state intact, catch up
    requires state that persists across crashes
    we must support this, for simultaneous power failure
    let's talk about the second strategy -- persistence

if a server crashes and restarts, what must Raft remember?
  Figure 2 lists "persistent state":
    log[], currentTerm, votedFor
  a Raft server can only re-join after restart if these are intact
  thus it must save them to non-volatile storage
    non-volatile = disk, SSD, battery-backed RAM, &c
    save after each point in code that changes non-volatile state
    or before sending any RPC or RPC reply
  why log[]?
    if a server was in leader's majority for committing an entry,
      must remember entry despite reboot, so next leader's
      vote majority includes the entry, so Election Restriction ensures
      new leader also has the entry.
  why votedFor?
    to prevent a client from voting for one candidate, then reboot,
      then vote for a different candidate in the same term
    could lead to two leaders for the same term
  why currentTerm?
    avoid following a superseded leader.
    avoid voting in a superseded election.

some Raft state is volatile
  commitIndex, lastApplied, next/matchIndex[]
  why is it OK not to save these?

persistence is often the bottleneck for performance
  a hard disk write takes 10 ms, SSD write takes 0.1 ms
  so persistence limits us to 100 to 10,000 ops/second
  (the other potential bottleneck is RPC, which takes << 1 ms on a LAN)
  lots of tricks to cope with slowness of persistence:
    batch many new log entries per disk write
    persist to battery-backed RAM, not disk
    be lazy and risk loss of last few committed updates

how does the service (e.g. k/v server) recover its state after a crash+reboot?
  easy approach: start with empty state, re-play Raft's entire persisted log
    lastApplied is volatile and starts at zero, so you may need no extra code!
    this is what Figure 2 does
  but re-play will be too slow for a long-lived system
  faster: use Raft snapshot and replay just the tail of the log

*** topic: log compaction and Snapshots (Lab 2D)

problem:
  log will get to be huge -- much larger than state-machine state!
  will take a long time to re-play on reboot or send to a new server

luckily:
  a server doesn't need *both* the complete log *and* the service state
    the executed part of the log is captured in the state
    clients only see the state, not the log
  service state usually much smaller, so let's keep just that

what log entries *can't* a server discard?
  committed but not yet executed
  not yet known if committed

solution: service periodically creates persistent "snapshot"
  [diagram: service state, snapshot on disk, raft log (in mem, on disk)]
  copy of service state as of execution of a specific log entry.
    e.g. k/v table.
  service hands snapshot to Raft, with last included log index.
  Raft persists its state and the snapshot.
  Raft then discards log before snapshot index.
  every server snapshots (not just the leader).

what happens on crash+restart?
  service reads snapshot from disk
  Raft reads persisted log from disk
  Raft sets lastApplied to snapshot's last included index
    to avoid re-applying already-applied log entries

problem: what if follower's log ends before leader's log starts?
  because follower was offline and leader discarded early part of log
  nextIndex[i] will back up to start of leader's log
  so leader can't repair that follower with AppendEntries RPCs
  thus the InstallSnapshot RPC

philosophical note:
  state is often equivalent to operation history
  one or the other may be better to store or communicate
  we'll see examples of this duality later in the course

practical notes:
  Raft's snapshot scheme is reasonable if the state is small
  for a big DB, e.g. if replicating gigabytes of data, not so good
    slow to create and write entire DB to disk
  perhaps service data should live on disk in a B-Tree
    no need to explicitly snapshot, since on disk already
  dealing with lagging replicas is hard, though
    leader should save the log for a while
    or remember which parts of state have been updated

*** read-only operations (end of Section 8)

Q: does the Raft leader have to commit read-only operations in
   the log before replying? e.g. Get(key)?

that is, could the leader respond immediately to a Get() using
  the current content of its key/value table?

A: no, not with the scheme in Figure 2 or in the labs.
   suppose S1 thinks it is the leader, and receives a Get(k).
   it might have recently lost an election, but not realize,
   due to lost network packets.
   the new leader, say S2, might have processed Put()s for the key,
   so that the value in S1's key/value table is stale.
   serving stale data is not linearizable; it's split-brain.

so: Figure 2 requires Get()s to be committed into the log.
    if the leader is able to commit a Get(), then (at that point
    in the log) it is still the leader. in the case of S1
    above, which unknowingly lost leadership, it won't be
    able to get the majority of positive AppendEntries replies
    required to commit the Get(), so it won't reply to the client.

but: many applications are read-heavy. committing Get()s
  takes time. is there any way to avoid commit
  for read-only operations? this is a huge consideration in
  practical systems.

idea: leases
  modify the Raft protocol as follows
  define a lease period, e.g. 5 seconds
  after each time the leader gets an AppendEntries majority,
    it is entitled to respond to read-only requests for
    a lease period without adding read-only requests
    to the log, i.e. without sending AppendEntries.
  a new leader cannot execute Put()s until previous lease period
    has expired
  so followers keep track of the last time they responded
    to an AppendEntries, and tell the new leader (in the
    RequestVote reply).
  result: faster read-only operations, still linearizable.

note: for the Labs, you should commit Get()s into the log;
      don't implement leases.

in practice, people are often (but not always) willing to live with stale
  data in return for higher performance

----