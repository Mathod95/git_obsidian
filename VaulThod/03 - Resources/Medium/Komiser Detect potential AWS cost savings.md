---
source: https://medium.com/komiser/komiser-optimize-cost-and-security-on-aws-183174ba58c5
tags:
  - APP/KOMISER
---




# Komiser: Detect potential AWS cost savings

![](https://miro.medium.com/v2/resize:fit:1000/1*ltmR08aYBacJFng6y5zf7w.png)  [Komiser](https://github.com/mlabouardy/komiser) : Cloud Environment Inspector
Over the last decade, the cost of Amazon Web Services (AWS) has become a primary concern of businesses. That’s no surprise: AWS has many services that offer a range of IT resources — from IT infrastructure and bandwidth to analytics tools and machine learning — and each affects the total cloud bill.
While AWS offers many fully-managed services like CloudWatch, CloudTrail, Trusted Advisor, etc to help you detect potential cost savings. Understanding and managing cloud costs isn’t simple with AWS.
That’s why, I came up one year ago, with an open source tool called  [Komiser](https://github.com/mlabouardy/komiser)  to help reduce your AWS infrastructure cost based on custom recommendations. [


## Komiser: AWS Environment Inspector



### In order to build HA & Resilient applications in AWS, you need to assume that everything will fail. Therefore, you…

hackernoon.com ](https://hackernoon.com/komiser-aws-environment-inspector-8340946b6237?source=post_page-----183174ba58c5--------------------------------)
After 1 year of intense development, I’m thrilled to announce the fresh new release of  [Komiser: 2 *.0.0 *  ](https://github.com/mlabouardy/komiser)with support of new AWS services:
![](https://miro.medium.com/v2/resize:fit:700/1*ltmR08aYBacJFng6y5zf7w.png) AWS Services supported by Komiser


#  **Highlights** 

![](https://miro.medium.com/v2/1*m5kMXf3djJecXVy0i7LR1Q.png) 
With the  [GDPR](https://en.wikipedia.org/wiki/General_Data_Protection_Regulation)  becoming real in EU, logging and storage of (potentially) personally identifiable information now need to be reduced in many organizations.  [Komiser](https://github.com/mlabouardy/komiser)  allows you to analyze and manage cloud cost, usage, security, and governance in one place. Hence, detecting potential vulnerabilities that could put your cloud environment at risk.
It allows you also to control your usage and create visibility across all used services to achieve maximum cost-effectiveness and get a deep understanding of how you spend on the AWS, GCP and Azure.


# Usage

Below are the available downloads for the latest version of  [Komiser](https://github.com/mlabouardy/komiser)  (2.0.0). Please download the proper package for your operating system and architecture.


## Linux:


```
wget https://cli.komiser.io/2.0.0/linux/komiser
```




## Windows:


```
wget https://cli.komiser.io/2.0.0/windows/komiser
```




## Mac OS X:


```
wget https://cli.komiser.io/2.0.0/osx/komiser
```


> 
 ** : make sure to add the execution permission to Komiser  `chmod +x komiser`  and update the user’s  *$PATH*  variable.

 [Komiser](https://github.com/mlabouardy/komiser)  is also available as a Docker image:


## Docker:


```
docker run -d -p 3000:3000 --name komiser mlabouardy/komiser:2.0.0
```


If you point your favorite browser to  [http://localhost:3000](http://localhost:3000/) , you should see  **Komiser ** awesome dashboard:
![](https://miro.medium.com/v2/resize:fit:700/0*Cp9DA9QDq_ZIY30L) 
> 
 *versioned*  documentation can be found on  [https://docs.komiser.io](https://docs.komiser.io/) 

 [Komiser](https://github.com/mlabouardy/komiser)  is written in Golang and is MIT licensed — contributions are welcomed whether that means providing feedback or testing existing and new features.
![](https://miro.medium.com/v2/resize:fit:700/1*IvqQ9IzItF8y2xiF_7CVGw.png)  [https://komiser.io](https://komiser.io/) 
Drop your comments, feedback, or suggestions below — or connect with me directly on Twitter  [ **** mlabouardy ](https://twitter.com/mlabouardy)