# SKIP 00X - CRD Domains

Summary: We should pivot to using the _spinkube.dev_ domain for all CRDs.

Owner: Caleb Schoepp <caleb.schoepp@fermyon.com>

Impacted Projects:

- [x] spin-operator
- [ ] `spin kube` plugin
- [x] runtime-class-manager
- [ ] containerd-shim-spin
- [ ] Governance
- [ ] Creates a new project

Created: 11/07/2024

Updated: n/a

## Background

Kubernetes resources are identified by a unique [group, version, and kind](https://book.kubebuilder.io/cronjob-tutorial/gvks). Furthermore, the resource must be namespaced under a valid domain name. Currently the Spin Operator project has two custom resource definitions (CRDs): `SpinApp` and `SpinAppExecutor`. These CRDs use the domain _spinoperator.dev_.

## Proposal

I am proposing that we migrate from using _spinoperator.dev_ to _spinkube.dev_ as the domain for the Spin Operator CRDs. I also propose that going forward any new CRDs created by the Spin Operator project or any other SpinKube project use the _spinkube.dev_ domain.

The benefit of this change include:

- Consistency: All SpinKube projects will use the same domain for their CRDs.
- Branding: The _spinkube.dev_ domain is more descriptive of the overall project than _spinoperator.dev_.
- Future Proofing: The _spinkube.dev_ domain is more generic and allows for more flexibility in the future should anything about the Spin Operator change.

Changing the domain of a CRD is a breaking change and will require a full upgrade process which involves uninstalling the old CRDs and reinstalling the new CRDs. This is suboptimal, but is only reasonably achievable early in the lifetime of project like we are right now. Breaking changes like this only get harder the longer we wait.

## Alternatives Considered

### Sticking with _spinoperator.dev_

Alternatively, we could stick with the status quo if we don't think the breaking change is worth it.
