---
tags:
  - DOCKER
source: https://blog.devgenius.io/docker-image-size-container-security-371e12c9e461
---




# DOCKER | Image Size & Container Security

![](https://miro.medium.com/v2/resize:fit:700/0*D4bE9asNPgQcAlbx.jpg) 
Welcome to our latest blog post on Docker! Here, we will explore the best practices for Docker image size and container security. Docker has changed the way applications are deployed. However, for optimal performance and protection against vulnerabilities, it’s crucial to use efficient image management and robust security practices.
In this blog post, we’ll take a closer look at different strategies for managing Docker images, including reducing image size and improving performance. We’ll also examine the best security practices for Docker containers, such as using secure base images, limiting container privileges, and implementing network security controls. By following these best practices, you can ensure that your Docker containers are secure, efficient, and optimised for performance. Let’s dive right in and learn how to make the most of Docker’s powerful capabilities!


# Docker Image Size & Security



## ➡️ Check out our YouTube video for a complete guide with real-life examples! 🤩



## ☑️ 1. Usage of Official & Verified Docker Images

When creating Docker containers, it’s important to use official images and specify the exact version of the base image used. Doing so will help you avoid security issues, reduce the risk of system failures, and prevent unexpected changes or inconsistencies.
To use official images, you can include the name of the image in your Dockerfile, and Docker will automatically pull it from the official registry. This best practice will save you time in the long run and help you create more secure and reliable images.
To specify the exact version of the base image, include it in your Dockerfile, like in the example below:

```
# Use an official Python runtime as the base image
FROM python:3.9-slim
```


But we will update it to this:

```
# Use a specific version of the base image
FROM python:3.9.10-slim
```


This will prevent any unwanted changes and inconsistencies that may arise from using different versions of the same image.


## 🖼️ 2. Optimising Caching Image Layers

Efficient Docker images can be created by optimising caching image layers and using a .dockerignore file to exclude unnecessary files.
1.   **To optimise caching image layers** , we can divide the image into smaller parts that can be cached individually. This way, when changes are made to the code, only the modified parts need to be rebuilt. This not only saves time but also reduces the final image size. Here’s how we can achieve this:


```
# Copy only necessary files
COPY requirements.txt /app/

# Install dependencies
RUN pip install --no-cache-dir -r requirements.txt
```


☝️When using Docker, it’s essential to be mindful that every RUN command creates a new layer in the image, which can significantly increase the overall image size.
 **To minimise the number of layers and reduce the final size, it’s advisable to combine multiple commands into a single RUN statement using “&&” like in the example below:** 

```
FROM alpine:latest

# Update package lists, install dependencies, and clean up
RUN apk update && \
 apk add --no-cache \
 package1 \
 package2 \
 && rm -rf /var/cache/apk/*
```


2. It’s useful to use a . **dockerignore**  file when building Docker images. This file allows us to exclude unnecessary files from the image, such as log files, configuration files, and temporary files. These files aren’t required for the application to function, so excluding them will make the image smaller and speed up the build process. Below is an example of what a .dockerignore file might look like:

```
# Ignore files and directories commonly unnecessary in Docker images

# Files
.git
.vscode
*.log

# Directories
node_modules
__pycache__
```




## 📑 3. Multi-Stage Deployments



## ➡️ Check out our YouTube video for a complete guide with real-life examples! 🤩



# Container Security



## 🧑‍💼 Least Privilege

To ensure the security of your applications, it is recommended to  **avoid using the root user account**  when running your containers.
Instead, create a  **separate user account**  solely for your application and utilize it to execute your program. This simple measure can significantly reduce the chances of unauthorised access to your system, as well as prevent potential malicious attacks.
Furthermore, using a non-root user account can help contain the damage in case your application is ever compromised. Remember to always use a non-root user account to ensure the safety and security of your applications.

```
# Use a small Python base image
FROM python:3.9-slim

# Set up a non-root user
RUN addgroup -g 1000 hitcuser && \
 adduser -u 1000 -G hitcuser -D hitcuser

# Set the working directory in the container
WORKDIR /app

# Copy the requirements file into the container
COPY requirements.txt .

# Install dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Set up any additional build steps here, if needed

# Change the ownership of the application directory to the non-root user
RUN chown -R hitcuser:hitcuser /app

# Switch to the non-root user
USER hitcuser
```


In our Dockerfile, we have created a non-root user named ‘hitcuser’ using the ‘useradd’ command and then switched to that user using ‘USER hitcuser’. Following standard practices for building and running a Python application, we have included copying files, etc. By implementing this approach, you can significantly reduce the attack surface of your containerised environment and mitigate the risk of potential security breaches.
In simpler terms, it is recommended to avoid running containers with root privileges and instead use a non-root user with only the necessary permissions to operate the application. This helps limit the damage caused by an attacker who gains access to the container, as they will only have access to the minimum set of resources needed for the application to function.


## 🔍 Image Scan

Let’s create a demo to scan Docker images using Trivy, a popular open-source scanner for container vulnerabilities.
First, ensure you have Trivy installed on your system. You can follow the installation instructions provided on the Trivy GitHub repository:  [https://github.com/aquasecurity/trivy#installation](https://github.com/aquasecurity/trivy#installation) 
Once Trivy is installed, you can use it to scan Docker images for vulnerabilities. Here’s a step-by-step demo:
Pull a Docker Image: For this demo, let’s use a Python Docker image as an example. You can pull the official Python image from Docker Hub.

```
docker pull python:3.9-slim
```


Scan the Docker Image with Trivy: Run Trivy to scan the pulled Docker image for vulnerabilities.

```
trivy image python:3.9-slim
```


Trivy will analyze the image layers and report any vulnerabilities found, along with their severity levels.
 **Review Scan Results: ** Examine the scan results provided by Trivy. The output will include information about each vulnerability detected, such as CVE IDs, severity levels, package names, and descriptions.

```
== Server ==
Server: Debian GNU/Linux 11 (bullseye)
== Trivy ==
Severity: CRITICAL
Description: glibc (GNU C Library) is prone to a stack-based buffer overflow when making function calls with a misaligned address. An attacker can leverage this vulnerability to execute code in the context of the affected application, resulting in a denial-of-service condition or the execution of arbitrary code.
References:
- https://security-tracker.debian.org/tracker/CVE-2017-1000366
…
```


 **Take Action: ** Performing regular image scans with Trivy or other similar tools can help you identify and address security vulnerabilities in Docker images proactively. Based on the scan results, you should take appropriate actions to remediate vulnerabilities, such as updating packages, applying patches, or using alternative images with fewer vulnerabilities.
 **Automate Scan: ** To automate vulnerability detection for Docker images, you can integrate Trivy scanning into your CI/CD pipeline using either its CLI or client libraries. By doing so, you can incorporate scanning into your existing automation workflows and effectively secure your containerized applications.


## ➡️ For a complete guide on Container Security, go to our YouTube video specifically created for this!!! 🤩



# Closing Thoughts

![](https://miro.medium.com/v2/resize:fit:700/0*LyafYATxbxVhxWsy) 
We have reached the end of our extensive discussion on Docker image size best practices and container security. In our upcoming video, we will cover a highly requested topic from our community — Docker Networking!
Also, don’t forget to hit the like button and subscribe to our  [ **YouTube channel **  ](https://www.youtube.com/@headintheclouds.online)to stay updated on our latest content 🔥 We regularly publish new videos on Docker, Kubernetes, DevOps, Cloud Computing, Web Development and other related topics.


## By subscribing to our channel, you will never miss a new release and you will have access to exclusive content! 🚀

Thank you for watching, and we look forward to seeing you in our upcoming videos! 🎬🤗