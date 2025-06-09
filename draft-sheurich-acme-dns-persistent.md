---
title: "Automated Certificate Management Environment (ACME) Challenge for Persistent DNS TXT Record Validation"
abbrev: "ACME Persistent DNS Challenge"
category: info

docname: draft-sheurich-acme-dns-persistent-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Automated Certificate Management Environment"
keyword:
  - acme
venue:
  group: "Automated Certificate Management Environment"
  type: "Working Group"
  mail: "acme@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/acme/"
  github: "sheurich/draft-sheurich-acme-dns-persistent"
  latest: "https://sheurich.github.io/draft-sheurich-acme-dns-persistent/draft-sheurich-acme-dns-persistent.html"

author:
  - fullname: "Shiloh Heurich"
    organization: Fastly
    email: "sheurich@fastly.com"

normative:

informative:

--- abstract

This document specifies "dns‑persistent‑01", a new validation method for the Automated Certificate Management Environment (ACME) protocol defined in RFC 8555. The method allows a Certification Authority (CA) to verify domain control using a structured, long‑lived DNS TXT record. This record contains a stable value linked to the applicant's account with the CA, rather than the short‑lived tokens required by the "http‑01" and "dns‑01" challenges. Persistence enables asynchronous, multi‑tenant, and highly constrained deployments, supports wildcard certificate issuance, and remains aligned with the CA/Browser Forum Baseline Requirements (BRs).

--- middle

# Introduction

The Automated Certificate Management Environment (ACME) protocol (RFC 8555) automates certificate issuance and renewal. Its existing challenge methods — "http‑01" and "dns‑01" — deliberately expire after a short interval, which is ideal for on‑demand issuance driven by an ACME client. Some environments, however, must decouple validation from issuance or reuse a validation over an extended period.

Examples include:

- IoT deployments that cannot host an HTTP service.
- Multi‑tenant hosting platforms where the entity updating DNS differs from the certificate subscriber.
- Organizations that batch issuance operations offline.
- Scenarios requiring wildcard certificates where DNS control is proven once for an extended period.

This document defines "dns‑persistent‑01", which proves domain control through a DNS TXT resource record that can remain in place for an extended period. To prove control over a Fully Qualified Domain Name (FQDN), the applicant provisions a DNS TXT record at an "Authorization Domain Name" formed by prepending the label `_acme-challenge` to the FQDN being validated (e.g., `_acme-challenge.example.com`). The RDATA of this TXT record contains a persistent value that identifies both the CA and the applicant's specific account with that CA. The record format is based on the "authority‑token" ABNF in RFC 8659, incorporating an `issuer-domain-name` and a mandatory `accounturi` parameter (RFC 8657, Section 3) that uniquely identifies the applicant's account. This method is compatible with the CA/Browser Forum Baseline Requirements (BRs), Section 3.2.2.4.22, and is suitable for issuing wildcard certificates.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Security Considerations

TODO Security

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
