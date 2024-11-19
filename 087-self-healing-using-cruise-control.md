# Self Healing using Cruise Control

This proposal is about integrating the [self-healing](https://github.com/linkedin/cruise-control/wiki/Configurations#selfhealingnotifier-configurations) feature of Cruise Control into Strimzi operator.
The self-healing feature of cruise control allows us to detect the anomalies using its anomaly detector mechanism and then based on the anomaly, cruise control can generate an optimization proposal that fixes the anomaly and move the cluster into healthy state again.

## Current situation

When the users are working with Kafka clusters, it is very much possible that the cluster can move to unhealthy state sure to some broker failure, disk failure or some other issue.
Currently, if we encounter any such scenario we need to fix these issues manually i.e. if there is some broker failure then we might move the partition replicas form that corrupted broker to some other healthy broker using the  `kafka-reassign-partitions.sh` tool.
While with small sized cluster it's easier to fix things manually but when it comes to very big cluster it quite hard to do everything manually since it would take a lot of time to fix all of those issues and then getting the cluster in healthy state.
Also, its very much possible while doing thing manually that we might lose some data or maybe mess up the cluster completely.

## Motivation

With the help of the self-healing feature we can resolve the issues like disk failure, broker failure and other issues automatically in the cluster.
Cruise Control treats these issues as `anomalies`. Cruise will be using the anomaly detector mechanism to detect any anomalies.
If the anomalies are present then a response will be passed on to the operator by the self-healing notifier and then if self is healing configuration is enabled, the cluster will be healed automatically by Cruise Control without any user intervention required.

## Proposal

This proposal is about introducing the self-healing feature of Cruise Control into the strimzi operator.
This feature would allow the cluster to be healed automatically if there are some anomalies detected.

Anomalies are basically issues which can arise in the cluster due to various reason like disk failure, broker failure, topic related issues etc.
Cruise control detects these anomalies using the Anomaly detector and then heals the cluster.

### Enabling self-healing in Kafka resource

To enable the self-healing feature, the user should set the `self.healing.enabled` configuration to `true` while passing configurations regarding Cruise Control in the Kafka custom resource.
Self-healing should be disabled by default.

In the following example, the user can decide to enable the self-healing feature of cruise control in the Kafka custom resource:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    # ...
  cruiseControl:
    config:
      # configuration to enable self-healing for all anomalies
      self.healing.enabled: true  # false by default 
```

Once self-healing is enabled, that would mean that it will now detect all the anomalies like disk failure, broker failure, topic anomaly etc. and then heal the cluster automatically without any user interaction.

In case the user wants to enable self-healing for some specific anomalies then they can use configurations like `self.healing.broker.failure.enabled` for broker failure, `self.healing.goal.violation.enabled` for goal violation etc.
They also have the flexibility to configure properties like the rate at which anomalies should be detected, the time before a fix should happen etc. and many [other configuration properties](https://github.com/linkedin/cruise-control/wiki/Configurations#selfhealingnotifier-configurations) just like these.

### Interaction between the Operator and Cruise Control

A two-way interactions should be made between the operator and cruise control such that the user get to turn of self-healing whenever they desire and also so that the operator can receive updates on what anomalies are being fixed or detected.

To establish this two-way communication, we are going to leverage the `Kubernetes custom resource` which can be used by both cruise control and the operator when running on Kubernetes.

#### Strimzi Notifier and Cruise Control role

Cruise Control uses anomaly detector classes to detect the anomalies and then based on those anomalies it provides the ability to alert the users that some anomaly was detected and self-healing is going to fix it.
These alerts are triggered by the `SelfHealingNotifier` class in cruise control.
But just an alert is not good if we want to have the interaction going between the operator and cruise-control, therefore we would need a new `custom` notifier which can be used instead of the inbuilt `SelfHealingNotifier` to set up a way for having an interaction between cruise control and operator

Strimzi Notifier will be a new `custom` notifier which should be used in place of the `SelfHealingNotifier` which is used by cruise control.
To implement it, we will be extending the `SelfHealingNotifier` class and overriding the methods that we need.

The new notifier can be easily used by adding another configuration property in the cruise control section of the Kafka custom resource called `anomaly.notifier.class`.

Here is an example:
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    # ...
  cruiseControl:
    config:
      # configuration to enable self-healing for all anomalies
      self.healing.enabled: true  # false by default 
      anomaly.notifier.class: io.strimzi.kafka.notifier.StrimziNotifier
```

The `StrimziNotifier` class is going to reside in a separate module in the operator repository called the `strimzi-notifier` so that we can later build a jar w.r.t it and then include that jar in the cruise-control image. 
This is required since we need to run the notifier as part of the cruise control image.
We would also require to add some fabric8 client dependencies to the cruise control image since we are going to use them for creating the anomaly custom resource.

With the new `Strimzi notifier` we would be overriding how alerting currently works in CC.
Everytime cruise control detects an anomaly, our new notifier is going to create a `KafkaAnomaly` custom resource which should contain information about the anomaly like anomaly-id, anomaly-type etc.

Example:
```yaml
- apiVersion: kafka.strimzi.io/v1beta2
  kind: Anomaly
  metadata:
    creationTimestamp: "2024-11-12T12:01:58Z"
    generation: 1
    name: my-cluster-anomaly
    namespace: myproject
    resourceVersion: "5704"
    uid: 95cc2621-052c-440e-a553-f14d003669be
  spec:
    anomalyId: a69c2d7a-ff53-40db-8cd0-abc7b448f936
    anomalyType: DISK_FAILURE
    anomalyProgress: FixPending
  status:
#......
```
the `anomalyId` represents the id assigned to the anomaly, or you can also say it is the user task id.
the `anomalyType` represents the type of anomaly that is detected.
the `anomalyProgress` represents the progress of the anomaly.
It can have different values like `FixPending`, `FixComplete` or `FixFailed`.

This Anomaly custom resource is going to be the source of interaction between both operator and cruise control.
So if self-healing is enabled and some anomaly is detected, a Kafka Anomaly resource will be deployed in the custer.
The operator is going to watch the `KafkaAnomaly` resource with the help of Kubernetes watcher.

Since we need to create `KafkaAnomaly` resource through the notifier, we would need to add some `ClusterRole` and `ClusterRoleBindings` w.r.t to the anomaly custom resource which we are going to be created.
These role and role binding yaml's should be part of the operator installation so that the user have them deployed when they are installing the operator.
That would provide them ease to easily enable self-healing whenever they require it.

#### Operator role

Once the `KafkaAnomaly` resource is created then the operator can watch this resource by the help of the Kubernetes watcher.
If an anomaly is detected then the operator work is to update the `.status` field of the anomaly resource with the current progress of the anomaly i.e. whether it is fixed or not

When an anomaly is detected, Cruise assigns a state to the anomaly i.e.DETECTED. In the same way various states are assigned to anomaly as per the action taken on it.
An anomaly can have following associated states: DETECTED, IGNORED, FIX_STARTED, FIX_FAILED_TO_START, CHECK_WITH_DELAY, LOAD_MONITOR_NOT_READY, COMPLETENESS_NOT_READY.
Cruise Control also provides a `/kafkacruisecontrol/state` endpoint which we can poll with a particular `substate` parameter to get the state of the anomaly.

Here is an example request to get the anomaly state using the state endpoint:
```sh
curl -vv -X GET "http://my-cluster-cruise-control:9090/kafkacruisecontrol/state?verbose &json=true&substates=anomaly_detector"
```

This query should return us the state of the anomaly. Here is an example result of the query:
```shell
AnomalyDetectorState: {selfHealingEnabled:[BROKER_FAILURE, DISK_FAILURE, GOAL_VIOLATION, METRIC_ANOMALY, TOPIC_ANOMALY, MAINTENANCE_EVENT], selfHealingDisabled:[], selfHealingEnabledRatio:{BROKER_FAILURE=1.0, DISK_FAILURE=1.0, GOAL_VIOLATION=1.0, METRIC_ANOMALY=1.0, TOPIC_ANOMALY=1.0, MAINTENANCE_EVENT=1.0}, recentGoalViolations:[], recentBrokerFailures:[{anomalyId=37d4c9be-675a-4124-bf35-83aba3410b3c, failedBrokersByTimeMs={3=1731574274942}, detectionDate=2024-11-14T08:51:14Z, statusUpdateDate=2024-11-14T08:51:14Z, status=FIX_STARTED}], recentMetricAnomalies:[], recentDiskFailures:[], recentTopicAnomalies:[], recentMaintenanceEvents:[], metrics:{meanTimeBetweenAnomalies:{GOAL_VIOLATION:0.00 milliseconds, BROKER_FAILURE:22.19 milliseconds, METRIC_ANOMALY:0.00 milliseconds, DISK_FAILURE:0.00 milliseconds, TOPIC_ANOMALY:0.00 milliseconds}, meanTimeToStartFix:0.00 milliseconds, numSelfHealingStarted:0, numSelfHealingFailedToStart:0, ongoingAnomalyDuration=6.91 seconds}, ongoingSelfHealingAnomaly:None, balancednessScore:100.000}
```
 We can then update the `.status` field of the `KafkaAnomaly` resource with the current status of the resource and other details if required.

The operator is also responsible to make sure that once the anomaly is fixed, the resource with respect to that particular anomaly should be removed. 
To know if the anomaly is resolved or not, we again need to poll the `/kafkacruisecontrol/state` endpoint but this time instead of polling the `anomaly_detector` substate we are going to poll the `executor` substate.
The `executor` substate tell us if the action for fixing the anomaly is complete or not.

Here is an example request:
```sh
curl -X GET "http://my-cluster-cruise-control:9090/kafkacruisecontrol/state?verbose &json=true&substates=executor"
```
and here is the result w.r.t to this query:
```sh
ExecutorState: {state: NO_TASK_IN_PROGRESS}
```
 if it says `NO_TASK_IN_PROGRESS`, that means that either the task is finished or there was no task started, if there is some task in progress, then you should get some output like this
```sh
ExecutorState: {state: INTER_BROKER_REPLICA_MOVEMENT_TASK_IN_PROGRESS, pending(7)/in-progress(0)/aborting(0)/finished(0)/total(7) inter-broker partition movements, completed(0)/total(0) bytes in MBs: 0.00%, max/min/avg concurrent inter-broker partition movements per-broker: 5/5/5.00, triggeredSelfHealingTaskId: 37d4c9be-675a-4124-bf35-83aba3410b3c, triggeredTaskReason: Self healing for BROKER_FAILURE: {Fixable broker failures detected: {Broker 3 failed at 2024-11-14T08:51:14Z}}, recentlyRemovedBrokers: [3]}
```
The `triggeredSelfHealingTaskId` mentioned in the output is similar to the `anomalyId` of anomaly that is being fixed. 
To make sure that when the anomaly is fixed the corresponding resource gets deleted the operator will set the `.spec.anomalyProgess` to `FixComplete` once the fix is completed and polling the `Excecutor` state says that there is no task in progress.
`StrimziNotifier` will also have a Kubernetes watcher which will be checking if the `anomalyProgress` field changed or not.
If it changed to `FixComplete` then it means that fix was complete and the resource can then be deleted by the notifier or if it says `FixFailed` then we should log the error regarding the reason why it failed(i.e a broker failure that cannot be fixed or some other issue).

### Execution Flow

- The user would first enable self-healing in cruise control configuration by setting the `self.healing.enabled` field as `true` and also overriding the `anomlay.notifier.class` to use the new `Strimzi` notifier. 
- Once the user has enabled self-healing, now the cluster is ready to use self-healing.
- Cruise control will now keep a look on the cluster regarding if there is any anomaly lurking or not. 
By default, the interval at which the detector will run to detect the anomalies is 5 minutes. You can change it by using `anomaly.detection.interval.ms` property.
- If there is an anomaly detected, then the `StrimziNotifier` will create an anomaly resource based on the anomaly detected.
The resource will contain information like anomalyID, anomalyType and anomalyProgress etc. 
- This `KafkaAnomaly` resource will be watched by both the `operator` and `cruise control.`
- The operator will then poll the `state` endpoint with `anomaly_detector` substate to know what the current state of the anomaly is.
It will also poll the `state` endpoint with  `executor` state to get the anomalyIds which are currently being fixed
- The operator will thenupdate the `anomalyProgress` field of the `KafkaAnomaly` as `FixStarted` if the anomaly fix has been started.
- To make sure that `anomaly` resource is only deleted when that particular anomaly is fixed, we will again poll the `state` endpoint with  `executor` substate to see if the task are now removed from there.
If there is no task in progress then it would mean that the previous have been completed and we will update the `anomalyprogress` accordingly
- The `Notifier` will keep a watch on the `KafkaAnomaly` resource, in case it see that `anomalyProgess` has moved to `FixCompleted` then it will that anomaly resource w.r.t the anomalyID which is fixed.
- In case a failure is detected and it is not fixable .....<TBD>

#### Pausing the self-healing feature

The user will also get the flexibility to pause the self-healing feature in case they know or plan to do some rolling restart of the cluster or some rebalance they are going to perform which can cause issue with the self-healing itself.
For this we will be having an annotation called `strimzi.io/pause-self-healing-feature`.
You can set this annotation on the Kafka custom resource to pause the self-healing feature.
Here is how you can enable this annotation:
```sh
kubectl annotate kafka my-cluster strimzi.io/pause-self-healing-feature: "true"
```
This would pause the self-healing feature for the next set of anomalies but the anomalies which are already being healed at that moment wouldn't be stopped. 

## Affected/not affected projects

This change will affect the `api` module also the way we currently generate the cruise control images. The `strimzi-notifier` module should build as a jar and then be included in the cruise control images.
We would also need to include some of  Kuberenetes dependencies in the cruise control image since we would be creating and watching resource through the Strimzi Notifier.
It will also affect the `installation` of the operator since we will be adding some extra `ClusterRole` and `ClusterRoleBindings` which will need to installed as part of the operator installation.

## Rejected Alternatives
 #  <TBD>
