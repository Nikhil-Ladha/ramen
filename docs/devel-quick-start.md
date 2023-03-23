<!--
SPDX-FileCopyrightText: The RamenDR authors
SPDX-License-Identifier: Apache-2.0
-->

# Ramen developer quick start guide

This page will help you to set up a development environment for the
*Ramen* project.

## What you’ll need

To set up a *Ramen* development environment you will need to run 3
Kubernetes clusters. *Ramen* makes this easy using minikube, but you need
enough resources:

- Bare metal or virtual machine with nested virtualization enabled
- 8 CPUs or more
- 16 GiB of free memory
- 20 GiB of free disk space
- Internet connection
- Linux - tested on *RHEL* 8.6 and *Fedora* 37.

## Getting the source

1. Fork the *Ramen* repository on github

1. Clone the source locally

   ```
   git clone git@github.com:my-github-user/ramen.git
   ```

1. Add an `upstream` remote, to make it easy to sync your repo from
   upstream.

   ```
   git remote add upstream https://github.com/RamenDR/ramen.git
   ```

1. Set up a commit-msg hook to sign off your commits

   *Ramen* requires a `Signed-off-by: My Name <myemail@example.com>`
   footer in the commit message. To add it automatically to all commits,
   add this hook:

   ```
   $ cat .git/hooks/commit-msg
   #!/bin/sh
   # Add Signed-off-by trailer.
   sob=$(git var GIT_AUTHOR_IDENT | sed -n 's/^\(.*>\).*$/Signed-off-by: \1/p')
   git interpret-trailers --in-place --trailer "$sob" "$1"
   ```

   And make the hook executable:

   ```
   chmod +x .git/hooks/commit-msg
   ```

That's all! You are ready to submit your first pull request!

## Setting up the `drenv` tool

*Ramen* uses the `drenv` tool to build a development environment
quickly. Please follow the
[Setup instructions](https://github.com/RamenDR/ramen/tree/main/test#setup)
to set up the `drenv` tool.

## Starting the test environment

*Ramen* supports many configurations, but the only development
environment available now is the `regional-dr.yaml`. This environment
simulates a Regional DR setup, when the managed clusters are located in
different zones (e.g. US, Europe) and storage is replicated
asynchronously. This environment includes 3 clusters - a hub cluster
(`hub`), and 2 managed clusters (`dr1`, `dr2`) using `Ceph` storage.

To start the environment, run:

```
drenv start regional-dr.yaml
```

Starting takes at least 5 minutes, depending on your machine and
internet connection.

You can inspect the environment using kubectl, for example:

```
kubectl get deploy -A --context hub
kubectl get deploy -A --context dr1
kubectl get deploy -A --context dr2
```

## Building the Ramen operator

1. Setup the environment variables for the Makefile

   ```
   export IMAGE_REPOSITORY=my-quay-user
   export IMAGE_TAG=canary
   ```

1. Build the *Ramen* operator container:

   ```
   make docker-build
   ```

   This builds the image `quay.io/my-quay-user/ramen-operator:canary`

1. Pushing the `ramen-operator` image to the clusters

   You can push the container image to your private quay repo, or load
   them into the clusters.

    - Loading the `ramen-operator` image into the clusters

      ```
      podman save quay.io/nirsof/ramen-operator:canary \
          -o /tmp/ramen-operator.tar

      for p in hub dr1 dr2; do
          minikube -p $p image load /tmp/ramen-operator.tar
      done

      rm /tmp/ramen-operator.tar
      ```

    - Pushing the `ramen-operator` image to your private repo

      ```
      make docker-push
      ```

      You may need to login first using `podman login quay.io`.

1. Deploying to hub and cluster

   ```
   kubectl config use-context hub
   make deploy-hub

   for p in dr1 dr2; do
       kubectl config use-context $p
       make deploy-dr-cluster
   done
   ```

## Configure the ramen operator on the hub

Ramen need to be configured to know about the managed clusters and the
s3 endpoints. To configure the test environment, run this script:

```
test/ramen-config/deploy regional-dr
```

If you need to remove your configuration, use:

```
test/ramen-config/undeploy regional-dr
```

## The next steps

At this point *Ramen* is ready to protect workloads in your cluster, and
you are ready for the next steps:

- Enable disaster recovery for an application
- Failing over the application to another cluster
- Relocating an application back to the original cluster