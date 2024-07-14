---
tags:
  - ARGOCD
source: https://blog.argoproj.io/argo-cd-v2-11-release-candidate-b83ba3008ba5
---

---



# Argo CD v2.11 Release Candidate

We are happy to announce that the Argo CD v2.11 Release Candidate has been published! We have summed up over 32 new features, 40 bug fixes, and 49 documentation updates.
As always, please test the release candidate and send us feedback about any bugs or issues you encountered. This is your big opportunity to make your voice heard and help Argo CD become even better.
![](https://miro.medium.com/v2/resize:fit:700/0*gguIsZv15HrjXDPG) 


# See who initiated the sync in the history UI

Argo CD has a dedicated screen for showing the history of sync operations for an application, along with an optional “rollback” method that points the cluster resources to a previous Git hash.
This screen is now enhanced with an “initiated by” property that shows the person that started the sync process:
![](https://miro.medium.com/v2/resize:fit:700/0*zDXXZS6fYTlnA0VU) 
This new feature will be very interesting to teams that have Argo CD applications without the auto-sync option enabled and they want to know which person started a deployment (a very popular pattern for production environments).
The value of the property is coming from the Application Resource where a new “initiatedBy” property was added accepting any string value.
Thanks  [Robin Lieb](https://github.com/robinlieb)  (Mercedes Benz) for implementing this feature.


# Improved monorepo support with the generate-path annotation

This is arguably the most important feature of this release and a lot of organizations will be interested to see how it improves both the speed and the user experience of Argo CD installations that are based on mono-repos.
The mono-repo pattern is the case where a single Git repository holds all Argo CD applications for all clusters. While this pattern makes things easy for maintenance and central administration, it also presents a lot of challenges for large Argo CD installations.
Argo CD already had several mechanisms in place to avoid performance issues with these patterns. One of the basic features is the  [ *manifest-generate-paths * annotation ](https://argo-cd.readthedocs.io/en/latest/operator-manual/high_availability/#manifest-paths-annotation) which can be placed on an application and gives some hints to Argo CD for which manifests the application is dependent on.
Until the previous Argo CD version this annotation was only honored for Git webhooks. When a Git webhook was detected Argo CD could detect if the files mentioned in the webhook were part of the set mentioned in the annotation and act accordingly.
The new feature in Argo CD v2.11 is that now these annotations are honored in sync operations as well and not just during a webhook event.
Now when Argo CD wants to generate the manifests of an application, it will compare the latest commit to the last cached revision according to the contents of the  *manifest-generate-paths * annotation. If nothing matches, the no sync operation will happen at all.
This is a game-changing feature as it affects the way large monorepos are handled by Argo CD. Performance improvements should be seen as the Argo CD controller will perform less sync operations especially for applications that are not affected by any of the files found in the latest commit.
On the other hand User Experience is also improved, as now users will be able to look at the Argo CD history UI and see only a list of commits that actually affected the application in question.
We hope that you test this feature extensively and let us know of the performance benefits and improvements or any issues that you encounter.
Thanks  [Alexy Mantha](https://github.com/alexymantha)  (GoTo) for implementing this feature.


# Multi-source application goodness

Creating Argo CD applications  [from multiple sources](https://argo-cd.readthedocs.io/en/latest/user-guide/multiple_sources/)  was one of the most requested Argo CD features for a long time. This feature allows you to group information from multiple places (for example a public Helm chart plus your local values file) to form a single Argo CD application.
Initial support for defining multi sources was already added in Argo CD version 2.6, but there were several places in the CLI and the UI where an application was assumed to have a single source.
With Argo CD version 2.11, multiple application sources are now supported in several more areas such as the Argo CD CLI app subcommands like add source, remove source, create, edit, set, unset, diff, manifests and history.
This means that you can now manage multiple sources using also the Argo CD CLI (in addition to the CRD).
Thank you  [Mangaal Angom Meetei](https://github.com/Mangaal)  and  [Ishita Sequeira](https://github.com/ishitasequeira)  (Red Hat) for implementing these features.


# Allow cluster sharding by app information

If you follow the “hub-and-spoke” architecture with Argo CD, there is a capability to employ sharding for different Argo CD application controllers. This allows you to assign specific clusters to specific controllers in order to distribute the load across the different shards.
Until recently the main sharding algorithm was based on cluster information only (hashing their ids). While this approach works in most scenarios, there are several cases where you want to have more control on how sharding works.
In the new Argo CD release there is now support for sharding clusters according to the number of applications they control. This is just the starting point for implementing several different sharding algorithms in Argo CD in the near future for even more flexible configurations.
Thanks  [Andrew Lee](https://github.com/Enclavet)  (AWS) for implementing this feature.


# Other Notable Changes

Here are some other new changes that made it into the release
-  [Apps in any namespace feature](https://argo-cd.readthedocs.io/en/latest/operator-manual/app-any-namespace/)  has been  [promoted from beta to stable](https://github.com/argoproj/argo-cd/pull/17529) 
- Enable users to  [run commands related to Argo application in any namespace](https://github.com/argoproj/argo-cd/pull/17360)  (By  [Mangaal Angom Meetei](https://github.com/Mangaal)  from Red Hat)
- Add ability to  [auto label clusters](https://github.com/argoproj/argo-cd/pull/17289)  for Kubernetes clusterinfo (By  [Blake Pettersson](https://github.com/blakepettersson)  from Akuity)
- Prune resources [ in reverse order of syncwave](https://github.com/argoproj/argo-cd/pull/16748)  during sync phases (By  [Siddhesh Ghadi](https://github.com/svghadi)  from Red Hat)
- New option to  [specify an aws profile](https://github.com/argoproj/argo-cd/pull/16767)  to use by argocd-server when adding an EKS cluster (By  [Isaac Gaskin](https://github.com/igaskin)  from Circle Fin)
- Enable  [–app-namespace flag](https://github.com/argoproj/argo-cd/pull/17437)  for argocd CLI (By  [Mangaal Angom Meetei](https://github.com/Mangaal)  from Red Hat
-  [maxPodLogsToRender](https://github.com/argoproj/argo-cd/pull/14617)  setting (By  [Lukas](https://github.com/lukaszgyg) 
- Add cli commands to  [add/delete sourceNamespaces](https://github.com/argoproj/argo-cd/pull/17337)  from AppProject (By  [Raghavi Shirur](https://github.com/raghavi101)  from Red Hat)
- Allow webhook settings to be referenced  [by external secret](https://github.com/argoproj/argo-cd/pull/16262)  (By  [Arthur Outhenin Chalandre](https://github.com/MrFreezeex)  from Ledger)
- Wait  [until resources are deleted](https://github.com/argoproj/argo-cd/pull/16733)  when using argo app wait command (By  [Michael Morris](https://github.com/MichaelMorrisEst)  from Ericsson)



# Where Can I Get the New Release?

For more details and installation instructions, check the  [release notes](https://github.com/argoproj/argo-cd/releases/tag/v2.11.0-rc1) , and  [upgrade instructions](https://argo-cd.readthedocs.io/en/latest/operator-manual/upgrading/2.10-2.11/) . Please try the release candidate and share your feedback. A big thanks to all Argo Community contributors and users for their contributions, feedback, and help in testing the release!
Photo by  [Lora Seis](https://unsplash.com/@diasplora?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash)  [Unsplash](https://unsplash.com/photos/a-stuffed-animal-with-a-hat-dS5xpjW38Qk?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash) 