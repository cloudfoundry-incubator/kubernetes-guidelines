# Kubernetes-idiomatic Cloud Foundry components guidelines

Also check [shorter version](./README.md)

* [Structure](#structure)
* [Wording](#wording)
* [Production readiness criterias](#production-readiness-criterias)
  * [Resilience](#resilience)
    * [Availability](#availability)
    * [Failure Recovery](#failure-recovery)
    * [Isolation](#isolation)
  * [Operability](#operability)
    * [Resource Planning](#resource-planning)
    * [Health Monitoring](#health-monitoring)
    * [Scalability](#scalability)
    * [Logging](#logging)
    * [Diagnostics Tooling](#diagnostics-tooling)
    * [Customisation](#customisation)
    * [Upgrades](#upgrades)
  * [Security](#security)
  * [Open Source](#open-source)
* [Guidelines](#guidelines)
  * [Code](#code)
    * [Health endpoint](#health-endpoint)
    * [Log location](#log-location)
    * [Configuration](#configuration)
    * [Rollback](#rollback)
    * [Work with signals](#work-with-signals)
    * [Resilience to upgrades](#resilience-to-upgrades)
  * [Packaging](#packaging)
    * [Non-root user](#non-root-user)
    * [Metadata](#metadata)
    * [Base image](#base-image)
    * [Dependencies](#dependencies)
    * [Keeping base image up to date](#keeping-base-image-up-to-date)
    * [Image location](#image-location)
    * [Packaging instructions](#packaging-instructions)
  * [Pod specification](#pod-specification)
    * [Image referenced by sha256](#image-referenced-by-sha256)
    * [Pod labels](#pod-labels)
    * [Readiness probe](#readiness-probe)
    * [Liveness probe](#liveness-probe)
    * [Number of containers](#number-of-containers)
    * [Number of init containers](#number-of-init-containers)
    * [Pod requests](#pod-requests)
    * [Pod limits](#pod-limits)
    * [Pod service account](#pod-service-account)
    * [Pod using service account](#pod-using-service-account)
    * [Pod security configuration](#pod-security-configuration)
    * [Using keys](#using-keys)
    * [Pod port names](#pod-port-names)
    * [Affinity](#affinity)
  * [Service specification](#service-specification)
    * [Using services](#using-services)
    * [Service pod names](#service-pod-names)
    * [Service labels](#service-labels)
  * [Other Kubernetes objects](#other-kubernetes-objects)
    * [Pod Disrution Budget](#pod-disrution-budget)
    * [Pod Security Policy](#pod-security-policy)
    * [Networking policy](#networking-policy)
    * [Istio RBAC](#istio-rbac)
    * [Access from outside](#access-from-outside)
    * [Using secrets](#using-secrets)
    * [Service accounts](#service-accounts)
    * [Replicas count](#replicas-count)
  * [Work with other components](#work-with-other-components)
    * [Optional parts](#optional-parts)
    * [Using custom DNS addreses](#using-custom-dns-addreses)
  * [Documentation](#documentation)
  * [Kubernetes versions support](#kubernetes-versions-support)
    * [Container runtime support](#container-runtime-support)

## Structure

The document consists of two big parts: criterias grouped in themes and guidelines that solving some of the problems the criterias define. The intend of the document is to help developers to create a component that the platform engineer would be able to operate using existing Kubernetes patterns.

## Wording

Component - a single application.
Platform engineer - a person responsible for deploying and operating Cloud Foundry.

## Production readiness criterias

### Resilience

Ability to provide acceptable level of service in the face of faults.

#### Availability

#### Failure Recovery

#### Isolation

### Operability

#### Resource Planning

#### Health Monitoring

....

Guidelines:

* [Health checks](#health-endpoint)

#### Scalability

#### Logging

Guidelines:

* [Health checks](#log-location)

#### Diagnostics Tooling

#### Customisation

#### Upgrades

As a platform operator, I am able to upgrade the Kubernetes clustrer with the minimum application downtime.

As a platform operator, I am able to upgrade Cloud Foundry with minimum control plane downtime.

Guidelines:

* [Health checks](#health-endpoint)

### Security

### Open Source

As an open source contributor, I am able to submit a patch for the component(that passing tests).

As an open source platform architect, I am able to consume the component.

## Guidelines

### Code

#### Health endpoint

Every component must have an endpoint with health information or crash if it is unhealthy.

Improves:

* [operability](#health-monitoring).
* [upgrades](#upgrades) by allowing using blue-green deployments.

Read more:

#### Log location

All logs must go in stdout/stderr

Improves:

* [logging](#logging)

#### Configuration

The component must be able to use up-to-date configuration

Improves:

* [customisation](#customisation)

#### Rollback

The application with version n-1 should be able to start with the database that has been migrated to version n

Improves:

* [upgradability](#upgrades)

Reason:

 It is very easy to implement blue-green deployment with the Kubernetes object and do the rollback. Ability to work with a previous version gives the platform engineer the ability to rollback using the `kubectl rollout undo` command.

 See also:

[Kubernetes documentation about deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

#### Work with signals

The application must respect SIGTERM signal and start sending NotReady probe

Improves:

* upgradability

See also:

[Termination of Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/#termination-of-pods)

#### Resilience to upgrades

The component is either expected to work during K8s control plane downtime or has a clear notice README that to achieve high availability, the control plane of Kubernetes must have multiple replicas.

Improves:

* [Kubernetes cluster upgrades](#upgrades)

### Packaging

#### Non-root user

All components should run with a non-root user unless it is completely impossible, including “FROM scratch” images. The user UID should not be 1337 due to [Istio limitations](https://istio.io/docs/ops/deployment/requirements/).

Improves:

* security

See also:

[Pod security policy](#pod-security-policy)

#### Metadata

All components images should have labels in the metadata with repo URL and SHA of the commit it the metadata as [recommended by OCI](https://github.com/opencontainers/image-spec/blob/master/annotations.md#pre-defined-annotation-keys)

Improves:

* operability
* open source
* [diagnostics tooling](#diagnostics-tooling)

Reasons:

* This allows the operator to check if the component has a CVE.
* This also helps scanners that work off of artifact metadata to determine source code/provenance (e.g. OSL, security)

See also:

[Eirini docker image](https://github.com/cloudfoundry-incubator/eirini/blob/16a093f7e43e56779c43b74143aee855173f1748/docker/opi/Dockerfile#L15-L17) as an example

#### Base image

The default base image should come from cloudfoundry/stacks. All components should have the possibility to change the base image

Improves:

* open source

Reasons:

* The base layer for the images should be the same.
* Other Cloud Foundry Foundation members want to build their own base images.

#### Dependencies

The run image should not have packages required for compilation, only for running. i.e. don’t have Go package or JDK in the final image, For java, only JRE should be shipped. It should be possible to get a list of current dependencies installed inside.

Improves:

* upgrades
* resource planning
* recovery
* security

Reasons:

* The unneeded packages increase the size of the image and require upgrading it more often.
* The operator needs to know if the packages in the image up-to-date or not.

See also:

[Kubernetes best practices](https://cloud.google.com/blog/products/gcp/kubernetes-best-practices-how-and-why-to-build-small-container-images) by Google Cloud

#### Keeping base image up to date

Images are continuously updated with the new version of the base layer. (pack rebase if possible)

Improves:

* security

#### Image location

Images are stored in the CFF organization under Dockerhub.

Improves:

* open source

#### Packaging instructions

The component has clear instructions on how to build its container image

Improves:

* open source

### Pod specification

#### Image referenced by sha256

The image references must always include sha256 for versioning. The tags in Docker registries are mutable, this can cause two different version of application to run on the cluster due to node restart.

Improves:

* [Kubernetes cluster upgrades](#upgrades)
* [availability](#availability)

See also:

[Kubernetes Configuration Best Practices](https://kubernetes.io/docs/concepts/configuration/overview/#container-images)

#### Pod labels

The component should have [the labels that suggested by Kubernetes](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/#labels), for example `app.kubernetes.io/name`, `app.kubernetes.io/version`. The `app.kubernetes.io/part-of` is set to Cloud Foundry

Improves:

* logging
* diagnostic tooling

#### Readiness probe

The readiness probe for the main container must be always present.

Improves:

* availability
* upgrades

Reasons:

* the pod won't serve traffic until it is ready.
* the rolling upgrade will wait for the pod to come up before deleting the existing pod.
* if [pod disruption budget](#pod-disrution-budget) is set, the node draining will proceed only if pods are ready.

See also:

[Kubernetes documentation about probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-readiness-probes)

#### Liveness probe

The liveness probe should only fail if the application is in unrecoverable state and has to be restarted. Ideally, liveness probe should not be set and application should crash. If it is present, it should point to a different endpoint than the readiness probe.

Improves:

* availability

Reasons:

* Failing liveness probe causes the restart of the pod - this might take more time if the pod is scheduled on the different node.
* Crashing of the pod might increase pressure on the running pods.

See also:

* [Outage report](https://keepingitclassless.net/2018/12/december-4-nre-labs-outage-post-mortem/)
* [Liveness probes are dangerous](https://srcco.de/posts/kubernetes-liveness-probes-are-dangerous.html)

#### Number of containers

The pod must have as little containers as possible. Ideally, a single pod should have a single container. Most of native Kubernetes deployments have up to three containers.

Improves:

* [upgrades](#upgrades)
* [resilience in general](#resilience)
* [resource planning](#resource-planning)

Reasons:

* All the containers are scheduled on a single node. They require more resources and slow down the start of the pod.
* When one container is down, the pod won't be processing requests.
* When configuration is changed for a single container, the whole pod has to restart.

#### Number of init containers

The non short-living pod must have up as little init containers as possible.  Most of native Kubernetes deployments to 2 init containers in its spec.

Improves:

* [upgrades](#upgrades)
* [failure recovery](#failure-recovery)

Reasons:

* The pod executes init containers sequentially and this slows down the pod startup. Consider using Kubernetes jobs instead.
* Crashes in init containers do not show up in pod crash count.
* The logs from init containers is impossible to get from the pod after pod is started.

#### Pod requests

Each container in a pod must always have configurable CPU & memory requests with sane defaults(required to run 50 applications /start 5 at the same time)

Improves:

* [Resource Planning](#resource-planning)
* [Availability](#availability)

Reasons:

* Kubernetes blocks the resources on the node and does not allow to schedule more applications that is possible. Overcommiting slows down the deployment.

See all:

[Managing Compute Resources](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/)

#### Pod limits

Memory limits are optional, but if they are present they must be at least 50% bigger than requests. CPU limits must be never set.

Improves:

* [Resource planning](#resource-planning)

Reasons:

* [CPU limits are broken now](https://github.com/kubernetes/kubernetes/issues/67577)
* Pod will restart if it uses more memory than allowed

See also:

* [Kubecon presentation](https://www.youtube.com/watch?v=UE7QX98-kO0)

#### Pod service account

Each component must have its own service account. It must never use default service account.

Improves:

* security
* customisation

Reason:

This allows attaching [pod security policy](#pod-security-policy) to the pod.

#### Pod using service account

If the pod does not need access to the Kubernetes API, the service account token is not mounted to it

Improves:

* security

#### Pod security configuration

The pod spec should satisfy [the restricted pod security policy provided by Kubernetes](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/policy/restricted-psp.yaml)

* Pod should drop all capabilities.
* Pod should have proper seccomd or apparmor annotation
* Pod should have property readOnlyRootFilesystem=readOnlyRootFilesystem

Improves:

* security

#### Using keys

If a pod requires TLS/SSL cert/keys for public consumption it must support utilizing cert-manager.

Improves:

* customisation

Reason:

Kubernetes secrets have a special format for certificates and the operators expect the components to use it.

#### Pod port names

Ports that are exposed by pod must have a name which should be the same as in corresponding service

Improves:

* customisation

Reason:

Istio requires proper [port names](https://istio.io/docs/ops/deployment/requirements/).

#### Affinity

The specification allows to set affinity and anti-affinity rules.

Improves:

* availability
* isolation
* resource planning

See also:

[Kubernetes documentation](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/)

### Service specification

#### Using services

All pods should be part of services.

Reason:

* [Istio requirements](https://istio.io/docs/ops/deployment/requirements/).

#### Service pod names

The component creates a service if it has to be accessed by other components.  Service ports should have the name of format `<protocol>[-<suffix>]`, e.g. `grpc-api` or `tcp-database`.  See more in Istio documentation.

Reason:

* [Istio requirements](https://istio.io/docs/ops/deployment/requirements/).

#### Service labels

The service must have the same labels as a pod

Improves:

* [logging](#logging)
* [diagnostics](#diagnostics-tooling)

### Other Kubernetes objects

#### Pod Disrution Budget

If the process is expected to have no downtime it has PodDisruptionBudget

Improves:

* [Kubernetes cluster upgrades](#upgrades)

Reason:

Metadata from higher level is ignored when the node is drained. The pod disruption budget prevents too many pods from going down.

See also:

[Kubernetes documentation about disruptions](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)

#### Pod Security Policy

Minimal pod security policy is provided. Ideally, it should be the same (or stricter) as [Kubernetes provided policy](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/policy/restricted-psp.yaml)

Improves:

* [Security](#security)

#### Networking policy

Sample networking policy is provided

Improves:

* [Security](#security)
* [Customisation](#customisation)
* [Open Source](#open-source)

#### Istio RBAC

Sample Istio RBAC is provided

Improves:

* [Security](#security)
* [Customisation](#customisation)
* [Open Source](#open-source)

#### Access from outside

If the component has to be accessed externally, it writes a K8s Ingress resource or a set of Istio VirtualService + Gateway resourcesprovides ingress with free form annotations and the ability to provide a load balancer

Improves:

* [Customisation](#customisation)
* [Open Source](#open-source)

#### Using secrets

If the component needs some secrets, it has an option to use an existing Kubernetes object with the predefined format. The Kubernetes object name can be provided by the platform engineer.
The secret for certificates uses known K8s format

#### Service accounts

Each component creates and attached its own service account.

Improves:

* [Security](#security)
* [Customisation](#customisation)

#### Deployment

Each stateless component is deployed as a deployment

#### Replicas count

The number of replicas is not specified in the template unless it can only be deployed as a single copy.

Improves:

* [Availability](#availability)

### Work with other components

#### Optional parts

If the component has a soft dependency(can work without it) on another component, the depending part can be skipped. i.e. Eirini deploys with rootfs patcher, but rootfs patcher can be skipped in the deployment.

#### Using custom DNS addreses

The address for the dependent component can always be specified by the platform engineer and has a sane default

### Documentation

Each non-alpha property that platform engineer can specify is documented in README.

### Kubernetes versions support

Each component is expected to support all supported by CNCF versions of Kubernetes by using correct API specification.

Improves:

* [upgrades](#upgrades)

Reason:

APIs get deprecated and has to be fixed in advance.

#### Container runtime support

The component must support both Docker and ContainerD as container runtimes.

Improves:

* [Kubernetes cluster customisation](#customisation)

Reason:

Docker is widely used software. ContainerD implements CRI(container runtime interface) and is used in several public cloud offerings.
