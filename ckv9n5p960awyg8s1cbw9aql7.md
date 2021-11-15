## Kubernetes: Zero-downtime deployments

First off let's define what zero-downtime deployments are.  I'll define it as follows

zero downtime deployments = **having no additional errors during a deploy**

Most commonly these are network errors from dropping connections, or from connections being routed to a service that isn't responding yet.



## Why?

We'd like to be able to deploy code at any time without negatively impacting users.  The more frequently we deploy the more important this becomes.


## What we'll need

1. Don't send traffic to pod until it's ready.  (Readiness check)
2. When shutting down a pod stop sending traffic, and give it time to shutdown. (graceful shutdown or preStop lifecycle)
3. Don't shutdown old pods before new pods are ready. (Rolling updates)



## Verify a pod is ready to receive traffic - Readiness checks

#### What is a readiness check?

A readiness check tells terraform where to request in your service in order to check if it's read to receive traffic.

#### How to implement

To tell kubernetes how to check if a service is read to start receiving traffic the most common way is to setup a simple "health" endpoint.  This is an endpoint that just returns successful after all startup for the service is complete and running.

Basic get based readiness check.  The path `healthz` is not a typo, and is the general convention of where this route should be.
```yaml
...
spec:
  containers:
  - name: myservice
    image: ...

    ports:
    - name: myport
      containerPort: 8080
      hostPort: 8080

    readinessProbe:
      httpGet:
        path: /healthz
        port: myport
```

There are several other types of readiness checks you can find  [here](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/). 

**Note: ** If you're using GKE for Kubernetes, when you create a service with external load-balancer it will have a health check of it's own.

**Note: **Liveness checks are very similar and for checking when an application goes down after it's been running for awhile. I highly recommend there use as well, but they don't really impact zero-downtime deployment.

## Let a service finish what it's doing, then shutdown

### preStop lifecycle hook

A preStop lifecycle hook is a command called before sending a terminate signal to a pod.

Paired with stopping sending traffic when this is called, we can effectively drain a pod before sending the terminate signal. 

Basic example of wait 20 seconds after stopping traffic, then terminate.
```yaml
...
spec:
  containers:
  - name: myservice
    image: ...
    ...

    lifecycle:
      preStop:
        exec:
           command: ["sleep", "20"]
```

This approach works well for simple services that only deal with incoming requests (rather than ones doing background processing on pubsub for example). 


### Graceful Shutdown

Graceful shutdown is **taking the time to finish serving in progress requests and tasks before shutting down. ** 

Default behavior is often to terminate immediately when receiving a Terminate signal.
This must be implemented in the services code, but has the advantage of more flexibility than preStop lifecycle hooks.

At a high level the service needs to do the following

1. Listen for shutdown signal
2. After, wait for current requests
3. Wait for background processes to finish (queue processing, etc.)
4. Exit


## Don't shutdown old pods before new are ready

For this we use "rolling updates".  This means both new and old pods are running for a period during the deploy.

Example with 1 instance of pod, making sure that stays as the minimum number of pods, and allowing at most 2 pods running during deploy.
```yaml
...
spec:
  replicas: 1
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  containers:
  - name: myservice
    image: ...
    ...
```

**Note: **There is alot of tuning you can do around these params to make your deploys go faster.  At the least you'll likely be working with more than one replica, and want more on the maxSurge parameter.

**Note: **These params can be percentages as well, which will work better with higher instance counts.

## Testing

To test we'll redeploy while sending requests, and check our status codes. 

Pseudo code to illustrate
```
setupDeploy()

statusCodeCounts = {}
async while true { sendRequests(statusCodeCounts) }

for i = 0; i < 10; i++ {
  redeploy()
}

// Assert we had responses
assert statusCodeCounts[200] > 0
// Assert that all requests were status code 200
assert len(statusCodeCounts) == 1 
```
You can find a full code version  [here](https://github.com/jimmiebtlr/blog_code/blob/main/nailing_zero_downtime_deployments_in_k8s/tests/infra_test.go) .


## Conclusion

Zero-downtime in kubernetes is quite simple to accomplish and will keep your users happier.  

If you found this useful please like/subscribe/share.  


## PS: Code Repo

You can find the full code used  [here](https://github.com/jimmiebtlr/blog_code/tree/main/nailing_zero_downtime_deployments_in_k8s), including terraform config, tests, and a test service.
