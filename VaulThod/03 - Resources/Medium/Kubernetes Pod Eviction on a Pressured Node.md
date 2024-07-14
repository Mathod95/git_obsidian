---
tags:
  - KUBERNETES/POD/EVICTION
source: https://medium.com/@muthanagavamsi/kubernetes-pod-eviction-on-a-pressured-node-6d0a506f5a2c
---




# Kubernetes Pod Eviction on a Pressured Node

Kubernetes pod eviction is PAINFUL 👇
But sometimes, it had to be done.
Okay.
In our story,
we have 3 happy pods on Node-1.
![](https://miro.medium.com/v2/resize:fit:700/1*Gv1lLYy3OUMv6Vf5g5K9hA.gif) 
And Memory spikes up high on NODE-1: Now what?
“Kubernetes has a decision to make.”\
“It must evict a pod.” \
“Which one?”
That’s where QoS classes helps Kubernetes.
At any point, a Pod can have 1 of the following QoS classes:
1. Guaranteed\
2. Burstable\
3. BestEffort
When a Node is under pressure,
Pod with BestEffort QoS takes priority for eviction.
Burstable next.\
Guaranteed last.
Unfortunately, Pod3 has BestEffort QoS.
So gets evicted first.
Happy endings though. Pod3 gets welcomed on Node-2 ☺️
Hope it helped. Repost if useful ♻️
If you are interested in Kubernetes & devops, checkout Mutha Nagavamsi.
P.S. Share your thoughts about QoS classes below. It helps everyone.
Thank you so much for reading this. If you found it interesting, do spread the word about it. You may also find my other content interesting, find them below.
 [Mutha Nagavamsi on Youtube.](https://www.youtube.com/@muthanagavamsi)  (Subscribe, it really helps)
 [Me on Substack.](https://mutha.substack.com/) 
 [Me on X.](https://twitter.com/MuthaNagavamsi) 