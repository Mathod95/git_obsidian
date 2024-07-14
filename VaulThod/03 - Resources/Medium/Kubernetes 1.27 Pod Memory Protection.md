---
tags:
  - KUBERNETES/MEMORY_PROTECTION
source: https://medium.com/@muthanagavamsi/kubernetes-1-27-pod-memory-protection-d23618f5f996
---




# Kubernetes 1.27: Pod Memory Protection

Kubernetes 1.27: MemoryThrottlingFactor is awesome⚡
It’s a kubelet configuration that controls how much memory a pod can use before it is throttled.
The default value is 0.9, which means that a pod can use up to 90% of its memory limit before it is throttled.
![](https://miro.medium.com/v2/resize:fit:700/1*bhqXtj5AwcsJP5zuDT90CQ.gif) 
Picture this:
If a pod has a memory limit of 100 MiB and we set the MemoryThrottlingFactor to 0.9,
it’s like saying, “Hey pod, you have 90 MiB of memory!” After crossing the limit, throttling is applied.
A throttled pod runs slowly but happily avoiding process termination because of Out of Memory (OOM) error. ☺️
You might ask:
We already have “resource limits” for this, why MemoryThrottlingFactor?
→ That’s the tricky part, sometimes a pod might get a sudden influx of data and start using more memory than expected (it’s limits). \
→ That’s where MemoryThrottlingFactor comes in! It’s a soft limit, therefore a pod may cross 90% for short period of time. \
→ It will eventually be throttled back down, that way, we avoid performance problems on the Pod.
That’s it for today. Hope this is useful. A Repost really helps ♻
PS: I’m  [Mutha Nagavamsi](https://medium.com/u/ec54a8ebe044?source=post_page-----d23618f5f996--------------------------------) , follow me for simplified Kubernetes, tech and cybersecurity content.