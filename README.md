# Bin Packing Scheduler

This document provides instructions on how to install a custom scheduler
in our managed Kubernetes instance.  

## Instructions

1. Edit lines 47-71, in `bp-scheduler.yaml` to set the correct scheduler
   and parameters.  Currently set for sane values for one of the two
   default bin packing schedulers.
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
