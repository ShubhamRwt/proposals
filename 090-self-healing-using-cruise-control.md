# Self Healing using Cruise Control

This proposal is about integrating the Self-healing feature of Cruise Control into the Strimzi operator.
The Self-healing feature of Cruise Control allows us to heal the anomalies detected in the cluster automatically. 
Anomaly are detected by Cruise Control using the Anomaly detector classes 

## Current situation

During normal operations it common for Kafka clusters to become unbalanced (through partition key skew, uneven partition placement etc.) or for problems to occur with disks or other underlying hardware which makes the cluster unhealthy.  
Currently, if we encounter any such scenario we need to fix these issues manually i.e. if there is some broker failure then we might move the partition replicas form that corrupted broker to some other healthy broker by using the `KafkaRebalance` custom resource in the `remove-broker` mode.
While with small sized cluster it is feasible to fix things manually, when it comes to larger ones it can be very time-consuming or just not feasible to fix all the issue on your own.
Also, it is possible that while doing thing manually, that we might lose data or cause other unforeseen issues with the Kafka Cluster.

## Motivation

With the help of the self-healing feature we can resolve the issues like disk failure, broker failure and other issues automatically in the cluster.
Cruise Control treats these issues as "anomalies", the detection of which is the responsibility of the anomaly detector mechanism.
Currently, Strimzi has the support to enable anomaly detection by using a custom notifier but the missing part is the interaction between the operator and Cruise Control. Also, even if the anomaly will be detected, the rebalancing would be done manually by using the `KafkaRebalance` custom resource
It would be great to have a way to keep fixing the anomalies whenever they are detected automatically

## Proposal

This proposal is about introducing the self-healing feature of Cruise Control into the strimzi operator.
This feature would allow the cluster to be healed automatically if there are some anomalies detected.

### Introduction

#### Anomaly Detection

Anomaly detection in Cruise Control makes use various anomaly detection classes to detect the anomaly.
The `AnomalyDetectorManager` class is then prompted regarding the detected anomalies and is responsible to handle these detected anomalied i.e whether to fix the anomaly or not.
The anomalies detected can be of various types: broker-failure, disk-failure, topic anomalies, metric anomalies and maintenance anomalies.
There are also multiple anomaly detection related [configurations](https://github.com/linkedin/cruise-control/wiki/Configurations#anomalydetector-configurations) like the anomaly detection interval for anomalies, the anomaly notifier class etc. which can be configured to make anomaly detection more efficient

#### Notifiers in Cruise Control

Whenever anomalies are detected, then Cruise Control provides ability to notify the user regarding the detected anomalies using the notifier classes.
The notification sent by these notifiers increases the visibility of the operation that are taken by Cruise Control.
Cruise Control by default uses `NoopNotifier` but this notifier is configurable and custom user-built notifiers can be used by using the `anomaly.notifier.class` property
Cruise Control also provides some [custom notifiers](https://github.com/linkedin/cruise-control/wiki/Configure-notifications) like Slack Notifier, Alerta Notifier etc. There are multiple other[Self-healing notifier](https://github.com/linkedin/cruise-control/wiki/Configurations#selfhealingnotifier-configurations) related configurations you can use to make notfier more efficient as per the use case

#### Self Healing

If Self-healing is enabled in Cruise Control and anomalies are detected, then automatic fix will start for the detected anomalies.
Cruise Control will generate an optimization proposal based on the anomaly that was detected and will try to fix it.
In case the anomaly is fixable, and load completeness is met then the anomaly will move to FIX_STARTED, in case anomaly is fixable then the anomaly would be IGNORED.
Similarly, the anomaly can move to other different states based on the factors like load completeness, goal requirement met etc.

### Integrating self-healing in Kafka resource

To enable the self-healing feature, the user should set the `self.healing.enabled` configuration to `true` while passing configurations regarding Cruise Control in the Kafka custom resource.
As currently, the self-healing feature will be disabled by default.

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

In case the user wants to enable self-healing for some specific anomalies then they can use configurations like `self.healing.broker.failure.enabled` for broker failure, `self.healing.goal.violation.enabled` for goal violation etc.

### Strimzi Notifier and Cruise Control role

`SelfHealingNotifier` is the base class which can be used when Self-healing is enabled by changing the `anomaly.notifier.class` to `com.linkedin.kafka.cruisecontrol.detector.notifier.SelfHealingNotifier`. 
The `SelfHealingNotifier` contains the logic of Self-healing and overrides multiple methods from the `AnomalyNotifier` interface giving them the implementation on what to do if certain anomalies are detected. Some of those methods are:`onGoalViolation()`
, `onBrokerFailure()`, `onDisk Failure`, `alert()` etc. The other custom notifier offered by Cruise control are also based upon the `SelfHealingNotifier`

Since we need some way to monitor the operations that are happening in Cruise Control though the operator, we will be using `StrimziNotifier`, a new `custom` notifier
To implement it, we will be extending the `SelfHealingNotifier` class and overriding the methods that we require.

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

The `StrimziNotifier` will override the `alert()` method of the base `SelfHealingNotifier` class and instead of publishing the logs, it will publish Kubernetes events.

The event published would look like this: 

```yaml
- action: DetectedAnomalyGettingFixed
  apiVersion: v1
  eventTime: "2024-12-19T08:06:21.314140Z"
  firstTimestamp: null
  involvedObject:
    kind: Pod
    name: <cruise-control-pod-name>
    namespace: myproject
  kind: Event
  lastTimestamp: null
  message: Fixing the anomaly
  metadata:
    creationTimestamp: "2024-12-19T08:06:21Z"
    generateName: cruise-control-event
    name: my-cluster-34c41626-0add-47a3-9b79-d6ee29d5d81f  # my-cluster-<Anomaly ID>
    namespace: myproject
    resourceVersion: "101070"
    uid: 80892c7d-e3a1-44ad-84dc-2d88e15a7ff0
  reason: Anomaly was detected in the cluster GOAL_VIOLATION # Detected Anomaly type
  reportingComponent: cruise-control
  reportingInstance: cruise-control
  source: {}
  type: Normal

```

The `StrimziNotifier` class is going to reside in a separate module in the operator repository called the `strimzi-notifier` so that we can later build a jar w.r.t it and then include that jar in the cruise-control image. 
This is required since we need to run the notifier as part of Cruise Control.
We would also require to add some fabric8 client dependencies to the cruise control image since we are going to use them for creating the anomaly custom resource. <TBD - replaced by fat jar concept>

### Feature Gate

The new feature gate will be called `UseSelfHealing`.
It will be introduced in an alpha state and will be disabled by default.
At this point, there is no timeline for graduation of this feature gate to beta or GA phase since it depends on things.
The schedule will be updated later based on what the community thinks about this feature.

### Execution Flow

1. The use should first enable the `UseSelfHealing` feature in order to use the self-healing configurations
   * If the feature gate is disabled, self properties will be forbidden to use 
2. After that usr should enable Self-healing in Cruise Control configuration of the Kafka custom resource by setting the `self.healing.enabled` field as `true` and also overriding the `anomaly.notifier.class` to use the new `StrimziNotifier` class. 
   * Once the user has enabled it, now the cluster is ready to self-heal.
   * Cruise control will now keep a look on the cluster regarding if there is any anomaly lurking or not. 
   * By default, the interval at which the detector will run to detect the anomalies is 5 minutes. You can change it by using `anomaly.detection.interval.ms` property.
3. If there is an anomaly detected, then the `StrimziNotifier` will publish a kubernetes event based on the anomaly detected.
   * The published event will contain information like anomalyID, anomalyType and anomalyProgress etc.
4. Cruise Control will then fix these detected anomalies
   * In case the anomaly is unfixable, then we will get notified about the anomaly but the fix will be ignored

### Pausing the self-healing feature

The user will also get the flexibility to pause the self-healing feature in case they know or plan to do some rolling restart of the cluster or some rebalance they are going to perform which can cause issue with the self-healing itself.
For this we will be having an annotation called `strimzi.io/self-healing-paused`.
We can set this annotation on the Kafka custom resource to pause the self-healing feature. Self-healing is not paused by default so missing annotation would mean that anomalies detected will keep getting healed automatically
When self-healing is paused, it would mean that all the anomalies that are detected after the pause annotation is applied would be ignored and no fix would be done for them.
The anomalies that were already getting fixed would continue with the fix
Here is how you can enable this annotation:
```sh
kubectl annotate kafka my-cluster `strimzi.io/self-healing-paused: "true"`
```
The kafka custom resource will look like this:
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  annotations:
     strimzi.io/self-healing-paused: "true"
  creationTimestamp: "2024-12-17T09:01:10Z"
  generation: 1
  name: my-cluster
  namespace: myproject
  resourceVersion: "109971"
  uid: 71928731-df71-4b9f-ac77-68961ac25a2f

```

## Affected/not affected projects

A new module name `strimzi-notifier` will be added to the operator repository. The `strimzi-notifier` module should build as a jar and then be included in the cruise control image.
<TBD>

## Rejected Alternatives
 #  <TBD>
