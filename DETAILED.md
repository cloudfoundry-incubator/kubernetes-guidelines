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
  * [Pod specification](#pod-specification)
  * [Service specification](#service-specification)
  * [Other Kubernetes objects](#other-kubernetes-objects)
  * [Work with other components](#work-with-other-components)
  * [Documentation](#documentation)
  * [Kubernetes versions support](#kubernetes-versions-support)

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

#### Diagnostics Tooling

#### Customisation

#### Upgrades

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

* operability

#### Configuration

The component must be able to use up-to-date configuration

Improves:

* operability

#### Rollback

The application with version n-1 should be able to start with the database that has been migrated to version n

Improves:

* upgradability

#### Work with signals

The application must respect SIGTERM signal and start sending NotReady probe

Improves:

* upgradability

#### Resilience to upgrades

The component is either expected to work during K8s control plane downtime or has a clear notice README that to achieve high availability, the control plane of Kubernetes must have multiple replicas.

Improves:

* upgradability

### Packaging

#### Non-root user

All components should run with a non-root user unless it is completely impossible, including “FROM scratch” images

Improves:

* security

#### Metadata

All components images should have labels in the metadata with repo URL and SHA of the commit it the metadata

Improves:

* operability
* open source

#### Base image

The default base image should come from cloudfoundry/stacks. All components should have the possibility to change the base image

Improves:

* open source

#### Dependencies

The run image should not have packages required for compilation, only for running. i.e. don’t have Go package or JDK in the final image, For java, only JRE should be shipped



* Images are continuously updated with the new version of the base layer. (pack rebase if possible)
* Images are stored in the CFF organization under Dockerhub
* The component has clear instructions on how to build its container image

### Pod specification

* The image references must always include sha256 for versioning
* The component should have the labels that suggested by Kubernetes At least app.kubernetes.io/name, app.kubernetes.io/version. The app.kubernetes.io/part-of is set to Cloud Foundry
* The readiness probe for the main container must be always present
* Liveness probe if present should point to a different endpoint than the readiness probe. Ideally something that does not require any processing
* In general, a single pod should have a single container. There are might be edge-cases, but the pod shouldn’t have more than 5 containers in its spec
* The long-living pod can have up to 2 init containers in its spec
* Each container in a pod must always have configurable CPU & memory requests with sane defaults(required to run 50 applications /start 5 at the same time)
* Memory limits are optional, but if they are present they must be at least 50% bigger than requests
* CPU limits must be never set
* Each component must have its own service account. It must never use default service account
* If the pod does not need access to the Kubernetes API, the service account token is not mounted to it
* The pod spec should satisfy [the restricted pod security policy provided by Kubernetes](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/policy/restricted-psp.yaml)
  * Pod should drop all capabilities
  * Pod should have proper seccomd or apparmor annotation
  * Pod should have property readOnlyRootFilesystem=readOnlyRootFilesystem
* If a pod requires TLS/SSL cert/keys for public consumption it must support utilizing cert-manager.
* Ports that are exposed by pod must have a name which should be the same as in corresponding service
* The specification allows to set affinity and anti-affinity rules
  
### Service specification

* The component creates a service if it has to be accessed by other components.  Service ports should have the name of format `<protocol>[-<suffix>]`, e.g. `grpc-api` or `tcp-database`.  See more in Istio documentation
* The service must have the same labels as a pod

### Other Kubernetes objects

* If the process is expected to have no downtime it has PodDisruptionBudget
* Minimal pod security policy is provided. Ideally, it should be the same (or stricter) as [Kubernetes provided policy](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/policy/restricted-psp.yaml)
* Sample networking policy is provided
* Sample Istio RBAC is provided
* If the component has to be accessed externally, it writes a K8s Ingress resource or a set of Istio VirtualService + Gateway resourcesprovides ingress with free form annotations and the ability to provide a load balancer
* If the component needs some secrets, it has an option to use an existing Kubernetes object with the predefined format. The Kubernetes object name can be provided by the platform engineer.
* The secret for certificates uses known K8s format
* Each stateless component is deployed as a deployment
* The number of replicas is not specified in the template unless it can only be deployed as a single copy.
* Each component creates and attached its own service account.

### Work with other components

* If the component has a soft dependency(can work without it) on another component, the depending part can be skipped. i.e. Eirini deploys with rootfs patcher, but rootfs patcher can be skipped in the deployment.
* The address for the dependent component can always be specified by the platform engineer and has a sane default

### Documentation

* Each non-alpha property that platform engineer can specify is documented in README

### Kubernetes versions support

* Each component is expected to support all supported by CNCF versions of Kubernetes by using correct API specification.
* If API specification differs in version x and x+2, the component has by default the version that is supported in CFCR. Optionally, it can provide a flag to use a different API version
