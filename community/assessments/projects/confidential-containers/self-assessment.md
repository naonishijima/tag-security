<!-- cSpell:ignore Mikko Ylinen Tobin Feldman Fizthum -->
# Confidential Containers Self-assessment

December 2024

Primary authors: Mikko Ylinen and Tobin Feldman-Fizthum

## Self-assessment outline

### Table of contents

* [Metadata](#metadata)
  * [Security links](#security-links)
* [Overview](#overview)
  * [Actors and Actions](#actors-and-actions)
  * [Background](#background)
  * [Goals](#goals)
  * [Non-goals](#non-goals)
* [Self-assessment use](#self-assessment-use)
* [Security functions and features](#security-functions-and-features)
* [Project compliance](#project-compliance)
* [Secure development practices](#secure-development-practices)
* [Security issue resolution](#security-issue-resolution)
* [Appendix](#appendix)

### Metadata

|||
| -- | -- |
| Assessment Stage | Incomplete. |
| Software | <https://github.com/confidential-containers/>  |
| Security Provider | Yes, the primary function of the project is to support the security of a system. |
| Languages | Rust, Go, and Rego (for policies) |
| SBOM | Currently, not generated by the project. `go.mod` and `Cargo.toml` files are available for each release. |

#### Security links

<!-- markdown-link-check-disable -->
| Doc | url |
| -- | -- |
| Security file | Github org-wide security policy in each repo: <https://github.com/confidential-containers/.github/blob/main/SECURITY.md> |
| Design overview | <https://confidentialcontainers.org/docs/architecture/design-overview/> |
| Trust Model | <https://confidentialcontainers.org/docs/architecture/trust-model/> |
| Attestation Flow | <https://confidentialcontainers.org/docs/attestation/> |
<!-- markdown-link-check-enable -->

### Overview

Confidential Containers (CoCo) leverages [Trusted Execution Environments (TEEs)](https://en.wikipedia.org/wiki/Trusted_execution_environment)
to protect containers and data and to deliver cloud native confidential computing.

The project enables the following:

* Run unmodified applications and containers
* Support for multiple confidential hardware platforms and cloud offerings
* End-to-end attestation flow built-in

#### Background

Confidential Containers provides a set of primitives for building confidential Cloud Native
applications. The following are the key ingredients in CoCo's feature set:

* running a Kubernetes pod inside of a confidential virtual machine (VM) leveraging k8s `RuntimeClass`es
* attestable [pod configuration policies](https://github.com/kata-containers/kata-containers/blob/main/docs/how-to/how-to-use-the-kata-agent-policy.md)
  (pod Spec, CRI/runtime allow/denylist) and other init data to allow
  workload owners to control the trusted compute base (TCB) of the runtime environment
* handling of encrypted and signed container images so that related secrets (image layer
  decryption keys) and integrity protected elements (image policies, trusted public keys)
  are provisioned after a successful remote attestation and processed inside the TEEs only
* sealed Kubernetes Secrets (transparently decrypted for workloads) and protected volumes

#### Actors and Actions

[The project design overview](https://confidentialcontainers.org/docs/architecture/design-overview/) and
[a detailed attestation flow](https://confidentialcontainers.org/docs/attestation/) are maintained
on the project website and not repeated here.

#### Goals

The goal of the Confidential Containers project is to standardize confidential
computing at the pod level and simplify its consumption in Kubernetes.

This enables Kubernetes users to deploy confidential container workloads using
familiar workflows and tools without extensive knowledge of the underlying
confidential computing technologies so that code integrity and data confidentiality
are guaranteed.

Confidential Containers maps pods to confidential VMs, meaning that everything
inside a pod is within their own TEE. CoCo infrastructure components inside the
confidential VM are responsible for a) providing a secure runtime environment for
the workload and its data, b) the necessary guardrails for protecting the API between
the untrusted host (including the kubelet and CRI runtimes) and the trusted VM,
and c) remote attestation functions.

The project also seeks to build tested reference use-cases to demonstrate the
benefits of confidential computing to the ecosystem. The main focus is in trusted
software supply chains and confidential AI (in different forms).

#### Non-goals

While the project provides many building blocks to make confidential container workload
deployments easy, the users must understand the security requirements of their workloads
(such as allowing exec'ing new processes, where the logs are written to etc.).

Moreover, running workloads in TEEs does not protect confidential data from adversaries
inside the TEE (e.g., triggered by unpatched vulnerabilities in the container images).

Finally, confidential computing platforms usually do not protect against hardware
side-channels. Neither does Confidential Containers. Also, confidential computing
does not protect against denial of service. Since the untrusted host is in charge
of scheduling, it can simply not run the guest and is true for Confidential Containers
as well.

### Self-assessment use

This self-assessment is created by the Confidential Containers team to perform an internal analysis of the
project's security. It is not intended to provide a security audit of Confidential Containers, or
function as an independent assessment or attestation of Confidential Containers' security health.

This document serves to provide Confidential Containers users with an initial understanding of
Confidential Containers' security, where to find existing security documentation, Confidential Containers plans for
security, and general overview of Confidential Containers security practices, both for development of
Confidential Containers as well as security of Confidential Containers.

This document provides the CNCF TAG-Security with an initial understanding of Confidential Containers
to assist in a joint-assessment, necessary for projects under incubation.  Taken
together, this document and the joint-assessment serve as a cornerstone for if and when
Confidential Containers seeks graduation and is preparing for a security audit.

### Security functions and features

* In confidential computing careful scrutiny is required whenever information
crosses the boundary between the trusted and untrusted contexts. Secrets should not
leave the enclave without protection and entities outside of the enclave should not be
able to trigger malicious behavior inside it. In Confidential Containers there
are APIs that cross the trust boundary. The main example is the API between the `Kata
Agent` in the guest and the `Kata Shim` on the host. This API is protected with an
[OPA policy](https://github.com/kata-containers/kata-containers/blob/main/docs/how-to/how-to-use-the-kata-agent-policy.md)
running inside the guest that can block malicious requests by the host.

* Attestation is a crucial part of confidential computing and a direct requirement
of many guest operations. For example, to unpack an encrypted container image, the
guest must retrieve a secret key. Inside the guest the `confidential-data-hub`
and `attestation-agent` handle operations involving secrets and attestation.

* When using Kata Containers container, images are pulled on the worker node with the
help of a CRI runtime like containerd. The image rootfs bundles are exposed to the
guest via filesystem passthrough (virtiofs, 9p). This is not suitable for confidential
workloads because the container images are exposed to the untrusted host. With
Confidential Containers images are pulled and unpacked inside of the guest. This
requires additional components such as `image-rs` to be part of the guest rootfs and
support from the CRI runtimes to make it possible to not do the image pull themselves.

### Project compliance

* Confidential Containers does not comply with any specific security standards.
* It follows [Remote ATtestation ProcedureS (RATS) architecture](https://www.ietf.org/archive/id/draft-ietf-rats-architecture-22.html)
  principles and terminology for its attestation architecture.
* It implements [KBS attestation protocol specification](https://github.com/confidential-containers/trustee/blob/main/kbs/docs/kbs_attestation_protocol.md)
  for the confidential VM (pod) and trusted key-broker-service communication.
* Confidential Containers Trustee attestation tokens follow [EAT Attestation Result (EAR)](https://datatracker.ietf.org/doc/draft-fv-rats-ear/) message format.

### Secure development practices

* CoCo repositories are hosted on [Github](https://github.com/confidential-containers/)
and code changes are submitted through pull-requests.
* All changes must be reviewed and approved by two maintainers and the CI tests must pass
before a pull-request can be merged to `main`.
* The CI checks run by Github workflows include:
  * Developer Certificate of Origin (DCO) check to verify commits are signed-off correctly
  * code linter and formatting
  * code unit (e.g., `cargo test`) and e2e tests (e.g., running full e2e attestation with a workload in a k8s cluster on real TEEs)
* CoCo uses `dependabot` to keep its Cargo and Go module dependencies up-to-date. Furthermore,
some sub-project repositories have also enabled automatic updates of (pinned) Github actions.
* The community [communication channels](https://github.com/confidential-containers/#join-the-community) include:
  * CNCF Slack: `#confidential-containers` (and sub-project specific channels with `#confidential-containers-` prefix).
  * Weekly community meeting and sub-project meetings.
  * Blog posts and other information (such as the design/architecture docs and guides) on the [project website](https://confidentialcontainers.org/).

### Security issue resolution

* Responsible Disclosures Process and Incident Response: Confidential Containers Github
  org-wide [security policy](https://github.com/confidential-containers/.github/blob/main/SECURITY.md)
  is available for each sub-project repository. It describes how the project vulnerabilities
  are reported, analyzed, patched, and informed.

### Appendix

* Known Issues Over Time: The project has published one [Security Advisory](https://github.com/confidential-containers/trustee/security/advisories/GHSA-7jc6-j236-vvjw).
* [![OpenSSF Best Practices](https://www.bestpractices.dev/projects/5719/badge)](https://www.bestpractices.dev/projects/5719)
* Case Studies: The project maintains a list of [ADOPTERS](https://github.com/confidential-containers/confidential-containers/blob/main/ADOPTERS.md).
  Vendors/adopters have posted blogs and talked publicly (e.g., at KubeCon) about using CoCo
  in many use-cases. Moreover, the project itself runs a working group for
  [use-case driven development](https://docs.google.com/document/d/1LnGNeyUyPM61Iv4kBKFbfgmBr3RmxHYZ7Ev88obN0_E/edit?tab=t.0#heading=h.b0rnn2bw76n).
* Related Projects / Vendors: Confidential Containers are connected to a wide array
  of projects. Some projects are directly implicated in the design of Confidential
  Containers such as [Kata Containers](https://katacontainers.io/). The support from
  container runtimes (containerd/CRI-O) and their add-ons (e.g., containerd snapshotters)
  are also critical. CoCo project is complementary (by enabling extra level of security)
  to many cloud-native projects and has no overlap with any other project given its feature
  set and capabilities. A more detailed [project alignment](https://github.com/confidential-containers/confidential-containers/blob/main/alignment.md)
  is available in the community repository.
