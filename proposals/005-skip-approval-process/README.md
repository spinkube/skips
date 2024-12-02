# SKIP 005 - SKIP Approval Process

Summary: A process for approving future SKIPs.

Owner: Caleb Schoepp <caleb.schoepp@fermyon.com>

Impacted Projects:

- [x] spin-operator
- [x] `spin kube` plugin
- [x] runtime-class-manager
- [x] containerd-shim-spin
- [x] Governance
- [ ] Creates a new project

Created: 30/10/2024

Updated: n/a

## Background

SKIPs were designed as a lightweight way to have discussions about changes that are more complex or affect multiple projects. They have worked well for this purpose, but we never codified how we should reach agreement around approving a SKIP. This proposal seeks to change that.

## Proposal

The following conditions must be met for a SKIP to be considered approved:

- The SKIP has been open for comment for at least two weeks and any objecting comments are resolved.
- The SKIP has been approved by at least two maintainers who are not an author.
- The SKIP has been approved by at least one maintainer of each different impacted project[^1].
- The SKIP has been discussed as an agenda item in at least one SpinKube community meeting[^2].

Once a SKIP meets these conditions it is considered approved. The SKIP may now be merged at will by either the author or a reviewer. It is considered official once it has been merged.

### Disagreement

Ideally the approval process of a SKIP will flow smoothly with disagreements being worked out in the SKIP PR. However, it is possible that a SKIP might encounter some blocking disagreement:

- A maintainer is strongly opposed to the SKIP.
- A maintainer disagrees with the impacted projects list.

As SpinKube is still early in its life we shouldn't be too prescriptive about scenarios like this. If an impasse is reached it should be resolved by the SpinKube maintainers gathering to discuss and reach a consensus.

## Alternatives Considered

### Handling disagreement with a vote

An alternative to solving disagreement through conversation and consensus was requiring an async vote from the maintainers with a super-majority. This process felt too rigid for where SpinKube is currently at and also introduces adversarial dynamics of who is included in the vote.

[^1]: It is expected that the SKIP author accurately reflects which projects are impacted by the SKIP. If a reviewer believes a project is or is not impacted they may request the author to update the SKIP.
[^2]: There are no quorum requirements.
