# SKIP 000 - What is a SKIP?

Summary: A lightweight process for coordinating changes to SpinKube

Owner: Danielle Lancashire <dani@builds.terrible.systems>

Impacted Projects:

- [ ] spin-operator
- [ ] `spin kube` plugin
- [ ] runtime-class-manager
- [ ] containerd-shim-spin
- [x] Governance
- [ ] Creates a new project

Created: 18/04/2024

Updated: 18/04/2024

## Background

SpinKube is a new project that was the "coming together" of many seperate
efforts under a common banner. We've quickly found that there are changes we
wish to make that cross across multiple projects, and decisions that require
broader scope to be understandable. We need a lightweight way to have those
discussions and reach agreement about direction.

## Proposal

A SKIP is a tool for recording a potential solution to a problem. It follows a
specific format. There is a specific process for adopting a SKIP. And there is
a process for recording when a SKIP is no longer the accepted practice.

SKIPs are living documents. They change over time in both minor and major ways.
The SKIP itself, combined with Git's change tracking, should indicate how it
has changed.

It is customary to refer to a SKIP by its number (”SKIP 004,” pronounced “skip
oh-oh-four” or “skip four”). When necessary, it may also be referenced by its
version (”SKIP 000 v1”)

### The Document Head

Two pieces of information appear at the top of the document before the
Background section:

- **Overview:** This is one paragraph that summarizes the document. It should
  contain both a summary of the decision point (problem) and a
  summary of the proposed solution.

- For any SKIPs exceeding a couple of pages, it is recommend that a table of
  contents also be added. In particular, with SKIPs that have many subheadings,
  the table of contents can serve as an outline of the structure of the proposal.

### The Background Section

In the background section, clearly articulate the decision point that has
necessitated this SKIP. Identify the key problem, a critical gap, or the
motivating factor. A background section may be one paragraph or may be a few pages.
The background section may be broken into multiple subsections (with subsections
starting at Heading 3 level).

On occasion, another document may clearly spell out the problem. In such cases,
one is encouraged to point to that document as the definitive source, and merely
summarize in the background section.

However, if no such document does exist, the background section should provide
ample material for understanding the issue. Links to other resources are
encouraged, though the author should take into consideration the possibility
that such resources may not always be available.

### The Proposal Section

The Proposal section outlines a proposed solution, process, or practice that
clearly answers the issue as outlined in the Background section. It will be one
or more paragraphs, possibly divided up into subsections. Before a SKIP can be
considered, the proposal must fully address the issue raised in the background.
That is, a SKIP should not be used to elaborate a problem, but to provide a
solution (or provide a partial solution) for a well-articulated problem.

While a proposal may reference external documents, a proposal should stand on
its own without requiring the reader to read the external documents. The purpose
of this requirement is to ensure that SKIPs remain a digestible and
authoritative source.

### The References Section

The references section includes links to relevant sources of information.
Not all links in the document must have an accompanying reference. This is not
a works cited, but a bibliography that points to other related sources such as
relevant SKIPs, external documentation, or witty XKCD comics.

## Alternatives Considered
