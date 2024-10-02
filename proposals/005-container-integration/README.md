# SKIP 005 - Container Integration in SpinKube

Summary: Provide a path for running containers alongside WebAssembly workloads in SpinKube.

Owner: Michelle Dhanani <michelle@fermyon.com>

Impacted Projects:

- [X] spin-operator
- [X] `spin kube` plugin
- [ ] runtime-class-manager
- [X] containerd-shim-spin
- [ ] Governance
- [ ] Creates a new project

Created: October 2, 2024

Updated: October 2, 2024

## Background

There are scenarios where one might want to run a container alongside a WebAssembly application in the same pod. One reason might be a continuty of tooling (service meshes, logs, metrics). Another reason is there is a part of a logical workload that can be WebAssembly but another part that cannot run as WebAssembly and would need to run as a sidecar container in the same Pod and be able to communicate to the WebAssembly bits over localhost.

## Proposal

One option is to add a way to configure containers in the SpinApp CR.

TBD - would like to collect feedback here.

## Alternatives Considered

What else have you considered and why is the proposed solution a better fit?
N/A
