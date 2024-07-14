---
tags:
  - AWS/GUARDDUTY
  - AWS/EKS
source: https://medium.com/@seifeddinerajhi/how-amazon-guardduty-can-help-keep-amazon-eks-secure-26036ce607de
---




# How Amazon GuardDuty can help keep Amazon EKS secure

> 
Top-Notch Protection for EKS with Amazon GuardDuty

![](https://miro.medium.com/v2/resize:fit:700/1*OTszM6hHZ_ZZaOmvXUzIgg.png) 


# 📚 Introduction:

 [ ** *Amazon GuardDuty* **  ](https://aws.amazon.com/guardduty/)offers extended coverage, allowing for ongoing monitoring and profiling of Amazon EKS cluster activities.
This involves identifying any potentially harmful or suspicious behavior that could pose threats to container workloads. The EKS Protection feature within Amazon GuardDuty delivers threat detection capabilities specifically designed to safeguard Amazon EKS clusters within your AWS setup.
> 
This protection encompasses two key components:  [EKS Audit Log Monitoring ](https://aws.amazon.com/about-aws/whats-new/2022/01/amazon-guardduty-elastic-kubernetes-service-clusters/)  [EKS Runtime Monitoring](https://aws.amazon.com/blogs/aws/amazon-guardduty-now-supports-amazon-eks-runtime-monitoring/) 

In this blog post, we’ll explore how Amazon GuardDuty can help you enhance the security of your EKS clusters, providing you with the tools and insights needed to keep your Kubernetes infrastructure safe and secure.


# Amazon GuardDuty:

Amazon GuardDuty is a managed threat detection service that uses a combination of machine learning, anomaly detection, and integrated threat intelligence to identify, flag, and prioritize potential threats.
 [January 2022](https://aws.amazon.com/about-aws/whats-new/2022/01/amazon-guardduty-elastic-kubernetes-service-clusters/)  its capabilities were expanded to include Amazon EKS. \
Key features include:
- No additional software is required to make it run.
- Continuous 24/7 monitoring of your AWS implementation without added cost or complexity.
- Global coverage.
- A system that monitors everything in your account and infrastructure level, alerting you of any anomaly behavior.
- An intuitive automatic threat severity level to help cybersecurity specialists prioritize potential threats.

EKS Protection includes  [EKS Audit Log Monitoring ](https://aws.amazon.com/about-aws/whats-new/2022/01/amazon-guardduty-elastic-kubernetes-service-clusters/)  [EKS Runtime Monitoring](https://aws.amazon.com/blogs/aws/amazon-guardduty-now-supports-amazon-eks-runtime-monitoring/) 
 **EKS Audit Log Monitoring** : EKS Audit Log Monitoring helps you detect potentially suspicious activities in EKS clusters within Amazon Elastic Kubernetes Service (Amazon EKS). EKS Audit Log Monitoring uses Kubernetes audit logs to capture chronological activities from users, applications using the Kubernetes API, and the control plane. For more information, see  [Kubernetes audit logs](https://docs.aws.amazon.com/guardduty/latest/ug/features-kubernetes-protection.html#guardduty_k8s-audit-logs) 
 **EKS Runtime Monitoring** : EKS Runtime Monitoring uses operating system-level events to help you detect potential threats in Amazon EKS nodes and containers within your Amazon EKS clusters. For more information, see  [Runtime Monitoring ](https://docs.aws.amazon.com/guardduty/latest/ug/runtime-monitoring.html) 
![](https://miro.medium.com/v2/resize:fit:700/0*tNswcSmSZircvc80.jpg) 


# Enable Amazon GuardDuty for EKS:

Run the following command to enable Amazon GuardDuty and then also enable EKS Protection for both EKS Audit Log Monitoring and EKS Runtime Monitoring.
config.json

```
[
  {
    "Name": "EKS_AUDIT_LOGS",
    "Status": "ENABLED",      
    "Name": "EKS_RUNTIME_MONITORING",
    "Status": "ENABLED",
    "AdditionalConfiguration": [
      {
        "Name": "EKS_ADDON_MANAGEMENT",
        "Status": "ENABLED"
      }
    ]
  }
]
```


Run the below command to enable EKS Protection for Amazon GuardDuty.

```
aws guardduty create-detector --enable --features file://config.json | jq -r '.DetectorId')
```


After EKS Protection in Amazon GuardDuty is enabled, it looks like below in the  [AWS GuardDuty Console ](https://us-west-2.console.aws.amazon.com/guardduty/home?region=us-west-2#/k8s-protection) 
![](https://miro.medium.com/v2/resize:fit:700/0*_kfBEX3gcMce1_Oh.png) 
Go to Findings. You should see there are no findings available yet.
![](https://miro.medium.com/v2/resize:fit:700/0*HkXaKwfwL1sTY86Z.png) 
GuardDuty Findings are automatically sent to EventBridge. You can also export findings to an S3 bucket. New findings are exported within 5 minutes. You can modify the frequency for updated findings below. Update to EventBridge and S3 occurs every 6 hours by default. Let us change it to 15 mins.
Go to the  **Settings**  **Findings export options**  and Click on the Edit.
![](https://miro.medium.com/v2/resize:fit:700/0*7eKdsdBNC8bmUlIU.png) 
Select  **15 minutes**  and Click on  **Save Changes** 
![](https://miro.medium.com/v2/resize:fit:700/0*172w38VPk0uZH11h.png) 
With Amazon GuardDuty already turned on with protection for your EKS clusters, you are now ready to see it in action. GuardDuty for EKS does not require you to turn on or store EKS Control Plane logs. GuardDuty can look at the EKS cluster audit logs through direct integration.
It will look at the audit log activity and report on the new GuardDuty finding types that are specific to your Kubernetes resources.


# Conclusion:

In summary, Amazon GuardDuty delivers advanced security for Amazon EKS. With features like EKS Audit Log Monitoring and EKS Runtime Monitoring, it offers top-notch protection against potential threats. By integrating GuardDuty, you ensure continuous monitoring and quick mitigation of security risks, maintaining a secure environment for your container workloads.
 ** *Until next time 🇵🇸 🎉* ** 
![](https://miro.medium.com/v2/resize:fit:700/0*tsPIsScSuqH331R7) Photo by  [Patrick Perkins](https://unsplash.com/@patrickperkins?utm_source=medium&utm_medium=referral)  [Unsplash](https://unsplash.com/?utm_source=medium&utm_medium=referral) 
Thank you for Reading !! 🙌🏻😁📃, see you in the next blog.🤘🇵🇸
🚀 Thank you for sticking up till the end. If you have any questions/feedback regarding this blog feel free to connect with me :
♻️ 🇵🇸 **LinkedIn**  [https://www.linkedin.com/in/rajhi-saif/](https://www.linkedin.com/in/rajhi-saif/) 
♻️🇵🇸  **Twitter**  [https://twitter.com/rajhisaifeddine](https://twitter.com/rajhisaifeddine) 
 ** *The end * ** 


# 🔰 Keep Learning !! Keep Sharing !! 🔰



# References:
 [


## Amazon GuardDuty Now Supports Amazon EKS Runtime Monitoring | Amazon Web Services



### Since Amazon GuardDuty launched in 2017, GuardDuty has been capable of analyzing tens of billions of events per minute…

aws.amazon.com ](https://aws.amazon.com/blogs/aws/amazon-guardduty-now-supports-amazon-eks-runtime-monitoring/?source=post_page-----26036ce607de--------------------------------) [


## Amazon GuardDuty now protects Amazon Elastic Kubernetes Service clusters



### Posted On: Amazon GuardDuty has expanded coverage to continuously monitor and profile Amazon Elastic Kubernetes Service…

aws.amazon.com ](https://aws.amazon.com/about-aws/whats-new/2022/01/amazon-guardduty-elastic-kubernetes-service-clusters/?source=post_page-----26036ce607de--------------------------------) [


## Amazon EKS Security Immersion Day



### This workshop covers various security features on Amazon EKS. This will be used as part of a dedicated Immersion day…

catalog.workshops.aws ](https://catalog.workshops.aws/eks-security-immersionday/en-US/5-detective-controls/1-guardduty-protection-for-eks?source=post_page-----26036ce607de--------------------------------)