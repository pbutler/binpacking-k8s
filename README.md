# Bin Packing Scheduler

This document provides instructions on how to install a custom scheduler
in our managed Kubernetes instance.  

## Motivation

The default scheduler built into Kubernetes is called `LeastAllocated` and
aims to assign pods in manner that optimizes availability by spreading
multiple process over as many nodes (and affects a round robin style
scheduling).  This prevents hot spots and over-provisioning.  However, in
many GPU workloads we want to prevent fragmentation and thus maximize
utilization on all nodes.  To this end `kube-scheduler` offers two other
strategies for assigning workloads: `MostAllocated` and
`RequestedToCapacityRatio`.  Both attempt to achieve bin packing and
reduce fragmentation in similar ways by weighting each resource type
(CPUs, memory, and of course GPUs) and then using those weights to
prioritize each packing each resource type.  

## Overview

Because the we use Nebius’s Managed Kubernetes (mk8s) we don’t have
access to the `kube-scheduler` config files we can’t change what the
default scheduler is, however, we can run an alternative scheduler as a
pod that implements bin packing and add the `spec.schedulerName` to the
pod template.  There is two ways to do it: 1) manually add it to each
and every pod or 2) use a mutating webhook to add the same field
specifying our new alternative scheduler to be used for scheduling the
new pods.  We can achieve the second by using
[OPA Gatekeeper](https://github.com/open-policy-agent/gatekeeper)
to act as a mutating webhook.  In the settings below, we only modify
pods in the default namespace but this can be configured to be more
expansive.  It may not be safe to modify all new pods in all namespaces.

## Instructions

1. Edit lines 47-71, in `bp-scheduler.yaml` to set the correct scheduler
   and parameters.  By default this is set to be sane values for one of
   the two default bin packing schedulers.
2. Run:

   ```bash
   kubectl create -f bp-scheduler.yaml
   ```

3. **(Optional)**: Test the new scheduler:
    1. Run `kubectl create -f` on each of
   the following files in the tests directory:
        1. `good-pod.yaml` -- should schedule successfully using the bin packing
        2. `nosched-pod.yaml` -- should  schedule successfully using the default
           scheduler
        3. `bad-pod.yaml` -- this should not schedule at this point as the
            `schedulerName` references a non-existent scheduler.  This
           is just a sanity check that the field is being listened to.
    2. Check the status of each:

        ```bash
        $ kubectl get pods
        NAME               READY   STATUS    RESTARTS   AGE
        badschedule-pod    0/1     Pending   0          1m
        goodschedule-pod   1/1     Running   0          1m
        noschedule-pod     1/1     Running   0          1m
   
        $ kubectl describe pod noschedule-pod | grep Scheduled
        Normal  Scheduled  34s   default-scheduler  Successfully assigned default/noschedule-pod to computeinstance-e03smzn85cvdpr5w93
  
        $ kubectl describe pod goodschedule-pod | grep Scheduled
        Normal  Scheduled  18s   bp-scheduler  Successfully assigned default/goodschedule-pod to computeinstance-e03zb14zbjf9h06hbw
        ```

       At this point the `goodschedule` pod and the `noschedule` pod should be
       running as they are set to use the bin packing scheduler and the
       default scheduler respectively.  The `badschedule` pod won't be
       scheduled because it is configured with a non-existent scheduler (this
       is just a sanity check).

    3. Delete the pods using `kubectl delete -f` each of the test files

4. If you want to have all pods scheduled according to the new
   scheduler continue, otherwise it is sufficient to set
  `spec.schedulerName` to `bp-scheduler`.

5. Install a mutating webhook to change the scheduler:

    ```bash
    kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/v3.21.0/deploy/gatekeeper.yaml
    kubectl create -f mutator.yaml
    ```

   This only modifies pods instantiated in the `default` namespace.  You
   can modify the `mutator.yaml` to expand this behavior.

6. Try the Step 3 tests again.  In this case all three should run and be
   scheduled using `bp-scheduler`.
