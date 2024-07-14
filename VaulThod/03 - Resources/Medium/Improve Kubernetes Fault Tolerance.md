---
tags:
  - KUBERNETES/POD/EVICTION
source: https://medium.com/@muthanagavamsi/improve-kubernetes-fault-tolerance-f29651736053
---




# Improve Kubernetes Fault Tolerance

Kubernetes: Until today, I don’t know this 👇
It’s crazy. Because you can control pod eviction time.
First, let’s understand this.
Why does a Pod even get evicted?
Maybe because of:
1. Resource pressure on the node\
2. Pods exceeding resource limits.
Fair enough?
![Kubernetes Fault Tolerance](https://miro.medium.com/v2/resize:fit:700/1*CjREJORMdysWOJ9YW6qmGQ.gif) 
Now how to control eviction?
By setting evictionPressureTransitionPeriod in Kubelet config.
Let’s say to 120 seconds.
Hmm, what does it do?
It enables a 120 second grace period before a Pod is automatically evicted.
An evicted Pod in a deployment, goes to a new node.
Is it Good?
Of course, it’s stupendously useful for your app in production.
This will support system fault tolerance.
Alternatively, on your Dev/QA
set “evictionPressureTransitionPeriod: 0s” to speed up eviction process.
Restart Kubelet after making changes.
That’s it for today.
If this is useful, do a Repost. It really helps ♻️
 [Mutha Nagavamsi](https://medium.com/u/ec54a8ebe044?source=post_page-----f29651736053--------------------------------) . Follow me for Kubernetes, Devops and tech content.
Before you leave, don’t forget to SMILE 😁
Thank you so much for reading this. If you found it interesting, do spread the word about it. You may also find my other content interesting, find them below.
 [Mutha Nagavamsi on Youtube.](https://www.youtube.com/@muthanagavamsi)  (Subscribe, it really helps)
 [Me on Substack.](https://mutha.substack.com/) 
 [Me on X.](https://twitter.com/MuthaNagavamsi) 