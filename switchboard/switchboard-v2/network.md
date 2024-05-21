# Network

Switchboard was architected to be permissionless and allow users to create and manage their own Switchboard network. Each chain can support many oracle queues, which can have varrying levels of security and trust assumptions.

An oracle queue is an independent realm of oracles, responsible for allocating oracle resources for update requests from data feeds, randomness, or buffer relayers. Oracle queue's act as an aggregator for on-chain consumers looking to publish data on-chain by specifying an upfront reward a requester is required to pay when a new update is requested by an oracle. Oracles act as an off-chain compute resource that can be utilized by on-chain programs needing a decentralized way to source data.

Each oracle queue is independent and maintain their own configurations, which dictates its degree of security. Queue's can require update requesters to be pre-approved to use a queues resources or allow any requester access to a queue. Queue's also specify a minimum stake oracles must maintain in their escrow wallet before joining a queue, which acts as a deposit to incentivize honest oracle behavior.

### Oracle Queue[​](http://localhost:3000/about/network#oracle-queue) <a href="#oracle-queue" id="oracle-queue"></a>

When creating a queue, an OracleQueueBuffer account must also be initialized with a size of 8 Bytes + (32 Bytes × `queue.maxSize`), where `queue.maxSize` is the maximum number of oracles the queue can support. The OracleQueueBuffer account `queue.dataBuffer` stores a list of oracle public keys in a round robin fashion, using `queue.currIdx` to track its position on the queue for allocating resource update request. Once a buffer is full, oracles must be removed before new oracles can join the network. An oracle can be assigned to many update request simultaneously but must continuously heartbeat on-chain to signal readiness.

An oracle with **PermitOracleHeartbeat** permissions _MUST_ periodically heartbeat on the queue to signal readiness, which adds the oracle to the queue and allows it to be assigned resource update requests. Oracle positions are periodically swapped in the OracleQueueBuffer account to mitigate oracles being assigned the same update requests on each iteration of the queue.

The queue uses `queue.gcIdx` to track its garbage collection index. When an oracle heartbeats on-chain, it passes the oracle account at index `queue.gcIdx`. If the oracle account has failed to heartbeat before `queue.oracleTimeout`, it is removed from the queue until its next successful heartbeat and will no longer be assigned resource update requests.

### Access Control[​](http://localhost:3000/about/network#access-control)

Oracle queue resources, such as oracles, aggregators, VRF accounts, or buffer relayer accounts, _MUST_ have an associated PermissionAccount initialized before interacting with a queue. Permissions are granted by `queue.authority`, which could be a DAO controlled account to allow network participants to vote on new entrants.

Oracles _MUST_ have **PermitOracleHeartbeat** permissions before heartbeating on a queue. This is to prevent a malicious actor from spinning up a plethora of oracles until it obtains the super majority, at which point it could misreport data feed results and cause honest oracles to be slashed.

See the table below for the minimum required permissions for a resource based on the queues settings:

### Crank[​](http://localhost:3000/about/network#crank) <a href="#crank" id="crank"></a>

A queue can choose to create one or many cranks. A crank is a scheduling mechanism that allows data feeds to request periodic updates. A crank can be turned by anyone, and if successful, the crank turner will be rewarded for jump starting the system.

A data feed is only permitted to join a crank if it has sufficient permissions (as detailed above) and the crank has available capacity. Data feeds on a crank are ordered by their next available update time with some level of jitter to mitigate oracles being assigned to the same update request upon each iteration of the queue, which makes them susceptible to a malicious oracle. The maximum update interval for a feed on a crank is based on its `aggregator.minUpdateDelaySeconds` and will be scheduled for an update attempt at this interval.

### Economic Security[​](http://localhost:3000/about/network#economic-security) <a href="#economic-security" id="economic-security"></a>

An oracle queue uses economic incentives to entice oracles to act honestly, which dictate a queue's security model.

#### Stake[​](http://localhost:3000/about/network#stake) <a href="#stake" id="stake"></a>

The queue's `queue.minStake` is the raw token amount in the token mints base unit (_Ex: lamports or satoshis_) required by an oracle to heartbeat on a queue. If an oracle's staking wallet falls below the minStake requirement, it is removed from the queue.

DeFi protocols with a significant Total Value Locked (TVL) should require oracles with a higher minimum stake to fulfill their update request. Oracles with a higher degree of _skin-in-the-game_ have a greater incentive to respond honestly.

#### Reward[​](http://localhost:3000/about/network#reward) <a href="#reward" id="reward"></a>

The queue's specified `queue.reward` is the number of tokens an oracle or crank turner receives for successfully completing an on-chain action. For a crank turner this is turning the crank and invoking a data feed update. For an oracle this is responding to an update request within the reliable margin from the accepted result.

Queues should reward oracles enough such that the economic incentive over the lifecycle of the feed exceeds the opportunity cost to attack a protocol consuming the feed.

#### Slashing[​](http://localhost:3000/about/network#slashing) <a href="#slashing" id="slashing"></a>

A queue may set `queue.slashingEnabled` to true in order to dissuade oracles from responding to update request outside a set margin of error.

A queue's `queue.varianceToleranceMultiplier` determines how many standard deviations an oracle must respond within before being slashed and forfeiting a portion of their stake. \[Defaults to 2 std deviations]

DeFi protocols with a significant TVL should require their feeds to be on a queue with slashing enabled.

### Governance[​](http://localhost:3000/about/network#governance) <a href="#governance" id="governance"></a>

An oracle queue can be governed by its network participants to control the various queue configuration parameters, such as:

* `queue.minStake` - require a higher up-front cost for oracles to entice honest behavior
* `queue.reward` - control the oracle reward payout for successfully fulfilling update request
* `queue.slashingEnabled` - to disincentivize malicious oracle behavior
* Permit new oracles to join the network
