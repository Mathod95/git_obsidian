---
tags:
  - KUBERNETES
source: https://medium.com/adidoescode/adidas-how-we-are-managing-a-container-platform-2-3-ce551abab337
---




# How we are managing a container platform: a tale about the present

In our previous posts, we explored how adidas built and managed its container platform. It worked fine at a smaller scale, but as we grew, we faced challenges. The goals for the new approach to managing adidas’ container platform were set:
-  **Reduce operational time.** 
-  **Efficiently manage platform clusters.** 
-  **Enhance process resilience.** 
-  **Improve visibility of configuration status globally and locally.** 



# GitOps!

We encountered a significant challenge in ensuring consistent configuration application across our entire platform. This led to synchronization issues and unexpected behaviors in various regions/clusters. Mainly, this was due to:
-  **The complex setup process for each component.** 
-  **Challenges with the scalability of our continuous deployment system** , intensified by maintenance window restrictions.

Consequently, we opted to shift our approach from a push mode  *(where a system pushes configuration to another system)*  to a pull mode  *(where the system retrieves configuration from a configuration repository)* 


## Configuration Structure

Perhaps one of the most challenging aspects to design and implement was, as mentioned in the previous post, the global operational scale of adidas’ container platform. Considering various internal clients across multiple execution environments, ** the configuration requires a layered approach** 
1.  The first layer includes settings common to all clusters within the platform, known as our  **Global configuration** 
2.  The following layer relates to an  **execution environment**  commonly found in Development, QA, or Production.
3.  Following this is the configuration layer tied to  **geographical zones** . Those managing platforms in China understand the unique considerations here, such as the necessity to overwrite configurations due to the lower speed of pulling images from container repositories in the US or Europe, ensuring the retrieval of data from local sources in China.
4.  The final layer, one we aim to eventually eliminate, is  **cluster-specific configuration** . Occasionally, specific clusters require special attention, such as a critical cluster handling private sales or VIP events.

![](https://miro.medium.com/v2/resize:fit:1000/1*qqvJq-OLUZltL2Kl-KK0pg.png) 
This structure offers the flexibility to independently customize details across these four configuration layers. Changes can be applied globally  *(affecting all clusters)* , per environment, per geographical region, or individually to each cluster  *(with cluster-specific configurations overriding geographical, environmental, and global settings)* 
This adjustment directly addresses the problems and objectives we had outlined earlier:


## Reduce the time spent on operations

![](https://miro.medium.com/v2/resize:fit:700/1*8DqCduiYg-m5JRT9ywOdKg.png) 
 **We’ve made substantial progress in reducing operational time by condensing configurations into a single repository** , specifically, a handful, instead of scattered across numerous repositories.
This consolidation has notably decreased manual operations previously conducted on the platform to verify the correct application of configurations;  **this was a genuine pain point for us.** 
Moreover,  **transitioning from a push model to a pull model has exponentially improved our efficiency** . Instead of a continuous deployment system pushing configurations, all clusters within the platform now concurrently fetch configurations. This minimizes the time spent waiting for configurations  *(or new features)*  to be applied across the platform.


##  **Manage platform clusters more efficiently** 

![](https://miro.medium.com/v2/resize:fit:700/1*vPCdZ5Izgj0HyrdAYN3fEA.png) 
Traditionally known as ‘ **treating clusters as cattle** ’, we’ve gained the capability to scale our container platform up or down without significant worries about the number of clusters. Although there’s continual improvement in this area, our current position far surpasses where we stood a couple of years ago.
Given adidas’ multitude of internal systems and diverse integrations, certain manual tasks are still unavoidable when creating a new cluster. For instance, adding these clusters to adidas’ internal network.  **We’re collaborating with various teams to streamline and secure these actions, aiming for more effective and automated processes** 


## Improve process resilience

In addition to more efficient platform cluster management, we’ve secured the entire process by incorporating a carefully curated set of alerts. Our focus isn’t on flooding with all possible alerts but rather on selecting those that truly matter and could potentially impact platform stability.
Also,  **we can now understand upcoming changes for the platform before they happen** . This is made possible by our continuous integration system, which launches a dry-run of configurations onto the platform, providing insights into the changes that different clusters would undergo if the proposed modifications were accepted by the team.
![](https://miro.medium.com/v2/resize:fit:700/1*e-d0CH-Tvc6AKquDTL1L0A.png) 
![](https://miro.medium.com/v2/resize:fit:485/1*kag7LlnNtuLEyOnlKeokEQ.png) 
We’ve implemented a pre-check to prevent the application of incorrectly structured configurations from the configuration repository. Previously, due to configuration fragmentation across different repositories with distinct structures, this was entirely unfeasible.
Lastly, we’ve implemented a mechanism wherein each cluster autonomously determines when to apply new configurations  *(maintenance windows)* . While the majority of the platform operates without any hourly restrictions, during critical adidas sales events, these mechanisms prevent any turbulence that might affect adidas’ business — remember, adidas is not just about technology.


## Enhance visibility of configuration status globally and locally

In addition to more efficient and secure platform cluster management, we’ve significantly improved the entire process by enhancing visibility and traceability of each cluster’s configuration status and the platform as a whole.
By aggregating metrics from all clusters within the platform, we now have comprehensive visibility into the platform’s status.
![](https://miro.medium.com/v2/resize:fit:1000/1*wPVi0jjip81cyW67Jyc1jg.png) 
This allows us to pinpoint finer details in case any specific cluster isn’t performing as expected within the larger platform.
![](https://miro.medium.com/v2/resize:fit:1000/1*gMjIUFwHn-PoDWNylBEu0g.png) 
These dashboards offer invaluable, read-only insights. Additionally, we receive alerts across multiple channels based on their criticality levels:
![](https://miro.medium.com/v2/resize:fit:700/1*T6cGvrLiwPJsmGdjXLdWXw.png) 
Furthermore, we’ve deployed another dashboard facilitating certain manual operations, eliminating the need for command-line interventions.
![](https://miro.medium.com/v2/resize:fit:1000/1*pW9L3jBYf-54S1b5CvW7cQ.png) 
Finally, we’ve introduced an automated change log generation within the configuration repository itself. This is crucial for understanding how the platform has changed and evolved over time.
![](https://miro.medium.com/v2/resize:fit:700/1*vyGcK9-Hv06dN1ZjgNAKrA.png) 


# Additional Insights

As expected in a company like adidas, this container platform runs components shared across multiple teams, for which we share responsibility. For instance, the entire monitoring stack, traceability, security, API… these are executed within the platform but are owned by other expert teams in those respective areas.
The configuration structure enables these peripheral teams to ask for changes and utilize all the mentioned mechanisms. This has enabled us to strengthen bonds with these teams as we’ve enhanced their processes at no cost to them. In fact, they can operate within different maintenance windows than those of the container platform itself, allowing them greater flexibility.
![](https://miro.medium.com/v2/resize:fit:700/1*ShIhPwV9oXPFqzstN8tLtw.png) 


# Wrapping Up This Chapter: Crossing the Finish Line

In this article, we’ve talked about how adidas manages its global container platform, kind of like completing a tough race. But just like in sports, there’s always room to do better and set new goals.
Next time, we’ll talk about ways to make things even better, like a team passing on the baton in a relay race. We’ll explore how to minimize cluster-specific configurations, automate internal infrastructure components of adidas, manage applications, and much more.
Keep an eye out for the next part where we try to make our container platform even better!
> 
 *The views, thoughts, and opinions expressed in the text belong solely to the author, and do not represent the opinion, strategy or goals of the author’s employer, organization, committee or any other group or individual.* 
