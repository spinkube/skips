# SKIP 001 - Adding `spin kube doctor` subcommand

Summary: 

This proposal adds a command to `spin kube` plugin to improve the user experience of troubleshooting. To run SpinKube on a given Kubernetes cluster needs a specific version of containerd as well as some pre-requisites such as cert-manager. This command runs all those checks programmatically against the cluster and provide details of what prerequisites are missing and optionally fix them automatically.

Owner: Rajat Jindal <rajat.jindal@fermyon.com>

Impacted Projects:

- [ ] spin-operator
- [X] `spin kube` plugin
- [ ] runtime-class-manager
- [ ] containerd-shim-spin
- [ ] Governance
- [ ] Creates a new project

Created: May 29th 2024

Updated: May 29th 2024

## Background

There are a bunch of Kubernetes providers with subtle differences in how they are configured. They may be [using the old version of containerd](https://github.com/spinkube/spin-plugin-kube/issues/80) by default or [may not be using containerd as default runtime](https://github.com/spinkube/documentation/pull/161) at all.

In addition to that we have a few prerequisites for SpinKube to work correctly, such as cert-manager, containerd-shim-spin.

Back and forth on asking this information from the user can consume a significant amount of time for both the user and the maintainer trying to help. 

## Proposal

I am proposing to add a subcommand to the `spin kube` plugin that takes care of verifying the details of the cluster and provides an overview of what is configured correctly and what is missing. 

One immediate usecase is when a user creates a ticket about the SpinKube not working on their cluster, we can point them to run this tool and share the output in the ticket for faster resolutions. 

`spin kube doctor` will have the concept of reusable and configurable checks. Each check will have a corresponding function that abstracts the details of how that check is performed:


```
type CheckFn func(ctx context.Context, k Provider, check Check) (Status, error)
```

An example of how to check if a crd is installed:

```
var isCrdInstalled = func(ctx context.Context, k provider.Provider, check provider.Check) (provider.Status, error) {
        _, err := k.DynamicClient().Resource(schema.GroupVersionResource{
                Group:    "apiextensions.k8s.io",
                Version:  "v1",
                Resource: "customresourcedefinitions",
        }).Get(ctx, check.ResourceName, metav1.GetOptions{})
        if err != nil {
                if errors.IsNotFound(err) {
                        return provider.Status{
                                Name:     check.Name,
                                Ok:       false,
                        }, nil
                }

                return provider.Status{
                        Name:     check.Name,
                        Ok:       false,
                        HowToFix: check.HowToFix,
                }, err
        }

        return provider.Status{
                Name: check.Name,
                Ok:   true,
        }, nil
}

```

We can then have a default set of checks that can be performed on the cluster with the possibility of customizing them if required. 

### how multiple providers are supported

Because these checks will be using Kubernetes API to perform checks, they should work on most of the Kubernetes distribution providers by default. However, in case we need to override the checks for a given distribution, we can do so by implementing the following interface for that provider (specifically the GetCheckOverride function):

```
type Provider interface {
        Name() string
        Client() kubernetes.Interface
        DynamicClient() dynamic.Interface
        Status(ctx context.Context) ([]Status, error)
        GetCheckOverride(ctx context.Context, check Check) CheckFn
}

```

An example of this to adjust the check to support k3d is shown below:

```
func (k *k3d) GetCheckOverride(ctx context.Context, check provider.Check) provider.CheckFn {
        switch check.Type {
        case checks.CheckBinaryInstalledOnNodes:
                return binaryVersionCheck
        }

        return nil
}

var binaryVersionCheck = func(ctx context.Context, k provider.Provider, check provider.Check) (provider.Status, error) {
        return checks.ExecOnEachNodeFn(ctx, k, check, []string{"/host/bin/containerd-shim-spin-v2"}, []string{"-v"})
}

```

#### Examples

Here is how the output currently looks like for a few providers:

##### k3d
```
$ go run main.go doctor --context k3d-wasm-cluster


#-------------------------------------
# Running checks for SpinKube setup
#-------------------------------------

✓ Containerd version is supported
✓ Containerd shim is installed and configured
✗ Spin App CRD is installed
✗ Spin App Executor CRD is installed
✗ Cert Manager CRD is installed
✗ Cert Manager is running
✗ Runtime Class is installed
✗ Spin Operator is running

Error: please fix above issues
exit status 1

```

##### Kind

```
$ go run main.go doctor --context kind-spin-kind


#-------------------------------------
# Running checks for SpinKube setup
#-------------------------------------

✗ Containerd version is supported
-> actual version: "1.7.1" not one of the expected versions: [~1.6.26-0 ~1.7.7-0]

✓ Containerd shim is installed and configured
✗ Spin App CRD is installed
✗ Spin App Executor CRD is installed
✗ Cert Manager CRD is installed
✗ Cert Manager is running
✗ Runtime Class is installed
✗ Spin Operator is running

Error: please fix above issues
exit status 1


```

##### Minikube

```
$ go run main.go doctor --context minikube


#-------------------------------------
# Running checks for SpinKube setup
#-------------------------------------

✗ Containerd version is supported
-> found container runtime "docker://26.0.1" instead of containerd

✗ Containerd shim is installed and configured
✗ Spin App CRD is installed
✗ Spin App Executor CRD is installed
✗ Cert Manager CRD is installed
✗ Cert Manager is running
✗ Runtime Class is installed
✗ Spin Operator is running

Error: please fix above issues
exit status 1

```

##### AKS

```
$ go run main.go doctor --kubeconfig ~/kubeconfig-mikkel


#-------------------------------------
# Running checks for SpinKube setup
#-------------------------------------

✓ Containerd version is supported
✓ Containerd shim is installed and configured
✓ Spin App CRD is installed
✓ Spin App Executor CRD is installed
✓ Cert Manager CRD is installed
✓ Cert Manager is running
✓ Runtime Class is installed
✓ Spin Operator is running


All looks good !!

```


## Alternatives Considered

An alternative way could be to document these steps and ask users to run these steps manually and provide the output.

