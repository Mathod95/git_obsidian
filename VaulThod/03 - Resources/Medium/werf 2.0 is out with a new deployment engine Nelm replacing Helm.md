---
tags:
  - APP/WERF
  - APP/OPENTELEMETRY
source: https://blog.werf.io/werf-2-nelm-replacing-helm-a11980c2bdda
---




# werf 2.0 is out with a new deployment engine Nelm replacing Helm

For four years, we have been developing and improving werf 1.2. Now, we are proud to unveil werf 2.0 stable! It accumulates all changes delivered to werf throughout the last 300+ releases and comes with Nelm — our new deployment engine, replacing Helm. Nelm is backward compatible with Helm, so there’s no need to make any special changes to the charts — you can use them just like before.
![](https://miro.medium.com/v2/resize:fit:700/1*S1szfWTLdr_g9jfXyRpp4A.png) 


# A brief reminder of what werf is

 *Feel free to skip this part of the article if you are an existing werf user or just aware of this project!* 
 [werf](https://werf.io/)  is an Open Source tool for powering your Kubernetes-based CI/CD pipelines. It works alongside the CI system of your choice and handles the entire CI/CD lifecycle: builds container images, deploys them to Kubernetes clusters, and eventually deletes them.
To deploy your apps with werf, you just need a Git repository with a Helm chart, a simple  `werf.yaml`  file and a Dockerfile. With such a repo in place, run the  `werf converge`  command to build the images, publish them to the container registry, and deploy them to your Kubernetes cluster.
To make it possible, werf relies on well-known technologies such as Docker, Buildah and Helm (or Nelm since now) under the hood. But it’s more than just a mere wrapper. For example, werf brings several unique features, such as distributed caching out-of-the-box, automatic tagging based on the image content, smart container registry cleanup based on special Git policies, and a number of other niceties.
Since December 2022, werf is a  [CNCF Sandbox project](https://www.cncf.io/projects/werf/) . In the past month, we’ve seen 10,000 active projects using werf (usually, one such project equals one Git repository where werf is applied). Today, werf boasts almost 4,000 stars on  [GitHub](https://github.com/werf/werf/)  and 8 years of very active and robust development.


# What Nelm is and what the future holds for it

 [Nelm](https://github.com/werf/nelm)  is the biggest change in werf 2.0, so let’s have a better take on it. We can briefly describe Nelm as our (partial) reimplementation of Helm 4 — the release we all have been waiting for but have never seen.
Helm itself consists of two key components: a chart subsystem and a resource deployment subsystem. We essentially rewrote the deployment subsystem from scratch (while maintaining backward compatibility). We have also improved and continue to refine the chart subsystem.
 ** *Note!* **  * Currently, you can try *  [ *Nelm*  ](https://github.com/werf/nelm/) * only as part of werf. However, in the future, it will become a standalone tool with a convenient API and the option to integrate it into other CI/CD solutions.* 
This is what Nelm brings into werf:
- The 3-Way Merge has been replaced by Server-Side Apply — a much more robust mechanism for updating resources in a cluster.
- The  `werf plan`  command shows the changes that will be made to the cluster during the next deployment.
- Resource operations (including tracking) during deployment have been efficiently parallelized.
- CRDs deployment has been improved.
- Resource tracking has been significantly improved and revamped.
- Numerous Helm bugs and deployment-related issues (e.g.,  [#6969](https://github.com/helm/helm/issues/6969) ) have been fixed.

 **  [ *Here is*  ](https://github.com/werf/werf/discussions/5657) * our GitHub discussion thread where most of these Nelm features were announced as we implemented them.)* 
![](https://miro.medium.com/v2/resize:fit:700/1*U3jW1Lpim740NrRcMIpoBQ.png) 
We’re working on a few more features, like the ability to set direct resource dependencies instead of using hooks, weights and init containers, as well as the ability for regular resources to use all the advanced hook features. We will announce them later as they become generally available.
You can learn more about Nelm in the next article we will publish soon.  *(By the way, *  [ *our Telegram group*  ](https://t.me/werf_io) * is a great way to stay tuned for updates!)* 


# How to try werf v2.0

Nelm has slightly different behavior in some cases (compared to Helm), such as stricter validation of charts. That’s why we decided to make it a default engine not in werf v1.2, but in v2.0 only.
 **werf v2.0 is almost fully compatible with v1.2**  —  [here is the list](https://werf.io/documentation/v2/resources/migration_from_v1_2_to_v2_0.html)  of backward-incompatible changes (it’s really tiny!). We recommend upgrading to version 2.0, which is much easier than it was when migrating from werf v1.1 to v1.2. What about werf v1.2? It goes into  *maintenance*  mode — no new features are planned for it.
Another big change is about version numbering — starting with werf 2.0,  **we will stick to semantic versioning**  and plan to release a major version about once a year. This will allow us to streamline and speed up the development without risking compromising backward compatibility in minor or patch versions. On the other hand, this will allow us to be more careful about backward compatibility in minor and patch updates.
Use the following command to try werf v2.0:

```
source $(trdl use werf 2 stable)
```


As a reminder, werf comes with several  [release channels](https://werf.io/about/release_channels.html) 
-  **Alpha** . Quick to deliver new features, but may be unstable.
-  **Beta** . Best suited for more extensive testing of new features in order to find problems.
-  **Early-Access** . Safe enough for non-critical environments and for local development; allows you to get new features earlier.
-  **Stable** . Generally safe and recommended for widespread use in any environment as a default option.
-  **Rock-Solid** . The most stable channel; recommended for critical environments with strict SLA demands.



# The long road from werf v1.2 to v2.0

Now, back to those very “300+ releases” mentioned earlier — werf has indeed accumulated quite a lot of new features and changes over all those years. Here are some of the most significant features that have emerged in the process of making werf 1.2 (not related to Nelm):
1.  Building Dockerfiles in werf using Buildah under Linux, Windows, and macOS.
2.  Layered caching in registry for Dockerfiles.
3.  Out-of-the-box support for building images for arbitrary platforms and for multiple platforms at once.
4.  The development mode (  `--dev` ) allowing you no longer worry about determinism and intermediate commits during debugging and developing.
5.  A new directive for dependencies images has been added to werf.yaml (as of version 1.2.60).
6.  The  `werf bundle render`  command to render bundle manifests for further deployment by third-party tools or for debugging.
7.  The  `werf kube-run`  command, which is similar to  `werf run` , but instead of a local container, it runs a pod in a K8s cluster.
8.  Status tracking and event collection for all resource types, not just Deployments/StatefulSets/DaemonSets/Jobs.
9.  The option to wait for an external (out-of-release) Kubernetes resource to be ready before deploying a release resource.
10.  Migration to the new  [trdl](https://trdl.dev/)  update manager. By the way, it is another Open Source project cultivated by the werf team.



# Official werf resources

-  [werf website](https://werf.io/) 
-  [werf Telegram chat](https://t.me/werf_io) 
-  [werf repo on GitHub](https://github.com/werf/werf) 
-  [Nelm repo on GitHub](https://github.com/werf/nelm) 

Let us know how your migration to werf v2.0 is going and stay tuned for more upcoming news regarding werf v2.0.x & Nelm updates!