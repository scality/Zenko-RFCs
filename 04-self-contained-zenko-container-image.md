# ZIP4 - SELF CONTAINED ZENKO CONTAINER IMAGE

## Overview

This ZIP is about having a very simple way to deploy Zenko without having to install Kubernetes or Minikube. Ideally
by just doing 'docker run'.

## Problem Description

_Detailed explanation of the problem space of the ZIP. Use as many
words as necessary._

The original discussion in Zenko forum discussion can be found [here](https://forum.zenko.io/t/how-to-run-zenko-on-a-single-machine/504).

## Use-cases Description

_What uses-cases does this address? what impact on actors does this
change have?_

We need to provide a way for users to easily deploy Zenko without worrying too much
about the technical details related to Kubernetes or Minikube installation. Those users
could be demo makers, decision makers, curious people, potential Zenko contributors.

Other actors, such as developers, deployers or people trying to go more in-depth could then deploy
Zenko in [Minikube](https://www.zenko.io/blog/deploy-zenko-locally-with-minikube/), 
[Metalk8s](https://www.zenko.io/metalk8s/) or use a Kubernetes cloud offering.

_Ensure you are clear about the actors in each use case: developers,
users, deployers, etc._

## Technical Details

_Describe in details how the ZIP will be implemented. Which components
will be modified, how, the protocols, mechanisms, etc. Use pseudo-code
and anything that will help reviewers understand what you have in
mind._

The current distribution of Zenko is using the [Zenko Helm chart](https://zenko.readthedocs.io/en/stable/) for Kubernetes. 

The technical difficulty is having to maintain multiple ways of deploying Zenko, as we faced in the past with the Swarm installation of Zenko.

The stack evolves quickly and the following problems could occur:
* One tuning of one component could be added/removed.
* One tuning of one component could change name.
* New components could be added.
* Default platform settings of components (e.g. at Kubernetes level) could change.
* Incompatibility of fundamental concepts between platforms (e.g. Swarm does not have the concept of ingresses).

If we had to maintain a Dockerfile with a supervisord (or equivalent) we would have to 
version it and test it to ensure that we did not break anything. The problems would then be:
- Potentially giving more work to developers maintaining another distribution of Zenko.
- Holding releases because the 'demo' Zenko would break.

One possibility to avoid this extra-maintainance cost could be to use [KIND - Kubernetes in Docker](https://github.com/kubernetes-sigs/kind).

## Alternatives

_What are other ways we could implement this, and why are we not using these._


