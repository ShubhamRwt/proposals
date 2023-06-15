# Kafka Roller 2.0

## Current situation

The Kafka Roller is a Cluster Operator component that's responsible for rolling (i.e. controlled restarts and dynamic reconfigurations) Kafka pods when needed (e.g. when a certificate is renewed).

### Current Kafka Roller Logic

- Take a list of Kafka pods and reorder them based on the readiness.
- Roll unready pods first without considering why it could be unready.
- Before restarting a Kafka pod, consider if it is a controller, or it has an impact on the availability.
- If a Kafka pod does not become ready after restarting within the timeout, then force restart it without considering
  why it's taking a long time.

### Known Issues

This logic ends up with the following known issues in the Kafka Roller:

- It doesn’t worry about partition leadership
    - Should let broker resume preferred leadership after rolling.
- Its availability
  check [KafkaAvailability](https://github.com/strimzi/strimzi-Kafka-operator/blob/bf4fa3f68cd83685bf56229c6bb98eccefabea72/cluster-operator/src/main/java/io/strimzi/operator/cluster/operator/resource/KafkaAvailability.java)
  is too resource-intensive
    - All topic descriptions in memory at once.
- It’s difficult to reason about.
- For KRaft we need logic for `process.role=controller` and `process.roles=broker,controller`.
- Large clusters: duration of rolling restarts becomes a problem.

## Motivation

Refactoring the Kafka Roller is a change that needs to be well planned and executed since the KafkaRoller is a crucial
and complex component of the Cluster Operator.

That is why this proposal aims for:

- Proposing the desired state of the Kafka Roller.
- Proposing a safe implementation path that focuses on incremental small changes which lead to achieving such desired
  state. 

[WIP] Need to understand how we are going to do things incrementally?

## Proposal

### Enhancement Areas

The proposal focuses on the following areas of enhancements:
- Identifying broker's role using the Admin API (broker | controller | broker, controller)
- New algorithm to perform parallel rolling based on rack awaress/partitions in common
- Safety criteries:
  - Availibility impact
  - Log recovery
  - CC rebalancing state
- Implementing post-restart delay between restarts so that it gives time for JIT to reach a steady state and reduce impact on clients.
- Publishing actions taken on the broker and current state of it as kube events. 

### The new algorithm from high level

1. Create ServerContext for each broker:

  ```
  ServerContext {
    state: ServerState # current state of broker based on pod status and broker state metric
    reason: String # whether this broker needs to be restarted
    numRestarts: int # incremented each time pod has been restarted
    lastTransitionTime: long # System.currentTimeMillis of last observed transitionåå
  }
  ```

  State options:
  - RESTARTING (pod is about to be restarted or broker state > 3)
  - RESTARTED (after successful `kubelet delete pod`)
  - STARTING (pod liveness check returning 200)
  - RECOVERING (broker state == 2)
  - SERVING  (broker state == 3 )
  - LEADING_ALL_PREFERRED (broker state == 3 && leading all preferred replicas)

2. Filter brokers that need to be rolled based on the predicate
3. Categorise brokers into ready |  recovering | !ready && !recovering
4. Restart brokers in !ready && !recovering catogory one by one:
    - Picked at random
    - Wait for its state to become SERVING or timeout to continue
5. Loop through each broker in ready category:
    - Partition brokers into groups based on the rack and controller condition
    - Loop through each group and batch brokers
      - Brokers in the same batch should not have any partitions in common
      - Each batch should not have more than one controller
      - Each batch should not be larger than configured batch size
    - Order batch by its size and the batch with the active controller should be the last one
    - Loop through each batch in order:
      - Restart brokers in parallel 
      - Wait for all brokers to have LEADING_ALL_PREFERRED state or timeout to continue to the next batch
      - Once all brokers have LEADING_ALL_PREFERRED state, apply post restart delay. 

## Implementation Milestones

TBD

## Affected/not affected projects

This proposal affects only
the [`strimzi-Kafka-operator` GitHub repository](https://github.com/strimzi/strimzi-Kafka-operator).

## Compatibility

<!--Call out any future or backwards compatibility considerations this proposal has accounted for.-->
TBD


