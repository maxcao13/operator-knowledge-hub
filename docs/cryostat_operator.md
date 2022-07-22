# Things to know about cryostat operator

Here are some useful things to know if you are working on https://github.com/cryostatio/cryostat-operator.

## Prerequisites

Minimum hardware requirements to run crc instance ([reference](https://crc.dev/crc/#minimum-system-requirements-hardware_gsg)):
- 4 physical CPU cores
- 9 GB of free memory
- 35 GB of storage space

**Note**: Some workload might require more resources, for examples, enabling cluster monitoring.

CRC requires the `libvirt` and `NetworkManager` packages to run on Linux. On Fedora, you will likely need to:
```bash
sudo dnf install NetworkManager
```

## Setup/Start CodeReady Container

Downloaded `crc` binary from [Openshift console](https://console.redhat.com/openshift/create/local) (**recommended**) or build from [source](https://github.com/code-ready/crc). Once downloaded/built, add it to `PATH`:

```bash
sudo install -o root -g root -m 0755 /path/to/crc /usr/local/bin/crc # This add crc to /usr/local/bin
```

Set up environment:

```bash
crc setup
```

Start a cluster instance:

```bash
crc start
```


**Note**: To enable cluster monitoring, run (before starting or after stopping instance):

```bash
crc config set enable-cluster-monitoring true
```

Then, start the instance again. Mostly likely, you will need to run `crc` with more RAM (default 9GB).

```bash
crc start -m 14336 # 14GiB (recommended for core functionality)
```

## Deployment

There are 2 ways to deploy the operator to the k8s/openshift cluster.

1. Manual deployment with `make deploy`. This way you can test our your changes to configuration files (i.e. `*.yaml`) with `crc`.
2. Bundle deployment with `make deploy_bundle` (OLM). Basically YAML definitions in config/ go in the bundle image, Go code goes in the operator image. Note that:
	> OLM runs by default in OpenShift Container Platform 4.7.

## Local changes

Any changes to go source files requires building a operator image and modify `OPERATOR_IMG`. The recommended way to test your local changes is manual deployment with `make deploy`.

> If you make changes to the Go sources, you'd need to build and push a custom operator image and pass that to make deploy with the OPERATOR_IMG variable. 

```bash
# Setting up env, (these are set to defaults in the operator Makefile, overwrite with these values in your shell or in the Makefile itself)
export IMAGE_VERSION="2.2.0-dev" # Tag
export IMAGE_NAMESPACE="quay.io/{YOUR_QUAY_USERNAME}" # Quay registry, e.g. "quay.io/thvo" or "quay.io/macao"
export OPERATOR_NAME="cryostat-operator"
export DEPLOY_NAMESPACE="default" 

# Build and push the image to remote registry
make oci-build && \
podman image prune -f && \
podman push $IMAGE_NAMESPACE/$OPERATOR_NAME:$IMAGE_VERSION

# Deploy the running cluster
make deploy OPERATOR_IMG=$IMAGE_NAMESPACE/$OPERATOR_NAME:$IMAGE_VERSION # or just `make deploy` if env variables set correctly
```

## Monitoring

**Note**: `oc` will be the primary cluster client here. To get help, run `oc help`.

To get basic information on the running pods:

```bash
oc get pods [pod-id]
```

To get "top" information (resource metrics):

```bash
oc adm top pods [pod-id] [--containers] # Container flag to check individual containers
```

To get basic informations on nodes:

```bash
oc describe nodes [node-id]
```

To get basic information on deployment (i.e. mostly likely the manager deployment):

```bash
oc describe deploy [deploy-id]
```

**Tips**: Any command above can be prefixed with `watch`. For example, `watch oc get pods`. This way you can periodically check cluster information without rerunning the command.


## Project Structure

Use this link here: https://book.kubebuilder.io/

With the underlying `Kubebuilder` as said in [FAQ](https://sdk.operatorframework.io/docs/faqs/):

> Operator SDK uses Kubebuilder under the hood to do so for Go projects, such that the operator-sdk CLI tool will work with a project created by kubebuilder. 

The project structure should be similar. Furthermore, some operator sdk commands are compatible in behavior with its kubebuilder counterpart:

> Just keep in mind that when you see an instruction such as: $ kubebuilder <command> you will use $ operator-sdk <command>

### api/v1beta1

Basically, we define our `Kind`s in this directory.

> Kubernetes functions by reconciling desired state (Spec) with actual cluster state (other objects’ Status) and external state, and then recording what it observed (Status). Thus, every functional object includes spec and status. A few types, like ConfigMap don’t follow this pattern, since they don’t encode desired state, but most types do.
	
Kinds are defined in source files as `<kind>_types.go`. Each kind will be defined as its own along with its `Spec`, `Status` and `<Kind>_List` type (a collection of instances of that `Kind`).

Others (you won't have to edit):
- `groupversion_info.go`: contains common metadata about the group-version (see tags at the top). Also defines commonly useful variables that help us set up our Scheme.
- `zz_generated.deepcopy.go`: contains the autogenerated implementation of the `runtime.Object interface`, which marks all of our root types as representing Kinds. The core of the runtime.Object interface is a deep-copy method, DeepCopyObject.

### internal/

Here we define implementations for our operator controller. This directory is deployed as a Go package that is referenced in `main.go`, which will be compiled into a running binary.

#### internal/test

We define our test resources, structs to be used in test files for controllers. Most importantly is the `resources.go`, which defines functions to create Cryostat CR with test specs.

#### internal/controllers/client

STILL EXPLORING...

#### internal/controller/common

This directory include utilities and resource definitions used in operator controller logics. For example, some network configurations:

#### internal/controller

STILL EXPLORING...

## Basic terminology

| Term | Definition |
|------|------------|
| `Groups` | Basically, a Group is a collection of related functionalities. |
| `Version` | Each group has one or more version (i.e. v2.0.0, beta, alpha). |
| `Kind` | Each API group-version contains one or more API types, which we call Kinds.  |
| `Resource` | A resource is simply a use of a Kind in the API. For instance, the pods resource corresponds to the Pod Kind. With CRDs, each Kind will correspond to a single resource. Resources are always lowercase, and by convention are the lowercase form of the Kind.|
| `GroupVersionKind` (GVK) | It refers to a kind in a particular group-version. Each GVK corresponds to a root Go type in a package. |
| `Scheme` | A way to keep track of what Go type corresponds to a given GVK |
| `CustomResourceDefinition` (CRD)| They are a definition of our customized Objects. |
| `CustomResource` (CR)| They are an instance of CRD. |
| `Controller` | Ensure, for any given object, the actual state of the world (both the cluster state, and potentially external state like running containers for Kubelet or loadbalancers for a cloud provider) matches the desired state in the object. Each controller focuses on one root Kind, but may interact with other Kinds. |
| `Reconciler` | The logic that implements the reconciling for a specific kind.  A reconciler takes the name of an object, and returns whether or not we need to try again (e.g. in case of errors or periodic controllers, like the HorizontalPodAutoscaler). |
## Understanding API Markers

For example, you might notice:

```go
// A ConfigMap containing a .jfc template file
type TemplateConfigMap struct {
	// Name of config map in the local namespace
	// +operator-sdk:csv:customresourcedefinitions:type=spec,xDescriptors={"urn:alm:descriptor:io.kubernetes:ConfigMap"}
	ConfigMapName string `json:"configMapName"`
	// Filename within config map containing the template file
	Filename string `json:"filename"`
}
```

Check out links below for information on these annotations:

- API Markers: https://sdk.operatorframework.io/docs/building-operators/golang/references/markers/
- Struct tags: https://stackoverflow.com/questions/10858787/what-are-the-uses-for-tags-in-go
- OLM Descriptors: https://github.com/openshift/console/blob/master/frontend/packages/operator-lifecycle-manager/src/components/descriptors/reference/reference.md

## Testing

An example of test framework setup: https://onsi.github.io/ginkgo/#separating_creation_and_configuration_

## Release on OperatorHub

Cryostat Operator is released on OperatorHub. Please check out the [latest tag](https://github.com/cryostatio/cryostat-operator/tags). To speed up the release process, you can run the test suite locally before opening the PR.

The original comments on the release 2.1 documented all steps you need: https://github.com/cryostatio/cryostat-operator/issues/396#issuecomment-1161988116

The content of the new release is under `bundle/` directory (on approriate tag). You might need to rename and make some changes. Please check out previous release for refererences.
