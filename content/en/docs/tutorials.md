+++
title =  "Tutorials"


date = 2019-01-28
lastmod = 2025-05-28

draft = false  # Is this a draft? true/false
toc = true  # Show table of contents? true/false
type = "docs"  # Do not modify.

# Add menu entry to sidebar.
linktitle = "Tutorials"
[menu.docs]
  parent = "getting-started"
  weight = 2
+++

This section provides guidance to help you quickly get started with Volcano, from deploying a basic Volcano Job/Deployment, to integrating with Volcano Queues
## Prerequisites
A Kubernetes cluster with Volcano components need to be installed successfully. If you haven't installed Volcano yet, please refer to [Installation](https://volcano.sh/en/docs/installation/).

## Quick Start: Deploy a Volcano Job
This quick start guide will walk you through deploying a simple Volcano Job. By default, Volcano Jobs use the default queue if no specific queue is provided.

### Step 1: Create a Volcano Job
Create a file named vcjob-quickstart.yaml with the following content:
```yaml
# vcjob-quickstart.yaml
apiVersion: batch.volcano.sh/v1alpha1
kind: Job
metadata:
  name: quickstart-job
spec:
  minAvailable: 3
  schedulerName: volcano
  policies:
    - event: PodFailed
      action: RestartJob
  tasks:
    - replicas: 3
      name: completion-task
      policies:
      - event: TaskCompleted
        action: CompleteJob
      template:
        spec:
          containers:
            - command:
              - sh
              - -c
              - 'echo "Job is running and will complete!"; sleep 100; echo "Job done!"'
              image: busybox:latest
              name: busybox-container
              resources:
                requests:
                  cpu: 100m
                limits:
                  cpu: 100m
          restartPolicy: Never
```
This job creates three pods and schedules the pods together as a group. The pod template uses a simple busybox container and sleeps for 100 seconds. 
If the pod is completed, the job will also convert to completed state. 

### Step 2: Monitor the Job and Pod Status
You can observe the progress of your VolcanoJob and its associated Pod.

First, check the VolcanoJob status:
```bash
kubectl get vcjob quickstart-job -oyaml
```
You should see output similar to this:
```yaml
apiVersion: batch.volcano.sh/v1alpha1
kind: Job
# ...
status:
  state:
    phase: Running
  # ...
```

Next, check the status of the Pod created by the Volcano Job:

```bash
kubectl get pod -l volcano.sh/job-name=quickstart-job
```
Initially, the pod will be in a Running state. After about 100 seconds, the busybox container will exit, and the pods' status will change to Completed.
```
NAME                               READY   STATUS      RESTARTS   AGE
quickstart-job-completion-task-0   0/1     Completed   0          3m59s
quickstart-job-completion-task-1   0/1     Completed   0          3m59s
quickstart-job-completion-task-2   0/1     Completed   0          3m59s
```

Once the Pod completes, the `TaskCompleted` policy within the VolcanoJob will trigger the `CompleteJob` action. 
This will transition the VolcanoJob's phase to Completed:
```yaml
apiVersion: batch.volcano.sh/v1alpha1
kind: Job
metadata:
  # ... (metadata details) ...
  name: quickstart-job
  namespace: default
  # ...
status:
  #...
  minAvailable: 3
  runningDuration: 1m49s
  state:
    lastTransitionTime: "2025-05-28T08:41:11Z"
    phase: Completed
  version: 3

```

## Deploy Standard Kubernetes Workloads (Deployment, StatefulSet, etc.)
Volcano seamlessly integrates with standard Kubernetes workloads like Deployment, StatefulSet, and others, extending their scheduling capabilities. 
This means you can leverage Volcano's advanced features, such as gang scheduling. With gang scheduling, you can specify a minimum number of pods that must be schedulable as a group before any pods from that workload are launched.

### Step 1: Create a Deployment with group-min-member Annotation
Let's create a Deployment that expects 3 replicas but requires at least 2 pods to be schedulable as a group by Volcano.
```yaml
# deployment-with-minmember.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-deployment
  annotations:
    scheduling.volcano.sh/group-min-member: "2"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      schedulerName: volcano
      containers:
        - name: my-container
          image: busybox
          command: ["sh", "-c", "echo 'Hello Volcano from Deployment'; sleep 3600"]
          resources:
            requests:
              cpu: 100m
            limits:
              cpu: 100m
```

Check the PodGroup status:
```bash
# First, find the PodGroup name (usually starts with 'podgroup-')
kubectl get pg
# Then inspect it:
kubectl get pg <podgroup-name> -oyaml
```
You will observe that Volcano's scheduler ensures that at least minMember (2 in this example) pods can be scheduled together before it allows any pods from this deployment to be launched. 
If there aren't enough resources for these pods, these pods will keep pending.

## Deploy Workloads with Custom Queues
### Step 1: Create a Custom Queue
Let's create a queue named "development-queue" with a specific CPU capability. Jobs assigned to this queue will contend for resources defined by its capability.

Create a file named queue.yaml:
```yaml
# queue.yaml
apiVersion: scheduling.volcano.sh/v1beta1
kind: Queue
metadata:
  name: development-queue
spec:
  weight: 1 # Relative weight for scheduling priority among queues
  reclaimable: false # If true, jobs in other queues can reclaim resources in this queue
  capability:
    cpu: 2
```
Create the queue in your cluster:
```bash
kubectl create -f queue.yaml
```
A new queue will be created:
```bash
kubectl get queue development-queue -oyaml
```
Expected output:
```yaml
apiVersion: scheduling.volcano.sh/v1beta1
kind: Queue
# ...
status:
  state: Open
```

### Step 2: Create a Volcano Job using the Custom Queue
Now, let's create a VolcanoJob that explicitly uses our development-queue.

Create a file named vcjob-with-queue.yaml and apply it:
```yaml
# vcjob-with-queue.yaml
apiVersion: batch.volcano.sh/v1alpha1
kind: Job
metadata:
  name: job-with-custom-queue
spec:
  minAvailable: 1
  schedulerName: volcano
  queue: development-queue # Assign this job to our custom queue
  tasks:
    - replicas: 1
      name: custom-queue-task
      policies:
      - event: TaskCompleted
        action: CompleteJob
      template:
        spec:
          containers:
            - command:
              - sh
              - -c
              - 'echo "Running in custom queue"; sleep 100; echo "Done!"'
              image: busybox:latest
              name: busybox-in-queue
              resources:
                requests:
                  cpu: 1
                limits:
                  cpu: 1
          restartPolicy: Never
```
### Step 3: Check the Status of the Custom Queue
You can monitor the status of your custom queue to see how many resources have been allocated
```bash
kubectl get queue development-queue -oyaml
```
Expected output:
```yaml
apiVersion: scheduling.volcano.sh/v1beta1
kind: Queue
metadata:
  # ...
  name: development-queue
  # ...
spec:
  capability:
    cpu: 2
  parent: root
  reclaimable: false
  weight: 1
status:
  allocated:
    cpu: "1"
    memory: "0"
    pods: "1"
  state: Open

```

---

## Next Steps: Advanced Concepts

Ready to tackle production-grade challenges? Check out our **[Advanced Concepts Tutorial Series](/en/docs/advanced-tutorials/)** to learn how to orchestrate Distributed Training (TensorFlow), Big Data (Spark), and GPU Resource Management.
