---
tags:
  - GRPC
source: https://itnext.io/grpc-name-resolution-load-balancing-everything-you-need-to-know-and-probably-a-bit-more-77fc0ae9cd6c
---
# gRPC Name Resolution & Load Balancing on Kubernetes: Everything you need to know (and probably a bit more)

Lately, we’ve faced a peculiar problem in our company. To give you a bit of context, we maintain a fleet of hundreds of micro-services in various languages, mostly Python and Go, mainly communicating via gRPC on top of a couple of self hosted Kubernetes clusters.
The problem became apparent while we were trying to do zero downtime rolling updates. With a pretty basic Kubernetes deployment configuration like the one below (and to be honest, a little more care! Read  [here](https://learnk8s.io/graceful-shutdown)  for more info) one might expect their services to stay roughly 100% available while performing the rolling update:

```
apiVersion: apps/v1
kind: Deployment
metadata:
 # ...
spec:
 replicas: 5
   # ...
 template:
   # ...
 strategy:
   type: RollingUpdate
   rollingUpdate:
     maxUnavailable: 2
     maxSurge: 3
```


But we noticed that we’re facing temporary downtimes (less than a minute) almost every time that the rolling update was happening. Likewise, killing a subset of pods resulted in the same thing. It became apparent after a thorough investigation that the problem was how gRPC resolved service names. Let’s dive deep into it, shall we?
![](https://miro.medium.com/v2/resize:fit:700/1*UAycTbrUGt41XAs6YK7RRg.png) 
> DISCLAIMER: I make lots of mistakes on a daily basis! If you notice any, please let me know.This blog is a digest of many amazing blogs by amazing people. I’ll try to put a link to each one wherever possible. Also you can find links to all the references at the end of the blog as well as a  [Github Repo](https://github.com/HomayoonAlimohammadi/grpc-kuber-load-balancing)  for a sample project regarding this subject.

![](https://miro.medium.com/v2/resize:fit:700/1*4T3ovMBvGpmHGVp29redqg.png) 


# Kubernetes Load Balancing

Kubernetes default load balancing happens at ** the connection level**  (per TCP connection,  **L3/L4** ). HTTP/1.1 introduced the concept of “Persistent Connections” which allows multiple request/response pairs to be sent over a single connection (one after the other). While this allows saving some resources, it appears troublesome while we consider how Kubernetes is going to load balance the calls from a client to a backend server.
Most likely in a Kubernetes environment a service is going to have multiple replicas in order to prevent having a single point of failure. Imagine our client and server communicate via HTTP/1.1. On the first call, a TCP connection is created between our client and  **one of the many backend services ** and request/response pairs travel back and forth on the same connection. This means that the rest of the backend replicas are left untouched, i.e. no load balancing at all!
In order to increase throughput, multiple TCP connections are created between client and backend replicas in order to reduce the waiting time of each request for other requests to complete. This means that if we assume a backend service replica (endpoint) is selected (roughly) at random for each TCP connection, and the load distribution is (roughly) uniform amongst different TCP connections, then the TCP connection based load balancing that Kubernetes offer might be “good enough” for our HTTP/1.1 client/server architecture.
Also, as a bonus point, configuring  `MaxConnectionAge`  for persistent connections can contribute to the load balancing as it can ensure connections are destroyed and recreated in smaller intervals (hence super low / super high load connections won’t stick around for too long).


# gRPC Load Balancing on Kubernetes

What about gRPC? Since it’s on top of HTTP/2 and utilizes  **multiplexing**  and  **streams ** (i.e. sending requests and responses concurrently), it can perform amazingly well using only a single TCP connection without having to create multiple connections like HTTP/1.1 does. But what happens to do connection based load balancing in Kubernetes? Well, you guessed it! It’s non-existant!
![](https://miro.medium.com/v2/resize:fit:380/1*zzi0JzwXO-ejHa-qW9f9YA.png) 
In fact, some kind of load balancing does occur while creating this single TCP connection, but as soon as it’s created, since no more connections are going to be created, nothing is load balanced any further. To slowly work our way through the problem, let’s analyze what we are working with step by step.


# Kubernetes Services

In Kubernetes, services that get assigned an IP address and port (e.g. ClusterIP) are just simply  **iptables rules** . No process is listening on their IP and port. Kubernetes uses kube-proxy and iptables to direct traffic to pods without an actual load balancing process. iptables can be configured in a smart way to somehow act as a random pick load balancer like below:
![](https://miro.medium.com/v2/resize:fit:700/1*q5Fqm79CtuGm9bs_n4rsxg.png) 
Here are the rules:
- Select pod 1 as the destination with a probability of 33%. Otherwise, move to the next rule
- Select pod 2 as the destination with a probability of 50%. Otherwise, move to the next rule
- Select pod 3 as the destination

Each pod has a 33% probability (1/3 to be exact) to get picked. All that said, Kubernetes load balancing is not round-robin or anything, it’s a random selection.
So when we use an IP assigned to a, let’s say, ClusterIP service, we are essentially looking up the iptables to rewrite the destination IP with another one belonging to a pod that the service abstracts.
In case of a gRPC communication, the iptables are invoked upon the first RPC since we need to establish a TCP connection. Although for further RPCs, no connection needs to be created, and so the iptables will not be invoked hence no load balancing occurs.


# Load Balancing Options

It seems like that the Kubernetes default load balancing mechanism is not going to cut it for us, so let’s consider and evaluate other options:
 **Proxy (Server Side) Load Balancing** 
- Clients send requests to a load balancer proxy, which then distributes the call to backend servers
- Suitable for user-facing services with untrusted clients from the open internet
- Backends report load to the LB for fair distribution

![](https://miro.medium.com/v2/resize:fit:574/0*N6gUb95-tqnFVGnW) 
 **Client Side Load Balancing** 
- Clients are aware of multiple backends and choose one for each RPC, using server load reports to inform decisions
- Simpler configuration may not consider server load, relying on round-robin selection

this type of load balancing mostly has two variations:
-  **Thick Client:**  Integrates load balancing logic, mantaining server availability, workload, and selection algorithms, often in conjuction with service discovery and other infrastructure libraries.

![](https://miro.medium.com/v2/resize:fit:563/1*h3ZLSh4LB1f0c7riHaOsGQ.png) 
-  **Lookaside Load Balancing:**  Offloads balancing to a special LB server, with clients querying for the best servers to use, allowing sophisticated algorithm implementtaion on the LB side while enabling simple client-side logic.

![](https://miro.medium.com/v2/resize:fit:551/1*p7nZ3YYuNpuX8mBRlAHPLQ.png) 
Now that we know options, let’s first choose to go with the the “Thick Client” one and see what we’re gonna need.


# Implementing Client Side Load Balancing

Our implementation has two main parts. A name resolver and a load balancer. Let’s start with the name resolver.


# Name Resolver

Name resolvers are responsible for resolving available endpoints for a given backend service. One of the most important things while implementing a resolver is to make sure endpoints are maximally up-to-date at any given point in time. This means that old endpoints (which their pods are deleted) should not be available in address pool while the new endpoint (which their pods are just created) should.
Let’s consider our options for the name resolver with regards to  `grpc-go` 
- Passthrough (default): Just returns the target provided by the  `ClientConn`  without any specific logic.
- DNS: Periodically resolves a name to a list of endpoints and updates the  `ClientConn`  on each resolution.
- Custom (more on this later)

How can we choose which name resolver we want to use? The name (and type) of a resolver is tightly coupled with what we provide in the target string:

```
grpc.Dial("myservice.namespace.cluster.local") // target
```


The string above is called the  `target` . The general format for a target is like this:

```
[scheme]://[authority]/endpoint
```


The first example we provided does not have a  `Scheme`  nor an  `Authority` . So the default values for both of them are going to be used. Let’s consider another examples which define schemes:

```
grpc.Dial("dns:///myservice.namespace.cluster.local")
// or
grpc.Dial("myScheme:///myservice.namespace.cluster.local")
```


In the first example  `dns`  is the scheme specified in the target while  `myScheme`  is specified in the second example.
All the name resolvers, whether they are implemented in  `grpc-go`  by default (e.g.  `passthrough`  or  `dns` ) use some kind of a slug as their unique identifier. As you might have guessed, the slug for the  `passthrough`  implementation is  `passthrough`  and the for  `dns` , it’s  `dns` 
Upon calling  `grpc.Dial` , if we’ve specified a scheme in target, it will be used to find its corresponding resolver and that resolver is going to do the name resolution for the rest of our client lifecycle.
If the scheme is not specified in the target though, the  `defaultScheme`  is going to be used which is just a variable created somewhere in  `grpc-go` 

```
var defaultScheme = "passthrough"

// default scheme can be changed and it should 
// happen before grpc.Dial in order to have effect
func SetDefaultScheme(s string) {
  defaultScheme = s
}
```


How is this slug used anyway? Well this slug maps us to a certain  `Builder`  for that name resolver. So having the resolver slug leads us to the resolver Builder which will “build” the resolver for us (i.e. starts name resolution with that specific resolver).

```
var m = make(map[string]Builder)

// this function is used to search for the builder
// which corresponds to our specified scheme in the target
func Get(scheme string) Builder {
 if b, ok := m[scheme]; ok {
  return b
 }
 return nil
}

// if no scheme was provided or the scheme was unknown 
// (the function above returned nil) we fall back to the default scheme
func GetDefaultScheme() string {
 return defaultScheme
}
```


So that’s how the name resolver is selected. Let’s see what’s the main duty of a every name resolver.
The resolver is missioned to update the state of a client connection like below:

```
func (r *resolver) resolve() {
  NewState := gatherNewState()
  r.clientConn.UpdateState(NewState)
}
```


The Updated state contains both the new (updated) endpoints as well as a  `ServiceConfig. ServiceConfig`  is something that will come handy in the next step, where we need to choose and implement a load balancing policy. Here is how the  `State`  struct looks:

```

type State struct {
 // Addresses is the latest set of resolved addresses for the target.
 //
 // If a resolver sets Addresses but does not set Endpoints, one Endpoint
 // will be created for each Address before the State is passed to the LB
 // policy.  The BalancerAttributes of each entry in Addresses will be set
 // in Endpoints.Attributes, and be cleared in the Endpoint's Address's
 // BalancerAttributes.
 //
 // Soon, Addresses will be deprecated and replaced fully by Endpoints.
 Addresses []Address

 // Endpoints is the latest set of resolved endpoints for the target.
 //
 // If a resolver produces a State containing Endpoints but not Addresses,
 // it must take care to ensure the LB policies it selects will support
 // Endpoints.
 Endpoints []Endpoint

 // ServiceConfig contains the result from parsing the latest service
 // config.  If it is nil, it indicates no service config is present or the
 // resolver does not provide service configs.
 ServiceConfig *serviceconfig.ParseResult

 // Attributes contains arbitrary data about the resolver intended for
 // consumption by the load balancing policy.
 Attributes *attributes.Attributes
}
```


In order to have the updated endpoints for our destination we need to utilized  **Headless ** services in Kubernetes. Since upon resolution, they will return the endpoints of their underlying pods instead of their own IP which will get replaced by one of the pod’s IP by the iptables.
It seems like we’re going to do just fine if we utilize a headless service with the dns resolver as our name resolver, right? Well it’s a bit tricky!
Remember that we said we need to have the most up-to-date version of endpoints in any given time? grpc-go built-in dns resolver has a tricky part that might counter act what we’re trying to achieve. Even after a successful DNS resolution, the resolver for wait for at least 30 seconds before attempting a re-resolution:

```
var (
 // MinResolutionRate is the minimum rate at which re-resolutions are
 // allowed. This helps to prevent excessive re-resolution.
 MinResolutionRate = 30 * time.Second
)
```


We’re hoping to make this rate at least optional via this  [pull request](https://github.com/grpc/grpc-go/pull/6962)  but as the time of writing this blog, it’s nothing we can do to circle our way around it.
Implementing a custom resolver is our last and probably most viable option to reach our goal of up-to-date endpoints. for this custom resolver, we can utilize  **Kubernetes endpoints controller**  in order to get notified about every endpoint change event and act upon them, as well as doing periodic DNS resolutions just to make sure.
To wrap up name resolvers, let’s consider them as “always running processes in the background” that are responsible for providing their underlying  `ClientConn`  with the latest “ `State` ” including updated endpoints as well as another thing called  `ServiceConfig`  (more on this one in the following section).


# Load Balancing

The State which was returned by the name resolver in the previous steps contains both a list of endpoints and a configuration called  `ServiceConfig` . One of the main use cases of the  `ServiceConfig`  is to decide which load balancing policy to go for (e.g.  `round_robin` ).  `ServiceConfig`  is based on a  [proto message](https://github.com/grpc/grpc-proto/blob/master/grpc/service_config/service_config.proto)  but the name resolver returns it in JSON form. Let’s look at an example ServiceConfig:
![](https://miro.medium.com/v2/resize:fit:700/1*-WE9InsNd77Fs-uYt3Jhjg.png) 
The  `loadBalancingConfig`  is what we use in order to decide which policy to go for ( `round_robin`  in this case). This JSON representation is based on a protobuf message, then why does the name resolver returns it in the JSON format?  [The main reason](https://github.com/grpc/grpc/blob/master/doc/service_config.md)  is that  `loadBalancingConfig`  is a  `oneof`  field inside the proto message and so it can not contain values unknown to the gRPC if used in the proto format. The JSON representation does not have this requirement so we can use a custom  `loadBalancingConfig` 
gRPC client can be configured to consider default service config upon dial as well:

```
grpc.Dial(
  target,
  grpc.WithDefaultServiceConfig(`"loadBalancingConfig": [{ "round_robin" }]`),
)
```


ServiceConfig can also be encoded in a DNS TXT record and get resolved by the resolver as well:

```
grpc_config.myserver  3600  TXT "grpc_config=[{\"serviceConfig\":{\"loadBalancingPolicy\":\"round_robin\",\"methodConfig\":[{\"name\":[{\"service\":\"MyService\",\"method\":\"Foo\"}],\"waitForReady\":true}]}}]"
```


After selecting the load balancing policy via the ServiceConfig, we need to know the basic duties and requirements of a load balancer.
Load Balancing policies are responsible for receiving server addresses and configurations, managing sub-channels, setting channel connectivity state (more on this later) and determining a sub-channel for each RPC.
Two of the built-in load balancers in grpc-go are  `pick_first`  and `round_robin` 
 **Connectivity State** 
gRPC channels are an abstraction that describe the overall state of a connection from origin to destination (e.g. service A to service B). Each channel contains a number of sub-channels each one having a so called “Connectivity State” which can be described by a state machine and has the following states:
-  **CONNECTING:**  Channel is attempting to establish a connection, including name resolution, TCP, and TLS handshakes
-  **READY:**  A successful connection has been established, and the channel can facilitate communication
-  **TRANSIENT_FAILURE:**  Temporary failures have occurred, prompting the channel to retry connection establishment with exponential backoff
-  **IDLE:**  The channel is not actively attempting to establish a connection due to lack of RPC activity. New RPCs can transition the channel out of this state
-  **SHUTDOWN:**  The channel is shutting down, either by user request or due to a non-recoverable error, and no new RPCs will be initiated

Depending on the implementation of the load balancer, the channel will evaluate its overall state by aggregating states from all its sub-channels.
So let’s wrap up what we’ve learned about load balancers. Via  `ServiceConfig`  we can decide which policy to use. The implementation is tasked with obtaining updated endpoints from the name resolver, creating and managing TCP connections with each one of them and route a given RPC upon arrival to one of those connections based on a specific logic.


# gRPC Client Initialization Flow

![](https://miro.medium.com/v2/resize:fit:560/1*e1zIlQWWw8Nckmcni-KRgw.png) 
![](https://miro.medium.com/v2/resize:fit:700/0*Wss6rKhWjo38PCe9) 
The pictures above depict how the gRPC client initialization happens under the hood.


# Availability in rolling updates

Another thing to keep in mind, which to be honest does not relate directly to the gRPC load balancing is how a rolling updates occur and how they can affect availability.
You might notice temporary downtimes while performing rolling updates even if you’ve configured everything as we pictured just fine. That’s because the signal to change the iptables and exiting the process happens in parallel (we might get unlucky)
![](https://miro.medium.com/v2/resize:fit:700/0*JTtcI2gTNVfdGVBU) 


# Try it yourself

As pictured in the sections above, proper gRPC load balancing can be achieved via different ways (e.g. Thick Client or Proxy LB).  [In this GitHub repo](https://github.com/HomayoonAlimohammadi/grpc-kuber-load-balancing)  there is a sample project which you can play around with and see for yourself how all these pieces of puzzle come together.
It requires you to have a Kubernetes cluster up and running for which I recommend using  [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/) 
There’s a couple of HTTP/1.1 services, one acting as the client and the other one acting as the server. You can observe the logs to see that HTTP/1.1 is actually load balanced well enough on Kubernetes.
On the other hand, a pair of gRPC services are available that act exactly the same as their HTTP/1.1 counter parts, except the fact that they communicate via gRPC. Ramp up the replicas of the server and witness that client calls are not balanced at all (all land on the same pod). You can monitor what’s happening under the hood thanks to  [channelz](https://grpc.io/blog/a-short-introduction-to-channelz/)  as well (it’s implemented in the gRPC services)
From here, you can install and inject LinkerD to your gRPC deployments (as described in the README instructions) and after that it becomes apparent that LinkerD is doing an amazing job load balancing the requests to all the available pods of the server.
![](https://miro.medium.com/v2/resize:fit:570/1*_4-jiLpwYkPjEFpCcTkauA.png) 


# Conclusion

Load balancing gRPC requests on Kubernetes can be challenging. In this blog we tried to deep dive into the inner-workings of each part and find how everything work and why they sometimes don’t.
If you’ve liked this blog, a share and a clap would really mean a lot to me. Also you can find links to all the references used below.
Feel free to contact me via  [LinkedIn](https://www.linkedin.com/in/homayoon-alimohammadi)  and  [Twitter](https://twitter.com/HomayoonAlm) . Also you can find more about me on my  [personal blog](https://homayoon.blog/) 


# References

-  [https://github.com/HomayoonAlimohammadi/grpc-kuber-load-balancing](https://github.com/HomayoonAlimohammadi/grpc-kuber-load-balancing) 
-  [https://medium.com/swlh/balancing-grpc-traffic-in-k8s-without-a-service-mesh-7005be902ef3](https://medium.com/swlh/balancing-grpc-traffic-in-k8s-without-a-service-mesh-7005be902ef3) 
-  [https://github.com/grpc/grpc-go/tree/master/examples/features/keepalive](https://github.com/grpc/grpc-go/tree/master/examples/features/keepalive) 
-  [https://github.com/grpc/grpc/blob/master/doc/connectivity-semantics-and-api.md](https://github.com/grpc/grpc/blob/master/doc/connectivity-semantics-and-api.md) 
-  [https://grpc.io/blog/grpc-load-balancing/](https://grpc.io/blog/grpc-load-balancing/) 
-  [https://kubernetes.io/blog/2018/11/07/grpc-load-balancing-on-kubernetes-without-tears/](https://kubernetes.io/blog/2018/11/07/grpc-load-balancing-on-kubernetes-without-tears/) 
-  [https://learnk8s.io/kubernetes-long-lived-connections](https://learnk8s.io/kubernetes-long-lived-connections) 
-  [https://github.com/grpc/grpc/blob/master/doc/load-balancing.md](https://github.com/grpc/grpc/blob/master/doc/load-balancing.md) 
-  [https://github.com/grpc/grpc/blob/master/doc/connection-backoff.md](https://github.com/grpc/grpc/blob/master/doc/connection-backoff.md) 
-  [https://fuchsia.googlesource.com/third_party/grpc/+/HEAD/doc/load-balancing.md](https://fuchsia.googlesource.com/third_party/grpc/+/HEAD/doc/load-balancing.md) 
-  [https://github.com/grpc/grpc/blob/master/doc/service_config.md](https://github.com/grpc/grpc/blob/master/doc/service_config.md) 
-  [https://github.com/grpc/grpc-proto/blob/master/grpc/service_config/service_config.proto](https://github.com/grpc/grpc-proto/blob/master/grpc/service_config/service_config.proto) 
-  [https://github.com/grpc/proposal/blob/master/A2-service-configs-in-dns.md](https://github.com/grpc/proposal/blob/master/A2-service-configs-in-dns.md) 
-  [https://learnk8s.io/graceful-shutdown](https://learnk8s.io/graceful-shutdown) 
