---
tags:
  - DOCKER
source: https://overcast.blog/building-efficient-multi-stage-dockerfiles-for-production-055f34c4baed
---




# Building Efficient Multi-Stage Dockerfiles for Production

![](https://miro.medium.com/v2/resize:fit:700/1*oIlXmqC3besQBIXjfGXVjw.png) 
Multi-stage builds in Docker use multiple  ``  statements within a single Dockerfile. This allows developers to construct multiple separate build stages with their own distinct base images. A common use case is to have one stage for building the application using a full-featured base image that includes build tools and dependencies, and another stage for running the application, using a stripped-down base image.
The key advantage here is the separation of build-time dependencies from the runtime environment. This separation ensures that the production image contains only the binaries and libraries necessary to run the application, excluding any unnecessary build tools or intermediate files.
This approach reduces the overall size of the final image and simplifies the Dockerfile structure, making it cleaner and more maintainable.


# Benefits of Multi-Stage Builds

1.  Reduced Image Size: By isolating build environments and artifacts, you can dramatically decrease the size of your final Docker images. Intermediate build stages can use larger, more complex base images that are never part of the final production image. For example, a Java application might use a Maven image to build the application but ultimately run it in a slim Java runtime environment, discarding all the Maven-specific layers.
2.  Enhanced Security: Smaller production images inherently contain fewer components, which minimizes the attack surface of your application. By excluding unnecessary tools and files, you reduce the number of ways an attacker can exploit the application container. This principle of minimalism in container design is fundamental to maintaining secure and robust Docker environments.
3.  Better Caching: Docker can cache the results of individual stages independently. If your build stage doesn’t change often but your application’s source code does, Docker can reuse the cached build stage and only rebuild the stages that have changed. This can significantly speed up the build process by avoiding unnecessary recomputation.



# When to Use Multi-Stage Builds

Multi-stage builds are particularly beneficial in the following scenarios:
- Continuous Integration/Continuous Deployment (CI/CD) Pipelines: When frequent builds are required, multi-stage builds can reduce build time and resource consumption. By leveraging Docker’s caching mechanism more effectively, you can speed up the build process in CI/CD pipelines, making it faster and more efficient.
- Production Deployments: For production environments, it is crucial to minimize the container image size to reduce deployment times and the surface area for potential security vulnerabilities. Multi-stage builds allow for the creation of minimalistic images that contain only the essentials needed to run the application, thereby optimizing deployment efficiency.
- Development Environments: While developing, you might need additional tools and dependencies that are not required in production. Multi-stage builds allow developers to create a lightweight production image while still using a more comprehensive environment for development and testing.
- Applications with Complex Build Environments: In cases where applications require complex build processes with many dependencies, multi-stage builds keep the final production image clean and free of unnecessary build tools. This is common in languages like C++ or Java, where the build environment significantly differs from the runtime environment.



# A Step-by-Step Tutorial to Implementing Multi-Stage Builds

Let’s dive into a practical example to showcase how multi-stage builds can be implemented effectively using Docker. This tutorial will guide you through creating an optimized Docker image for a Java application using Maven for building and a Java Runtime Environment (JRE) for execution. This approach minimizes the final image size, ensures security by limiting the attack surface, and leverages Docker’s caching mechanisms for faster build processes.


## Defining the Build Environment

First, you need to set up your build environment. Start by selecting an appropriate base image that includes all the necessary build tools and dependencies for compiling your Java application. Here, Maven is used as the build system, so you’ll start with a Maven image that also includes JDK 11. This stage is responsible for building your application from source.

```
# Stage 1: Build stage using Maven
FROM maven:3.6.3-jdk-11 as builder
WORKDIR /app
COPY . .
RUN mvn clean package
```


In the Dockerfile above,  `FROM maven:3.6.3-jdk-11 as builder`  initializes a new build stage and names it  `builder` . This name will be used to refer to the artifacts created in this stage later on. The  `WORKDIR /app`  directive sets the working directory to  ``  inside the container, where your application code will reside. The  `COPY . .`  command copies your local source code into the  ``  directory inside the container. Finally,  `RUN mvn clean package`  executes the Maven command to compile the application and package it into a JAR file.


## Compiling and Building the Application

This step is included in the first stage and involves compiling the Java code into bytecode and packaging it into an executable JAR file. The Maven base image contains all the necessary tools to compile the application, and this process results in a JAR file that is ready to be executed. This compiled artifact is what you will use in the next stage of the build.


## Preparing the Production Image

Now that you have your application built and packaged in the  `builder`  stage, you can create a lean production image. This next stage uses a lighter base image, specifically a JRE image, which is sufficient to run the Java application without the additional overhead of JDK or Maven.

```
# Stage 2: Production stage with JRE
FROM openjdk:11-jre-slim
COPY --from=builder /app/target/my-app.jar /app/my-app.jar
WORKDIR /app
CMD ["java", "-jar", "my-app.jar"]
```


The second stage starts with  `FROM openjdk:11-jre-slim` , indicating the use of a slim version of the OpenJDK 11 JRE. This base image contains only the runtime environment needed to run a Java application, significantly reducing the image size. The  `COPY --from=builder /app/target/my-app.jar /app/my-app.jar`  command copies the JAR file from the  `builder`  stage into the production image. The production image is finalized with the  `WORKDIR /app`  directive to set the working directory, and the  `CMD ["java", "-jar", "my-app.jar"]`  command, which tells Docker how to run the containerized application.


# Best Practices for Multi-Stage Builds

Incorporating best practices into your multi-stage builds not only enhances the efficiency of your Docker images but also optimizes their performance and manageability. Let’s delve into three crucial practices that can significantly improve your Docker build processes.


## Optimize Build Context

The build context is all the files and directories that Docker loads into the build process when you issue a build command. A large build context can significantly slow down the build process as more data needs to be sent to the Docker daemon. To optimize the build context, use a  `.dockerignore`  file to exclude files and directories that are not necessary for building the Docker image. This could include logs, local databases, and any code or data files not used in the actual application build.
For example, create a  `.dockerignore`  file in the same directory as your Dockerfile with the following contents to exclude typical unnecessary files:

```
**/.git
**/node_modules
**/npm-debug.log
logs/
tmp/
```


This configuration prevents your local git directories, node modules, logs, temporary files, and debug logs from being included in the Docker context, thereby speeding up the build process by handling only the necessary files.


## Use Specific Base Images

Selecting the right base image is crucial for the efficiency of your Docker images. Use specific base images that provide just the runtime environment your application needs. For instance, if you’re deploying a Python application, instead of using a generic Python image, you might choose a slim or alpine version that includes only the essential packages and libraries required to run your application.
Here’s how you might set up a Python application using a slim base image:

```
# Use an official Python runtime as a parent image
FROM python:3.8-slim

# Set the working directory to /app
WORKDIR /app
# Copy the current directory contents into the container at /app
COPY . .
# Install any needed packages specified in requirements.txt
RUN pip install --trusted-host pypi.python.org -r requirements.txt
# Make port 80 available to the world outside this container
EXPOSE 80
# Run app.py when the container launches
CMD ["python", "app.py"]
```


This Dockerfile uses  `python:3.8-slim`  as the base image, which is significantly smaller than the full Python image, reducing the overall image size and thus improving the deployment speed.


## Minimize Layers

Docker images are built up from layers; each instruction in a Dockerfile adds a new layer. Reducing the number of layers in your Dockerfile can lead to smaller image sizes and faster pull times. To minimize layers, combine Dockerfile instructions wherever possible. For example, instead of having separate  ``  instructions for each package installation step, you can chain these commands together into a single  ``  instruction:

```
# Incorrect: Multiple RUN instructions creating multiple layers
RUN apt-get update
RUN apt-get install -y package1
RUN apt-get install -y package2

# Correct: A single RUN instruction that combines multiple steps
RUN apt-get update && \
    apt-get install -y package1 package2 && \
    rm -rf /var/lib/apt/lists/*
```


This approach reduces the number of layers by combining update and install steps into a single layer and cleans up afterward by removing unnecessary files, which also helps to keep the image size down.


# Conclusion

By implementing multi-stage builds, you can streamline your Dockerfiles for production, ensuring they are efficient, secure, and maintainable. This approach is not just about optimizing your Docker environment but also about embracing best practices that will make your deployments smoother and more reliable.


# Resources

- Docker Official Documentation on Multi-Stage Builds:  [https://docs.docker.com/develop/develop-images/multistage-build/](https://docs.docker.com/develop/develop-images/multistage-build/) 
- Best Practices for Writing Dockerfiles:  [https://docs.docker.com/develop/develop-images/dockerfile_best-practices/](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/) 
- Efficient Docker Images with Multi-Stage Builds:  [https://www.docker.com/blog/further-examples-of-docker-multi-stage-builds/](https://www.docker.com/blog/further-examples-of-docker-multi-stage-builds/) 



# Learn more
 [


## 13 Docker Tricks You Didn’t Know



### Docker has become an indispensable tool in the development, testing, and deployment pipelines of modern applications…

overcast.blog ](https://overcast.blog/13-docker-tricks-you-didnt-know-47775a4f678f?source=post_page-----055f34c4baed--------------------------------) [


## 13 Ways to Troubleshoot Docker Faster in 2024



### Efficient troubleshooting is vital to minimize downtime, improve system reliability, and enhance overall performance…

overcast.blog ](https://overcast.blog/13-ways-to-troubleshoot-docker-faster-in-2024-5b064c20c9e2?source=post_page-----055f34c4baed--------------------------------) [


## 13 Kubernetes Configurations You Should Know in 2024



### As Kubernetes continues to be the cornerstone of container orchestration, mastering its configurations and features…

overcast.blog ](https://overcast.blog/13-kubernetes-configurations-you-should-know-in-2024-54eec72f307e?source=post_page-----055f34c4baed--------------------------------)