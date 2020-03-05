# Kubernetes-idiomatic Cloud Foundry components guidelines

See the description for each guideline in [DETAILED document](DETAILED.md)

## Code

* Every component must have an endpoint with health information or crash if it is unhealthy.
* All logs must go in stdout/stderr
* The component must be able to use up-to-date configuration
* The application with version n-1 should be able to start with the database that has been migrated to version n
* The application must respect SIGTERM signal and start sending NotReady probe
* The component is either expected to work during K8s control plane downtime or has a clear notice README that to achieve high availability, the control plane of Kubernetes must have multiple replicas.

## Packaging

* All components should run with a non-root user(but not the user with UID 1337) unless it is completely impossible. Even “FROM scratch” images
* All components images should have labels in the metadata with repo URL and SHA of the commit it the metadata
* All components should have the possibility to change the base image
* The default base image should come from `cloudfoundry/stacks`
* The run image should not have packages required for compilation, only for running. i.e. don’t have Go package or JDK in the final image, For java, only JRE should be shipped
* Images are continuously updated with the new version of the base layer.
* Images are stored in the CFF organisation under Dockerhub
* The component has clear instructions on how to build its container image

## Pod specification

* The image references must always include sha256
* The component should have the labels that suggested by Kubernetes At least `app.kubernetes.io/name`, `app.kubernetes.io/version`. The `app.kubernetes.io/part-of` is set to Cloud Foundry
* The readiness probe for the main container must be always present
* Liveness probe if present should point to a different endpoint than the readiness probe. Ideally, something that does not require any processing
* In general, a single pod should have a single container. There are might be edge-cases, but the pod shouldn’t have more than 5 containers in its spec
* The long-living pod can have up to 2 init containers in its spec
* Each container in a pod must always have configurable CPU & memory requests with sane defaults(required to run 50 applications /start 5 at the same time)
* Memory limits are optional, but if they are present they must be at least 50% bigger than requests
* CPU limits must be never set
* Each component must have its service account. It must never use the `default` service account
* If the pod does not need access to the Kubernetes API, the service account token is not mounted to it
* The pod spec should satisfy [the restricted pod security policy provided by Kubernetes](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/policy/restricted-psp.yaml)
  * Pod should drop all capabilities
  * Pod should have proper `seccomd` or `apparmor` annotation
  * Pod should have property readOnlyRootFilesystem=readOnlyRootFilesystem
* If a pod requires TLS/SSL cert/keys for public consumption it must support utilising cert-manager.
* Ports that are exposed by pod must have a name which should be the same as in the corresponding service
* The specification allows setting affinity and anti-affinity rules
  
## Service specification

* The component creates a service for its pods.
* Service ports should have the name of format `<protocol>[-<suffix>]`, e.g. `grpc-api` or `tcp-database`.  See more in Istio documentation
* The service must have the same labels as a pod

## Other Kubernetes objects

* If the process is expected to have no downtime it has PodDisruptionBudget
* Minimal pod security policy is provided. Ideally, it should be the same (or stricter) as [Kubernetes provided policy](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/policy/restricted-psp.yaml)
* Sample networking policy is provided
* Sample Istio RBAC is provided
* If the component has to be accessed externally, it writes a K8s Ingress resource or a set of Istio VirtualService + Gateway resources provides ingress with free form annotations and the ability to provide a load balancer
* If the component needs some secrets, it has an option to use an existing Kubernetes object with the predefined format. The Kubernetes object name can be provided by the platform engineer.
* The secret for certificates uses known K8s format
* Each stateless component is deployed as a deployment
* The number of replicas is not specified in the template unless it can only be deployed as a single copy.
* Each component creates and attached its own service account.

## Work with other components

* If the component has a soft dependency(can work without it) on another component, the depending part can be skipped. i.e. Eirini deploys with rootfs patcher, but rootfs patcher can be skipped in the deployment.
* The address for the dependent component can always be specified by the platform engineer and has a sane default

## Documentation

* Each non-alpha property that platform engineer can specify is documented in README

## Kubernetes versions support

* Each component is expected to support all supported by CNCF versions of Kubernetes by using correct API specification.
* If API specification differs in version x and x+2, the component has by default the version that is supported in CFCR. Optionally, it can provide a flag to use a different API version
* The component must support both Docker and containerd as container runtimes.
