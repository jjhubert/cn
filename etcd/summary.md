# etcd
[overview](https://www.processon.com/view/link/61f60023f346fb4e338a05e7)
## etcd component
1. consistency module
    - leader election(based on the raft)
    - log replication(distribute message to the other node)
    - entry state(mark record current state)
        - unstable
        - committed
        - applied
2. log module
    - wal log(record data change)
    - snapshot(archive the data)
3. state machine module
    - kvindex
      - treeindex(B+Tree)
      - memory store
      - improve query efficiency
    - boltdb
      - disk store
4. kvserver
    - authentication
    - rate limit
    - quota
## etcd node role
1. follower
2. candidate
3. leader
4. learner(Not vote. It is not quorum.)

## timeout
1. election timeout
   - from follower to candidate to need the time
   - the time is random
   - between 150ms and 300ms
2. heartbeat timeout
   - a heartbeat communication cycle

## leader election
### role
- follower
- candidate

### communication
```
follower: not receive leader heartbeat
follower: change follower to candidate
candidate: request votes to the other node
the other followers: receive vote request
the other followers: reply ok
candidate: receive more than half votes
candidate: change candidate to leader
```

### election timeout
1. The election timeout is the amount of time a follower waits until becoming a candidate.
2. The election timeout between 150ms and 300ms.
3. All the follower get a random election timeout.
4. After the election timeout the follower becomes a candidate and starts a new election term.
5. Votes for itself
6. Sends out Request vote message to other nodes.
7. If not receive more than half votes, all the followers will reset the election timeout.

## log replication
### condition
- only leader receive request
- node forward to the leader node if node receive request.

### role
- leader
- follower

### communication
leader = l\
follower = f
```
l: write log when from client receive a write request
l: distribute a write request to the others node
f: record log when receive a write request
f: reply ack
l: commit log record after receive node's reponse
l: response to the client
l: distribute commit request to the others node
f: commit log record after receive the commit request
f: reply ack
```