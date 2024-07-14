---
created: 12 mai 2024, 19:52
tags:
  - DOCKERFILE
source: https://medium.com/@tonistiigi/new-dockerfile-capabilities-in-v1-7-0-be6873650741
---




# New Dockerfile capabilities in v1.7.0

We recently released the new versions of the  [BuildKit builder toolkit](https://github.com/moby/buildkit/releases/tag/v0.13.1)  [Docker Buildx CLI](https://github.com/docker/buildx/releases/tag/v0.13.1) , and  [Dockerfile frontend for BuildKit (v1.7.0)](https://hub.docker.com/r/docker/dockerfile) . In this blog post, I’ll go over some of the new Dockerfile capabilities and explain how you can start using them in your projects.
![](https://miro.medium.com/v2/resize:fit:6456/1*A4D403GqBkvpxqxR8JZI8g.png) 


# Versioning

First, let’s have a quick reminder of how Dockerfile is versioned and what you should do to update to  **v1.7.0** 
While most projects use Dockerfiles to build images, BuildKit is not limited only to that format. BuildKit supports multiple different frontends for defining the build steps for BuildKit to process. Anyone can create these frontends, package them as regular container images, and load them from a registry when you invoke the build.
With the new release, we have published two such images to  [Docker Hub](https://hub.docker.com/r/docker/dockerfile)  `docker/dockerfile:1.7.0`  and  `docker/dockerfile:1.7.0-labs` 
To use these frontends, you need to specify a  `#syntax`  directive at the beginning of the file to tell BuildKit which frontend image to use for the build. Here we have set it to use the latest of the  `1.7.x`  major version.
For example:

```
#syntax=docker/dockerfile:1.7
FROM alpine
…
```


This means that BuildKit is decoupled from the Dockerfile frontend syntax. You can start using new Dockerfile features right away without having to worry about which BuildKit version you’re using.  **All the examples described in this post will work with any version of Docker that supports BuildKit (**  [ **The default builder as of Docker Engine v23**  ](https://docs.docker.com/build/buildkit/#getting-started) **), as long as you define the correct **  ` **#syntax** `  ** directive on the top of your Dockerfile.** 
You can learn more about Dockerfile frontend versions from  [Docker docs](https://docs.docker.com/build/dockerfile/frontend/) 


# Variable expansions

When you write Dockerfiles, build steps can contain variables defined using the build arguments ( `` ) and environment variables ( `` ) instructions. The difference between build arguments and environment variables is that environment variables are kept in the resulting image and persist when a container is created from it.
When you use such variables, you most likely use  `${NAME}`  or, more simplified,  `$NAME`  ``  `` , and other commands.
You might not know that Dockerfile also supports two forms of bash-like variable expansion:
-  `${variable:-word}`  sets a value to word if the variable is  **unset** 
-  `${variable:+word}`  sets a value to word if the variable is  **** 

Up to this point, these special forms were not that useful because the default value of  ``  instructions can also be set directly.

```
FROM alpine
ARG foo="default value"
```


If you are an expert in various shell applications, you know that Bash and other tools usually have many additional forms of variable expansion to ease the development of your scripts.
In Dockerfile v1.7, we have added:
-  `${variable#pattern}`  and  `${variable##pattern}`  to remove the shortest or longest prefix from the start of the variable’s value.
-  `${variable%pattern}`  and  `${variable%%pattern}`  to remove the shortest or longest suffix from the end of the variable’s value.
-  `${variable/pattern/replacement} ` to first replace occurrence of a pattern
-  `${variable//pattern/replacement}`  to replace all occurrences of a pattern

From just looking at the new rules, what you would use them for might not be completely obvious. So, let’s look at some examples we have seen in actual Dockerfiles.
Let’s start with some simple examples.
Projects still can’t agree on whether versions for downloading your dependencies should have a  **  prefix or not. The following allows you to get the format you need:

```
# example VERSION=v1.2.3
ARG VERSION=${VERSION#v}
# VERSION is now '1.2.3'
```


In the next case, multiple variants are used by the same project:

```
ARG VERSION=v1.7.15
ADD https://github.com/containerd/containerd/releases/download/${VERSION}/containerd-${VERSION#v}-linux-amd64.tar.gz /
```


To configure different command behaviors for multi-platform builds, BuildKit provides useful built-in variables like  `TARGETOS`  and  `TARGETARCH` . Unfortunately, not all projects use the same values. For example, while in containers and in the Go ecosystem, we refer to 64-bit ARM architecture as  `arm64`  , sometimes you need  `aarch64`  instead.

```
ADD https://.../download/bun-v1.0.30/bun-linux-${TARGETARCH/arm64/aarch64}.zip /
```


In this case, the URL also uses a custom name for AMD64 architecture. To pass a variable through multiple expansions, use another  ``  definition with an expansion from the previous value. You could also write all the definitions on a single line, as  ``  allows multiple parameters, but that may hurt readability.

```
ARG ARCH=${TARGETARCH/arm64/aarch64}
ARG ARCH=${ARCH/amd64/x64}
ADD https://.../download/bun-v1.0.30/bun-linux-${ARCH}.zip /
```


Note that the example above is written in a way that if a user passes their own  `--build-arg ARCH=value` , then that value is used as-is.
Now, let’s look at how new expansions can be useful in multi-stage builds.
One of the techniques described in  [“Advanced multi-stage build patterns”](https://medium.com/@tonistiigi/advanced-multi-stage-build-patterns-6f741b852fae)  shows how build arguments can be used so that different Dockerfile commands run depending on the build-arg value. For example, you can use that pattern if you build a multi-platform image and wish to run additional  ``  ``  commands only for specific platforms.
If this method is new to you, you can learn more about it from the original post. In summarized form, the idea is to define a global build argument and then define build stages that use the build argument value in the stage name while pointing to the base of your target stage via the build-arg name.
Old example:

```
ARG BUILD_VERSION=1

FROM alpine AS base
RUN …

FROM base AS branch-version-1
RUN touch version1

FROM base AS branch-version-2
RUN touch version2

FROM branch-version-${BUILD_VERSION} AS after-condition

FROM after-condition
RUN …
```


When using this pattern for multi-platform builds, one of the limitations is that all the possible values for the build-arg need to be defined by your Dockerfile. This is problematic as we want Dockerfile to be built in a way that it can build on any platform and not limit it to a specific set.  [Here](https://github.com/moby/buildkit/blob/v0.13.0/Dockerfile#L31-L37)  [are](https://github.com/moby/moby/blob/a8abb67c5ee2ee2e92a28caf0dd6bfeded30a0e9/Dockerfile#L158-L167)  some examples of Dockerfiles where dummy stage aliases must be defined for all architectures, and no other architecture can be built. Instead, the pattern we would like to use is that there is one architecture that has a special behavior, and everything else shares another common behavior.
With new expansions, we can write this to demonstrate running special commands only on RISC-V, which is still somewhat new and may need custom behavior:

```
#syntax=docker/dockerfile:1.7

ARG ARCH=${TARGETARCH#riscv64}
ARG ARCH=${ARCH:+"common"}
ARG ARCH=${ARCH:-$TARGETARCH}

FROM --platform=$BUILDPLATFORM alpine AS base-common
ARG TARGETARCH
RUN echo "Common build, I am $TARGETARCH" > /out

FROM --platform=$BUILDPLATFORM alpine AS base-riscv64
ARG TARGETARCH
RUN echo "Riscv only special build, I am $TARGETARCH" > /out

FROM base-${ARCH} AS base
```


Let’s go over these  ``  definitions.
- The first sets  ``  `TARGETARCH`  but removes  *“riscv64”*  from the value.
- Next, as we described earlier, we don’t actually want the other architectures to use their own values but want them all to share a common value. So we set  ``  *“common”*  except if it was cleared from the previous  *riscv64*  rule.
- Now, if we still have an empty value, we default it back to  `$TARGETARCH` 
- The last definition is optional, as we would already have a unique value for both cases, but it makes the final stage name  `base-riscv64`  nicer to read.

Additional examples of including multiple conditions with shared conditions, or conditions based on architecture variants can be found  [here](https://gist.github.com/tonistiigi/024172617d8178e4f5a7665120397264) 
Comparing this to the initial example of conditions between stages, the new pattern isn’t limited to just controlling the platform differences of your builds but can be used with any build-arg. If you used this pattern before, then you can effectively now define an  *“else”*  clause, while previously, you were limited to only  **  clauses.


# Copy with keeping parent directories.

> 
The following feature has been released in the “ [labs](https://docs.docker.com/build/dockerfile/frontend/#labs-channel) ” channel. Define the following at the top of your Dockerfile to use this feature.
 `#syntax=docker/dockerfile:1.7-labs` 

When you are copying files in your Dockerfile, for example:

```
COPY app/file /to/dest/dir/
```


It means that the source file is copied directly to the destination directory. If your source path was a directory, then all the files inside that directory are copied directly to the destination path.
But what if you have a file structure such as:

```
.
├── app1
│ ├── docs
│ │ └── manual.md
│ └── src
│   └── server.go
└── app2
  └── src
    └── client.go
```


You wish to copy only files in  `app1/src` , but so that the final files at the destination would be  `/to/dest/dir/app1/src/server.go`  and not just  `/to/dest/dir/server.go` 
With the new  `COPY --parents`  flag, you can write:

```
COPY --parents /app1/src/ /to/dest/dir/
```


This will copy the files inside the src directory and recreate the  `app1/src`  directory structure for these files.
Things get more powerful when you start to use wildcard paths. To copy the src directories for both apps into their respective locations, you can write:

```
COPY --parents */src/ /to/dest/dir/
```


This will create both  `/to/dest/dir/app1`  and  `/to/dest/dir/app2` , but it will not copy the  *“docs”*  directory. Previously, this kind of copy was not possible with a single command. You would have needed multiple copies for individual files ( [like here](https://github.com/moby/moby/issues/39530#issuecomment-531516250) ) or used some workaround with the  `RUN --mount`  instruction instead.
You can also use double-star wildcard ( ** ) to match files under any directory structure. For example, to copy only the Go source code files anywhere in your build context, you can write:

```
COPY --parents **/*.go /to/dest/dir/
```


If you are thinking about why you would need to copy specific files instead of just using  `COPY ./`  to copy all files, remember that your build cache gets invalidated when you include new files in your build. If you copy all files, the cache gets invalidated when any file is added or changed, while if you copy only Go files, then only changes in these files influence the cache.
The new  `--parents`  flag is not only for  ``  instructions from your build context, but obviously, you can also use them in multi-stage builds when copying files between stages using  `COPY --from` . One thing to note is that with  `COPY --from`  syntax, all source paths are expected to be absolute, meaning that if the  `--parents`  flag is used with such paths, they will be fully replicated as they were in the source stage. That may not always be desirable, and instead, you may wish to keep some of the parents but discard and replace others. In that case, you can use a special  ``  relative pivot point in your source path to mark which parents you wish to copy and which should be ignored. This special path component resembles how  `rsync`  works with the  `--relative`  flag.

```
#syntax=docker/dockerfile:1.7-labs

FROM … AS base
RUN ./generate-lot-of-files -o /out/
# /out/usr/bin/foo
# /out/usr/lib/bar.so
# /out/usr/local/bin/baz

FROM scratch
COPY --from=base --parents /out/./**/bin/ /
# /usr/bin/foo
# /usr/local/bin/baz
```


The above example shows how only  *“bin”*  directories are copied from the collection of files that the intermediate stage generated, but all the directories will keep their paths relative to the  *“out”*  directory.


# Exclusion filters

> 
The following feature has been released in the “ [labs](https://docs.docker.com/build/dockerfile/frontend/#labs-channel) ” channel. Define the following at the top of your Dockerfile to use this feature.
 `#syntax=docker/dockerfile:1.7-labs` 

Another related case when moving files in your Dockerfile with  ``  and  ``  instructions is when you want to move a group of files, but exclude a specific subset. Previously, your only options were to use  `RUN --mount`  or try to define your excluded files inside a  `.dockerignore`  file.
However,  `.dockerignore`  files are not a good solution for this problem as they only list the files excluded from the client-side build context and not from builds from remote Git/HTTP URLs and are limited to  [one per Dockerfile](https://docs.docker.com/build/building/context/#dockerignore-files) . You should use them similarly to  `.gitignore`  to mark files that are never part of your project, but not as a way to define your application-specific build logic.
With the new  `--exclude=[pattern]`  flag, you can now define such exclusion filters for your  ``  and  ``  commands directly in the Dockerfile. The pattern uses the same format as  `.dockerignore`  files.
The following example copies all the files in a directory except Markdown files:

```
COPY --exclude=*.md app /dest/
```


You can use the flag multiple times to add multiple filters. The next example excludes Markdown files and also a file called  `README` 

```
COPY --exclude=*.md --exclude=README app /dest/
```


Double-star wildcards exclude not only Markdown files in the copied directory but also in any subdirectory.

```
COPY --exclude=**/*.md app /dest/
```


As in  `.dockerignore`  files, you can also define exceptions to the exclusions with  ``  prefix. The following excludes all Markdown files in any copied directory, except if the file is called  *“important.md”*  — in that case, it is still copied.

```
COPY --exclude=**/*.md --exclude=!**/important.md app /dest/
```


This double negative may be confusing initially, but note that this is a reversal of the previous exclude rule, and include patterns are defined by the source parameter of the  ``  instruction.
When using  `--exclude`  together with the previously described  `--parents`  copy mode, note that the exclude patterns are relative to the copied parent directories or to the pivot point  ``  if one is defined.
For example, with a directory structure like:

```
assets
├── app1
│ ├── icons32x32
│ ├── icons64x64
│ ├── notes
│ └── backup
├── app2
│ └── icons32x32
└── testapp
  └── icons32x32
```



```
COPY --parents --exclude=testapp assets/./**/icons* /dest/
```


This would create the directory structure below. Note that only directories with the icons prefix were copied, the root parent directory  *“assets”*  was skipped as it was before the relative pivot point, and additionally, “ *testapp”*  was not copied as it was defined with an exclusion filter.

```
dest
├── app1
│ ├── icons32x32
│ └── icons64x64
└── app2
  └── icons32x32
```




# Summary

Hopefully, this post gave you some ideas for improving your Dockerfiles, and some of the patterns shown here help you describe your build more efficiently. Remember that your Dockerfile can start using all these features today by just defining the  `#syntax`  line on top, even if you haven’t updated to the latest Docker yet.
If you have any issues or have suggestions you want to share, then let us know in  [the issue tracker](https://github.com/moby/buildkit) 
For a full list of other features in the new Buildkit, Buildx, and Dockerfile releases, check out the changelogs:
 [BuildKit v0.13.0](https://github.com/moby/buildkit/releases/tag/v0.13.0) 
 [Buildx v0.13.0](https://github.com/docker/buildx/releases/tag/v0.13.0) 
 [Dockerfile v1.7.0](https://github.com/moby/buildkit/releases/tag/dockerfile%2F1.7.0)  [v1.7.0-labs](https://github.com/moby/buildkit/releases/tag/dockerfile%2F1.7.0-labs) 
Thanks to community members  [@tstenner](https://github.com/tstenner)  [@DYefimov](https://github.com/DYefimov) , and  [@leandrosansilva](https://github.com/leandrosansilva)  for helping to implement these features!
 *Originally posted to *  [ *https://www.docker.com/blog/new-dockerfile-capabilities-v1-7-0/*  ](https://www.docker.com/blog/new-dockerfile-capabilities-v1-7-0/) ** 