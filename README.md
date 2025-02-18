# Kubernetes Topology Spread Constraints Example

This document demonstrates how to use topology spread constraints in Kubernetes to control pod placement across different nodes or zones, ensuring high availability and fault tolerance without overly restrictive anti-affinity rules.

## Prerequisites

*   A working Kubernetes cluster with at least one worker node (ideally two or more for better demonstration).
*   `kubectl` command-line tool configured to connect to your cluster.

## Problem Scenario: Rolling Updates and Anti-Affinity

Traditional pod anti-affinity rules can prevent pods with the same label from coexisting on the same node, improving resilience. However, this can create issues during rolling updates.  When a third pod needs to be scheduled during a rolling update, anti-affinity might prevent it from being placed on the existing nodes, forcing you to add a new node or change the update strategy.

Topology spread constraints offer a more flexible solution by allowing you to specify a degree of imbalance that is acceptable, ensuring some level of distribution while allowing pods to be scheduled even when resources are limited.

## Step 1: Creating a Pod with Topology Spread Constraints

1.  **Create a `pod.yaml` file with the following content:**

    This pod definition includes a `topologySpreadConstraint` that aims to distribute pods labeled with `app: bar` across different zones (`topology.kubernetes.io/zone`). `maxSkew: 1` means that the distribution can tolerate a difference of one pod between any two zones. `whenUnsatisfiable: DoNotSchedule` tells the scheduler not to schedule the pod if the constraint can't be satisfied.

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: random-generator
      labels:
        app: bar
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: bar
      containers:
      - image: k8spatterns/random-generator:1.0
        name: random-generator
    ```

2.  **Inspect the nodes and their labels:**

    ```bash
    kubectl get nodes --show-labels
    ```

    Examine the output. Initially, your worker node (in this example, `node01`) *will likely not have* the label `topology.kubernetes.io/zone`.

3.  **Apply the pod definition:**

    ```bash
    kubectl apply -f pod.yaml
    ```

4.  **Check the pod status:**

    ```bash
    kubectl get pod random-generator
    ```

    The pod will likely be stuck in the `Pending` state. This is because the `topologySpreadConstraint` requires the nodes to have the label specified by `topologyKey`, which is `topology.kubernetes.io/zone`, and currently they don't.

## Step 2: Allowing the Pod to Schedule by Labeling the Node

1.  **Label the worker node with the required topology key:**

    This command adds the `topology.kubernetes.io/zone=us-west-1` label to the node `node01`.  The actual value (`us-west-1`) is arbitrary; it just needs to exist for the scheduler to consider the topology spread constraint.

    ```bash
    kubectl label node node01 topology.kubernetes.io/zone=us-west-1
    ```

2.  **Check the pod status again:**

    ```bash
    kubectl get pods -o wide
    ```

    The pod should now transition to the `Running` state. The output will show the pod running on `node01`:

    ```
    NAME               READY   STATUS    RESTARTS   AGE   IP           NODE     NOMINATED NODE   READINESS GATES
    random-generator   1/1     Running   0          2m   10.244.0.5   node01   <none>           <none>
    ```

## Explanation of Topology Spread Constraints

*   **`topologySpreadConstraints`:**  A field within the pod spec that defines rules for distributing pods across different topology domains (e.g., nodes, zones, regions).
*   **`maxSkew`:**  The maximum allowed difference in the number of matching pods between any two topology domains.  A value of 1 allows for a single pod difference.
*   **`topologyKey`:**  The node label key used to define the topology domain.  In the example, we use `topology.kubernetes.io/zone`, which is a common label for representing availability zones.  You can use other labels relevant to your cluster topology.
*   **`whenUnsatisfiable`:** Specifies how to handle the scheduling decision when the constraint cannot be satisfied.  Options include:
    *   `DoNotSchedule`:  The scheduler will not schedule the pod.
    *   `ScheduleAnyway`: The scheduler will ignore the constraint and schedule the pod anyway. This is useful for scenarios where availability is more important than strict distribution.
*   **`labelSelector`:**  Specifies which pods the constraint applies to.  In the example, it applies to pods with the label `app: bar`.

## Use Cases

Topology spread constraints are valuable for:

*   **High Availability:**  Distributing pods across multiple availability zones to ensure that the application remains available even if one zone fails.
*   **Fault Tolerance:**  Reducing the impact of node failures by distributing pods across different nodes.
*   **Rolling Updates:**  Performing rolling updates without being blocked by overly restrictive anti-affinity rules.
*   **Even Resource Utilization:** Distributing pods across nodes to ensure that resources are used more evenly.

## Built-in Cluster-Level Topology Spread Constraints

Kubernetes also has built-in cluster-level topology spread constraints. These allow you to define default distribution policies based on standard Kubernetes labels like hostname and zone, potentially simplifying configuration and providing default levels of high availability.  These may also consider or ignore node affinity and taint policies depending on configuration. Be sure to research those to see if they fit your use case.