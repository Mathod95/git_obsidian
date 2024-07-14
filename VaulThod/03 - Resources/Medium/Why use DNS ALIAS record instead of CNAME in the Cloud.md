---
tags:
  - AWS/ROUTE53
  - AWS/LOADBALANCER
source: https://adil.medium.com/why-use-dns-alias-record-instead-of-cname-in-the-cloud-ca995b7a364d
---




# Why use DNS ALIAS record instead of CNAME in the Cloud?

Having a custom domain on top of a cloud resource is essential.
![](https://miro.medium.com/v2/resize:fit:700/0*jFESz-2oT_BmdxD7) Photo by  [Joanna Kosinska](https://unsplash.com/@joannakosinska?utm_source=medium&utm_medium=referral)  [Unsplash](https://unsplash.com/?utm_source=medium&utm_medium=referral) 
Cloud providers create a resource (such as a load balancer), and provide you with a domain, such as: \
 `some-production-loadbalancer-123456.eu-west-1.elb.amazonaws.com` 
You add this domain as a CNAME record to  `app.yourcompany.com` 


#  **What happens if you delete the resource on the Cloud?** 

If you remove the resource but leave the CNAME record on the custom domain ( `app.yourcompany.com` ), you will encounter the Dangling CNAME problem.


#  **What is the Dangling CNAME issue?** 

Let’s have a look at an actual example;
I created a domain:  `blog.example.adililhan.com` \
I created a load balancer on AWS: \
 `adil-blog-test-1393565408.eu-west-1.elb.amazonaws.com` 
![](https://miro.medium.com/v2/resize:fit:700/1*bAf6XT-Qt0F6cofClx0g1w.png) 
I added the load balancer’s domain name as a CNAME to my domain:
![](https://miro.medium.com/v2/resize:fit:700/1*HjqMdxQVmZQaDT4iK7xR-g.png) 
Just for testing, I will send a request:
![](https://miro.medium.com/v2/resize:fit:634/1*UVPvzeYjjOqtxtnvVjMztw.png) 
I received a response from the AWS resource (an EC2 instance).
I will delete the load balancer and see what happens:
![](https://miro.medium.com/v2/resize:fit:700/1*8Q8gHgp9riB_hpuUVNhtyw.png) 
DNS Result:
![](https://miro.medium.com/v2/resize:fit:700/1*cGysWsSYEeoemq5lBHGGTA.png) 
AWS has no longer returned anything for the load balancer domain since it was deleted.
I will create a new load balancer with the same name:  `adil-blog-test` 
![](https://miro.medium.com/v2/resize:fit:700/1*GHCbbdaid-IptCU8YQ9FOA.png) 
This is the new load balancer’s domain name:
 `adil-blog-test-1109044799.eu-west-1.elb.amazonaws.com` 
It is quite similar to the previous one. Only the numbers are different.


#  **This is the problem: Random numbers** 

AWS uses the following naming convention to establish domains for its load balancers:  `$YourLoadBalancerName-$RandomNumber.$RegionName.elb.amazonaws.com` 
Attackers may identify 3 variables in your DNS output:

```
$YourLoadBalancerName -> adil-blog-test
$RandomNumber -> 1109044799
$RegionNumber -> eu-west-1
```


So, the attacker cannot control just one variable:  **$RandomNumber** 
I removed the load balancer, but maintained its domain name in my custom domain (  `blog.example.adililhan.com` 
So, if the attacker deletes/creates a sufficient number of load balancers, the attacker will eventually get the same number of removed one.


#  **Solution: Use DNS Alias Records** 

DNS has a lot of distinct record types: A, AAAA, CNAME, MX, etc.
The ALIAS record type is not one of the standard record types.
Only a few service providers, such as  ****  **CloudFlare** , and  **Azure** , support the  **DNS ALIAS**  record type.
> 
The ALIAS record type is comparable to CNAME. You provide the load balancer’s domain name in the service provider configuration.
The service provider will check the returning IP addresses from the load balancer’s domain address and update your domain’s A records accordingly.
If the load balancer’s IP address changes, your custom domain’s A records are immediately updated.

Let’s add the load balancer’s domain name as an ALIAS record.
![](https://miro.medium.com/v2/resize:fit:700/1*YJzVt6fKiG_B9BjOycyWBA.png) 
Check the DNS:
![](https://miro.medium.com/v2/resize:fit:700/1*d1wzjN10FLEjD_1j-BNemQ.png) 
As a result, the attacker cannot determine the load balancer's domain name.
For testing purposes, I will send a request:
![](https://miro.medium.com/v2/resize:fit:700/1*OnYQ7e-Q3kbe6w1BxvWlgA.png) 
I deleted the load balancer:
![](https://miro.medium.com/v2/resize:fit:700/1*5h3jWTxjtV0dS-OtTLFMbg.png) 
Upon detecting the deletion of the load balancer, AWS proceeded to remove the IP address associated with the load balancer from my custom domain:
![](https://miro.medium.com/v2/resize:fit:700/1*3wPVvnfgHyPUQg9csA2UTg.png) 
The ALIAS record eliminates the need to monitor CNAME entries on your domains.
This technique is also known as DNS Flattening.