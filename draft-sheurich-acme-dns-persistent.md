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

This document specifies "dns-persistent-01", a new validation method for the Automated Certificate Management Environment (ACME) protocol. This method allows a Certification Authority (CA) to verify an applicant's control over a domain name by confirming the presence of a persistent, structured DNS TXT record. This record contains a stable value that identifies the applicant's account with the CA, rather than the short-lived tokens required by the "http-01" and "dns-01" challenges defined in RFC 8555. The persistence of this record facilitates asynchronous issuance workflows, simplifies management in multi-tenant environments, supports wildcard certificate issuance, and is designed to be compatible with the CA/Browser Forum Baseline Requirements.

--- middle

# Introduction

The Automated Certificate Management Environment (ACME) protocol {{RFC8555}} defines mechanisms for automating certificate issuance and domain validation. The existing challenge methods, "http-01" and "dns-01", rely on short-lived, single-use tokens. While effective for many use cases, this model presents challenges for environments that must decouple the act of domain validation from certificate issuance, or where DNS updates are managed by a different entity than the one requesting the certificate.

Examples include:

- Internet of Things (IoT) deployments where devices may not be able to host an HTTP service or coordinate DNS updates in real-time.
- Multi-tenant hosting platforms where the entity managing the DNS zone is distinct from the tenant subscribing to the certificate.
- Organizations that wish to pre-validate domains and batch issuance operations offline or at a later time.
- Scenarios requiring wildcard certificates where domain control is proven once and reused over an extended period.

This document defines a new ACME challenge type, "dns-persistent-01". This method proves control over a Fully Qualified Domain Name (FQDN) by having the applicant provision a DNS TXT record at a specific name derived from the FQDN. The content (RDATA) of this record is a persistent value that identifies both the CA and the applicant's specific account with that CA.

The record format is based on the "authority-token" ABNF in {{RFC8659}}, incorporating an issuer-domain-name and a mandatory accounturi parameter {{RFC8657}} that uniquely identifies the applicant's account. This approach aligns with the validation method described in Section 3.2.2.4.22 of the CA/Browser Forum Baseline Requirements and is suitable for issuing wildcard certificates.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# The "dns-persistent-01" Challenge

The "dns-persistent-01" challenge allows an ACME client to demonstrate control over an FQDN by proving it can provision a DNS TXT record containing a specific, persistent value. The value links the FQDN to the client's ACME account with the CA.

When an ACME client accepts a "dns-persistent-01" challenge, it proves control by provisioning a DNS TXT record at the Authorization Domain Name.

# Challenge Response and Verification

To respond to the challenge, the ACME client provisions a DNS TXT record for the Authorization Domain Name being validated. The Authorization Domain Name is formed by prepending the label _acme-challenge to the FQDN for which authorization is being sought.

The RDATA of this TXT record MUST fulfill the following requirements:

The RDATA value MUST conform to the issue-value syntax as defined in {{RFC8659}}, Section 4.2.

The issue-value MUST contain an issuer-domain-name parameter. The value of this parameter MUST be an Issuer Domain Name that the CA discloses in its Certificate Policy (CP) and/or Certification Practices Statement (CPS).

The issue-value MUST contain an accounturi parameter. The value of this parameter MUST be a unique URI identifying the account of the applicant which requested the validation, as described in {{RFC8657}}, Section 3.

For example, if the ACME client is requesting validation for the FQDN example.com from a CA that uses authority.example as its Issuer Domain Name, it would provision the following DNS TXT record:

_acme-challenge.example.com. IN TXT "authority.example; accounturi=https://authority.example/acct/123"

The ACME server verifies the challenge by performing a DNS lookup for the TXT record at the Authorization Domain Name and checking that its RDATA conforms to the required structure and contains both the server's issuer-domain-name and the accounturi of the requesting client.

# Wildcard and Suffix Validation

This validation method is suitable for validating Wildcard Domain Names (e.g., *.example.com).

Furthermore, once an FQDN has been successfully validated using this method, the CA MAY also consider this validation sufficient for issuing certificates for any other FQDNs that end with all the Domain Labels of the validated FQDN. For example, a successful validation for example.com may also be used to issue a certificate for www.example.com.

Validation Data Reuse
This validation method is explicitly designed for persistence. If the DNS TXT record has a Time-to-Live (TTL) that is less than the CA's defined validation data reuse period, then the CA MUST consider the reuse period for this validation to be the greater of either the record's TTL or 8 hours.

# Security Considerations

The persistence of the validation record means that its compromise could allow an attacker to obtain certificates for the validated domain until the record is removed or changed. Clients SHOULD protect the integrity of their DNS records accordingly.

CAs performing validations using the "dns-persistent-01" method MUST implement Multi-Perspective Issuance Corroboration as specified in Section 3.2.2.9 of the CA/Browser Forum Baseline Requirements. To count as a corroborating validation, each network perspective used by the CA MUST observe the same challenge information (i.e., the same TXT record RDATA) as the primary network perspective.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
