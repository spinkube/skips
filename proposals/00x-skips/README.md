# SKIP 00X - Documenting Spinkube

Summary: Describe how we document SpinKube and its projects.

Owner: Matt Fisher <matt.fisher@fermyon.com>

Impacted Projects:

- [x] documentation
- [x] spin-operator
- [x] `spin kube` plugin
- [x] runtime-class-manager
- [x] containerd-shim-spin
- [ ] Governance
- [ ] Creates a new project

Created: July 23rd, 2024

Updated: n/a

## Background

Documentation is important for a reader to understand SpinKube's value. As we move forward from
proof-of-concept to fully-fledged production system, we need to make sure that all of our
documentation is clear, concise, and up-to-date.

## Proposal

When we first started the SpinKube project, we created a website at `spinkube.dev` to host our
documentation. In addition, each of our projects has its own README file in its repository. These
README files and the website are the primary sources of documentation for SpinKube.

Moving forward, I propose that we establish a set of principles for how we document SpinKube and its
projects using the current system as a framework. These principles should be followed by all
existing and future projects in the SpinKube ecosystem.

### Prinicples

The following describes high-level principles that should be followed when documenting SpinKube:

1. We place high importance on the consistency and readability of our documentation.
1. We use a consistent writing style across all projects.
1. We structure documentation in a consistent manner.
1. We keep links and references up-to-date.
1. We ensure that documentation matches the version of the project it describes.

### Writing style

Documentation should be written in American English. This includes spelling, grammar, and
punctuation. For example, use "color" instead of "colour", "center" instead of "centre", and "gray"
instead of "grey".

Use the active voice instead of the passive voice. For example, use "SpinKube creates a new SpinApp"
instead of "A new SpinApp is created by SpinKube".

Use the present tense instead of the past tense. For example, use "SpinKube creates a new SpinApp"
instead of "SpinKube created a new SpinApp".

Use the imperative mood instead of the indicative mood. For example, use "Run `spin kube scaffold`"
instead of "You can run `spin kube scaffold`".

Use the second person instead of the first person. For example, use "You can run `spin kube
scaffold`" instead of "I can run `spin kube scaffold`".

When using pronouns in reference to a hypothetical person, such as "a user with a session cookie",
gender-neutral pronouns (they/their/them) should be used. Instead of:

- he or she... use they.
- him or her... use them.
- his or her... use their.
- his or hers... use theirs.
- himself or herself... use themselves.

Try to avoid using words that minimize the difficulty involved in a task or operation, such as
"easily", "simply", "just", "merely", "straightforward", and so on. People's experience may not
match your expectations, and they may become frustrated when they do not find a step as
"straightforward" or "simple" as it is implied to be.

Try to avoid using acronyms and jargon that are not widely known. For example, use "Kubernetes"
instead of "k8s", or "Custom Resource Definition" instead of "CRD". If you must use an acronym or
jargon, define it the first time it is used in the document, and include a link to the glossary
section of the document.

Sentences should be hard-wrapped at 100 characters. This makes it easier to read and edit the
documentation in a text editor.

In section titles, capitalize only initial words and proper nouns. For example, use "How to install
SpinKube" instead of "How to Install SpinKube".

Here are some style guidelines on commonly used terms throughout the documentation:

- SpinKube - Always capitalize the "S" and "K" in SpinKube. It is lowercase only in source code.
- email - no hyphen, lowercase.
- GitHub - uppercase "G" and "H", no space.
- Kubernetes - always capitalize the "K" in Kubernetes.
- Spin - always capitalize the "S" in Spin.
- SpinApp - always capitalize the "S" and "A" in SpinApp.

### How the documentation should be structured

Documentation should be structured in a way that makes it easy to find the information that the
reader is looking for.

When referring to a "reader", we refer to their "persona", or the role the reader asssumes as
they're reading. This could be a user, an operator, a contributor, or someone else. It is important
to know who is the intended persona because readers will have different needs, different levels of
understanding, and different expectations as they learn more about the project. For example:

- A user is someone who uses SpinKube to deploy applications. They know how Spin works and how to
  write a Spin application, but they may not be familiar with Kubernetes. They might be interested
  to know how to deploy their application to SpinKube, diagnose issues with their application, and
  how to submit bug reports.
- A contributor is someone who contributes to SpinKube and its subprojects. They are familiar with
  Spin and Kubernetes, and they want to improve SpinKube by adding new features or fixing bugs. They
  might be interested to know how to run SpinKube locally, understand SpinKube's architecture, and
  how to contribute to SpinKube.
- An operator is someone who manages SpinKube in a production environment. They are familiar with
  Kubernetes, and they want to make sure that SpinKube is running smoothly and efficiently. They
  might be interested in how to deploy SpinKube, how to monitor SpinKube, and how to scale their
  infrastructure as their needs increase.

As a general guideline, documentation should be structured as follows:

- Introduction: A brief overview of the project, its purpose, and its features.
- Getting started: Instructions on how to install SpinKube on various platforms.
- Topic guides: In-depth guides on specific topics, such as how to deploy an application, how to
  monitor application traffic, using SpinKube features, and so on.
- Reference: technical reference for APIs and other aspects of SpinKube's machinery.
- Contributing: Information on how to contribute to the project, such as how to report bugs, how to
  request features, how to submit a pull request, and so on.

Note that the contributing guide should not describe how to compile and run each project locally.
This information should be in the README file of each project.

### Links and references

Links and references should be kept up-to-date. If a link or reference is no longer valid, it should
be removed or updated. This includes links to other documents, websites, and so on.

When referring to another document outside of SpinKube github organization, such as a project README
file, a blog post, or a website, consider its permanence. If the document is likely to change or
move, consider copying the relevant information (and attribute the source), or link to a more stable
source.

### Versioning

The documentation should match the version of the project it describes. For example, if the
quickstart guide for SpinKube installs version 0.2.0, the rest of the documentation should include
instructions for version 0.2.0. Features that are available in later versions of SpinKube (such as a
new feature in a development branch) should not be included in the documentation because the reader
will not be able to use those features until SpinKube version 0.3.0.

Note that this document does not propose a versioning scheme for spinkube.dev itself. That is a
separate issue that should be addressed in a separate SKIP.

## References

The following references were consulted in the creation of this SKIP:

- [Django documentation](https://docs.djangoproject.com/)

## Alternatives Considered

No alternatives were considered.
