# SKIP 005 - SpinKube Releases

Summary: A proposal to create SpinKube releases which denote a stable set of releases of the SpinKube projects.

Owner: kate.goldenring@fermyon.com

Impacted Projects:

- [x] spin-operator
- [x] `spin kube` plugin
- [x] runtime-class-manager
- [x] containerd-shim-spin
- [ ] Governance
- [ ] Creates a new project

Created: 10-31-2024

Updated: 10-31-2024

## Background

SpinKube is a collection of four projects, the container-shim-spin, Spin Operator, runtime class manager (kwasm for now), and spin-kube plugin. These project have their own release lifecycles; however, there are times in which certain features are only enabled in specific releases of each project are used. Users and marketplace offering maintainers will want to know when a new feature has been added to all the projects of SpinKube. One way to denote that is with SpinKube releases.

To give an example of why making it clear which versions of the projects are compatible, we can look at the OpenTelemetry (OTEL) feature. Support for configuring OTEL for monitoring Spin applications was added to the shim in `v0.15.0` and the ability to pipe OpenTelemetry parameters to the shim from a `SpinApp` was added in the Spin Operator `v0.3.0`. This means that people using the `v0.3.0` operator with a shim version less than `v0.15.0` may set OTEL details in their `SpinApp` to find that it is not configured in the app by the shim.

## Proposal

SpinKube will periodically provide releases. A release defines the release set of SpinKube projects. For example, SpinKube v0.X that brings in support for OTEL could be defined as:

- Shim / Node Installer: v0.15.0
- Operator: v0.3.0
- Kwasm: v0.2.3
- `spin kube` plugin: v0.3.0

> Note: a `spin kube` plugin version is specified even though it does not provide the ability to configure OTEL via the scaffold command. However, the output SpinApp can be updated with the OTEL fields. The `spin kube` version is provided to indicate the version that uses the same SpinApp CRD as the Spin Operator

SpinKube releases will not be cut at a certain cadence; rather, project maintainers may advise that a new release be cut upon the addition of new features.

The installation guide of the SpinKube documentation will display SpinKube release matrices and a features changelog. Marketplace maintainers can reference changes to this document.

In the future, we may also consider providing a "feature detection" Spin application that can be deployed to your cluster to determine which features your executor supports.

## Alternatives Considered

This could just be documented with compatibility matrixes (such as [this one](https://github.com/spinkube/documentation/pull/253/commits/e93aed2d5b30d2c220289572acc80b3dc4e4a448)).
