---
layout: post
title:  "Docker DevSecOps with Trivy"
date:   2021-10-27 10:00:00 -0500
categories: blog
---

Recently at work our team introduced docker vulnerability scanning into our DevOps pipelines. This 
allows our team to quickly review any patchable vulnerabilities in our containers and prevents 
us from deploying known vulnerabilities into our public environments. We accomplished this through the 
use of a free container scanning tool called [Trivy][trivy-link].
<!--more-->

We used Trivy because it is free to integrate, maintains an up to date database of Common 
Vulnerabilities and Exposures (CVEs), and provides Docker images to allow quick and easy addition 
into our automation pipelines. In this post, my goal is to explain a little about the importance of 
scanning built containers for vulnerabilities, give a brief introduction to Trivy, how our team 
included this into our automation pipelines, and how we patch base image vulnerabilities.

## Importance of Docker Scanning

I'm not going to spend too much time documenting the importance of security in development of 
applications and their deployment mechanisms. Liran Tal over at Snyk (another great option for 
vulnerability scanning - even built directly into the Docker CLI) has a great article written 
on hacking a vulnerable and official Node image. You can see his article [here][vuln-link].

## Trivy Introduction

[Trivy][trivy-link] was found by our team to be the cheapest and quickest way to get vulnerability 
scanning into our automation pipeline when we started using Docker to containerize our applications. 
Using Trivy can also be done using Docker so no extra tool installs are needed. However, you can 
install Trivy locally if you wish - [here][trivy-install] are the instructions. From the command line, 
running a container scan is as simple as

`docker run --rm aquasec/trivy image mcr.microsoft.com/azure-functions/dotnet:4`

This command will pull the Trivy image, update the vulnerability database, and scan the provided image 
with tag. In this case, I am scanning the V4 image of the Azure Functions dotnet runtime. Once done running, the output should look something like this (granted there are no vulnerabilities).

```
mcr.microsoft.com/azure-functions/dotnet:4 (debian 11.1)
========================================================
Total: 0 (UNKNOWN: 0, LOW: 0, MEDIUM: 0, HIGH: 0, CRITICAL: 0)
```

## Automation

The next step for us was including this in our automation pipelines. With the docker image, all we 
needed was a simple script and some extra command line parameters. After pulling the image to the build 
server, the script ended up being the following.

```bash
docker run --rm \
-v /var/run/docker.sock:/var/run/docker.sock \
-v $HOME/Library/Caches:/root/.cache/ \
-v "$(pwd):/src" \
aquasec/trivy image \
--exit-code 0 \
--severity LOW,MEDIUM \
--ignore-unfixed \
--no-progress \
--format template --template "@contrib/junit.tpl" \
-o src/junit-report-low-med.xml \
$IMAGE:$TAG

docker run --rm \
-v /var/run/docker.sock:/var/run/docker.sock \
-v $HOME/Library/Caches:/root/.cache/ \
-v "$(pwd):/src" \
aquasec/trivy image \
--exit-code 1 \
--severity HIGH,CRITICAL \
--ignore-unfixed \
--no-progress \
--format template --template "@contrib/junit.tpl" \
-o src/junit-report-high-critical.xml \
$IMAGE:$TAG
```

Of course this ended up a little more complicated than the first CLI run of the tool. Let me explain 
the command line parameters and why there are 2 calls.

First, there are some `-v` calls to Docker - this creates volumes to share data between the local 
filesystem and the image's filesystem. This is needed to output the scan results.

In the first command, we set the exit code to 0 with the `--exit-code` parameter. However, this scan 
is only looking at LOW and MEDIUM CVEs via the `--severity` parameter. We do not fail the build if 
there are any LOW or MEDIUM CVEs, hence the 0 exit code.

The `--ignore-unfixed` parameter only alerts us of fixable vulnerabilities. We can't patch 
vulnerabilites that don't have a fix yet. The `--no-progress` parameter hides the database update 
output. This is just to keep the build server output cleaner.

The last 2 parameters - `--format` and `-o` are used to style the output in junit format and save the 
report to the provided location on the local filesystem.

The differences in the 2 commands are the change in exit code to 1 and the severity to HIGH/CRITICAL. 
If there are any high/critical CVEs that are patchable, we fail the build and patch the image.

We run this script at 2 points in our automation pipelines. First, we build an image on PR for 
testing. Once the image is built, the image is scanned. If it fails, the PR cannot be merged into 
main. The second place we run this is a nightly scan on the `latest` tag for every container we build. 
If the build fails, a message is sent to our team members via Microsoft Teams and we patch the 
vulnerability and get that into prod as quickly as possible. Both of these are required because there 
may not be a PR for every image each day.

The open question is what if there is a high/critical CVE that is not patched in your image? This 
question crossed our mind as well. If there's no fix available, we decided whether to allow the 
deployment or not on a case by case basis. Let me know if your team does something different.

## Patching Images

Most of the containers we build are just adding a DLL to a runtime image for Azure Functions. There 
are 2 different methods to patching containers - patch each container via its Dockerfile or create 
a base image that each of your applications use. Each has their tradeoffs.

Patching a base image is easier in terms of where to change. For example, when we first started 
looking into the V4 runtime of Azure Functions there was a vulnerability in libc that needed patching. 
A patched base image file looks like this for Azure Functions V4.

```
FROM mcr.microsoft.com/azure-functions/dotnet:4-slim

RUN apt-get update && \
    apt-get install -y --no-install-recommends linux-libc-dev=5.10.70-1 && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

CMD [ "/azure-functions-hot/Microsoft.Azure.WebJobs.Script.WebHost" ]
```

The `FROM` line shows what image is being patched. The `RUN` line updates the packages, installs 
the patched library version, and cleans the apt cache to keep the image small. Lastly, the `CMD` 
line is taken from the base image to keep the entry point the same for child containers.

The hard part here is managing a base image and alerting all child containers that they need to be 
rebuilt. With Azure DevOps, this can be done via [pipeline triggers][trigger-linkl]. There are also 
probably other ways.

Patching a child image is done the same way. The difference is you would take the `apt-get install` 
line and add it to each of your images using the `dotnet:4-slim` base image. Additionally, you may 
have to add the other lines as well if you are not using a `RUN` command at all in your child image. 
However, this will have to be done in each container. So if you have 10 Function Apps running on V4, 
you will have to add this patch to every Dockerfile.

## Wrap Up

I hope this article gave you some insight on how you could use Trivy in your Docker automation to 
patch CVEs. As always, let me know if an area needs more clarification or you have any questions. 
Contact links are in the footer.

[trivy-link]: https://aquasecurity.github.io/trivy/v0.20.2/
[trivy-install]: https://aquasecurity.github.io/trivy/v0.20.2/getting-started/installation/
[vuln-link]: https://snyk.io/blog/hacking-docker-containers-by-exploiting-base-image-vulnerabilities/
[trigger-link]: https://docs.microsoft.com/en-us/azure/devops/pipelines/process/pipeline-triggers?view=azure-devops
