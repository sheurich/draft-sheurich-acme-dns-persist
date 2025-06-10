---
title: "Automated Certificate Management Environment (ACME) Challenge for Persistent DNS TXT Record Validation"
abbrev: "ACME Persistent DNS Challenge"
category: info

docname: draft-sheurich-acme-dns-persistent-latest
submissiontype: IETF
number:
date: 2025-06-10
consensus: true
v: 3
area: "Security"
workgroup: "Automated Certificate Management Environment"
keyword:
  - acme
  - dns
  - validation
  - persistent
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
  RFC1034:
    title: "Domain Names - Concepts and Facilities"
    date: 1987-11
    target: https://www.rfc-editor.org/info/rfc1034
  RFC1035:
    title: "Domain Names - Implementation and Specification"
    date: 1987-11
    target: https://www.rfc-editor.org/info/rfc1035
  RFC2119:
    title: "Key words for use in RFCs to Indicate Requirement Levels"
    date: 1997-03
    target: https://www.rfc-editor.org/info/rfc2119
  RFC8174:
    title: "Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words"
    date: 2017-05
    target: https://www.rfc-editor.org/info/rfc8174
  RFC8555:
    title: "Automatic Certificate Management Environment (ACME)"
    date: 2019-03
    target: https://www.rfc-editor.org/info/rfc8555
  RFC8657:
    title: "Certification Authority Authorization (CAA) Record Extensions for Account and Method Binding"
    date: 2019-11
    target: https://www.rfc-editor.org/info/rfc8657
  RFC8659:
    title: "DNS-Based Authentication of Named Entities (DANE) Bindings for Unsecured DNS"
    date: 2019-11
    target: https://www.rfc-editor.org/info/rfc8659

informative:

--- abstract

This document specifies "dns-persistent-01", a new validation method for the Automated Certificate Management Environment (ACME) protocol. This method allows a Certification Authority (CA) to verify control over a domain by confirming the presence of a persistent DNS TXT record containing CA and account identification information. This method is particularly suited for environments where traditional challenge methods are impractical, such as IoT deployments, multi-tenant platforms, and scenarios requiring batch certificate operations. The validation method is designed to fulfill requirements specified in the CA/Browser Forum Baseline Requirements for persistent DNS TXT record validation.

--- middle

# Introduction

The Automated Certificate Management Environment (ACME) protocol {{RFC8555}} defines mechanisms for automating certificate issuance and domain validation. The existing challenge methods, "http-01" and "dns-01", require real-time interaction between the ACME client and the domain's infrastructure during the validation process. While effective for many use cases, these methods present challenges in certain deployment scenarios.

Examples include:

- Internet of Things (IoT) deployments where devices may not be able to host an HTTP service or coordinate DNS updates in real-time.
- Multi-tenant hosting platforms where the entity managing the DNS zone is distinct from the tenant subscribing to the certificate.
- Organizations that wish to pre-validate domains and batch issuance operations offline or at a later time.
- Scenarios requiring wildcard certificates where domain control is proven once and reused over an extended period.
- Environments with strict change management processes where DNS modifications require approval workflows.

This document defines a new ACME challenge type, "dns-persistent-01". This method proves control over a Fully Qualified Domain Name (FQDN) by having the applicant provision a DNS TXT record at a specific subdomain that contains persistent validation information linking the domain to both the Certificate Authority and the applicant's account.

The record format is based on the "issue-value" syntax from {{RFC8659}}, incorporating an issuer-domain-name and a mandatory accounturi parameter {{RFC8657}} that uniquely identifies the applicant's account. This design provides strong binding between the domain, the CA, and the specific account requesting validation.

## Relationship to CA/Browser Forum Requirements

This validation method is designed to fulfill the requirements specified in Section 3.2.2.4.22 of the CA/Browser Forum Baseline Requirements for persistent DNS TXT record validation. Certification Authorities implementing this method MUST comply with both this specification and the applicable Baseline Requirements, including requirements for Multi-Perspective Issuance Corroboration and validation data reuse periods.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

**Authorization Domain Name**: The domain name formed by prepending the DNS TXT Record Persistent DCV Domain Label to the domain name being validated.

**DNS TXT Record Persistent DCV Domain Label**: The label "_validation-persist" as defined in Appendix A of the CA/Browser Forum Baseline Requirements.

**Issuer Domain Name**: A domain name disclosed by the CA in Section 4.2 of the CA's Certificate Policy and/or Certification Practices Statement to identify the CA for the purposes of this validation method.

**Validation Data Reuse Period**: The period during which a CA may rely on validation data, as defined by the CA's practices and applicable requirements.

# The "dns-persistent-01" Challenge

The "dns-persistent-01" challenge allows an ACME client to demonstrate control over an FQDN by proving it can provision a DNS TXT record containing specific, persistent validation information. The validation information links the FQDN to both the Certificate Authority performing the validation and the specific ACME account requesting the validation.

When an ACME client accepts a "dns-persistent-01" challenge, it proves control by provisioning a DNS TXT record at the Authorization Domain Name. Unlike the existing "dns-01" challenge, this record is designed to persist and may be reused for multiple certificate issuances over an extended period.

## Challenge Object

The challenge object for "dns-persistent-01" contains the following fields:

- **type** (required, string): The string "dns-persistent-01"
- **url** (required, string): The URL to which a response can be posted
- **status** (required, string): The status of this challenge
- **issuer-domain-name** (required, string): The Issuer Domain Name that the client MUST include in the DNS TXT record

Example challenge object:

```json
{
  "type": "dns-persistent-01",
  "url": "https://ca.example/acme/authz/1234/0",
  "status": "pending",
  "issuer-domain-name": "authority.example"
}
```

# Challenge Response and Verification

To respond to the challenge, the ACME client provisions a DNS TXT record for the Authorization Domain Name being validated. The Authorization Domain Name is formed by prepending the label "_validation-persist" to the domain name being validated.

For example, if the domain being validated is "example.com", the Authorization Domain Name would be "_validation-persist.example.com".

The RDATA of this TXT record MUST fulfill the following requirements:

1. The RDATA value MUST conform to the issue-value syntax as defined in {{RFC8659}}, Section 4.2.

2. The issue-value MUST contain an issuer-domain-name parameter. The value of this parameter MUST be the Issuer Domain Name provided by the CA in the challenge object.

3. The issue-value MUST contain an accounturi parameter. The value of this parameter MUST be a unique URI identifying the account of the applicant which requested the validation, constructed according to {{RFC8657}}, Section 3.

For example, if the ACME client is requesting validation for the FQDN "example.com" from a CA that uses "authority.example" as its Issuer Domain Name, and the client's account URI is "https://ca.example/acct/123", it would provision the following DNS TXT record:

```
_validation-persist.example.com. IN TXT "authority.example; accounturi=https://ca.example/acct/123"
```

The ACME server verifies the challenge by performing a DNS lookup for the TXT record at the Authorization Domain Name and checking that its RDATA conforms to the required structure and contains both the correct issuer-domain-name and a valid accounturi for the requesting account.

## Multi-Perspective Validation

CAs performing validations using the "dns-persistent-01" method MUST implement Multi-Perspective Issuance Corroboration. To count as corroborating, a Network Perspective MUST observe the same challenge information as the Primary Network Perspective.

## Validation Data Reuse and TTL Handling

This validation method is explicitly designed for persistence and reuse. If the DNS TXT record has a Time-to-Live (TTL) that is less than the CA's defined validation data reuse period, then the CA MUST consider the validation data reuse period to be equal to the TTL or 8 hours, whichever is greater.

CAs MAY reuse validation data obtained through this method for the duration of their validation data reuse period, subject to the TTL constraints described above.

# Wildcard Certificate Validation

This validation method is suitable for validating Wildcard Domain Names (e.g., *.example.com). A single DNS TXT record placed at the Authorization Domain Name can validate both the base domain and wildcard certificates for that domain.

For example, a TXT record at "_validation-persist.example.com" can validate certificates for both "example.com" and "*.example.com".

# Subdomain Certificate Validation

Once an FQDN has been successfully validated using this method, the CA MAY also consider this validation sufficient for issuing certificates for other FQDNs that are subdomains of the validated FQDN, provided that all the Domain Labels of the validated FQDN appear as a suffix in the certificate subject.

**Security Warning**: This capability creates significant security implications. Organizations using this feature MUST carefully control subdomain delegation and monitor for unauthorized subdomains. CAs SHOULD provide mechanisms for domain owners to opt out of subdomain validation or limit its scope.

For example, validation of "dept.example.com" could authorize certificates for "server.dept.example.com" but would not authorize certificates for "other-dept.example.com".

# Security Considerations

## Persistent Record Risks

The persistence of validation records creates extended windows of vulnerability compared to traditional ACME challenge methods. If an attacker gains control of a DNS zone containing persistent validation records, they can potentially obtain certificates for the validated domains until the validation records are removed or modified.

Clients SHOULD protect validation records through appropriate DNS security measures, including:

- Using DNS providers with strong authentication and access controls
- Implementing DNS security extensions (DNSSEC) where possible
- Monitoring DNS zones for unauthorized changes
- Regularly reviewing and rotating validation records

## Account Binding Security

The accounturi parameter provides strong binding between domain validation and specific ACME accounts. However, this binding depends on the security of the ACME account itself. If an ACME account is compromised, attackers may be able to obtain certificates for domains with persistent validation records associated with that account.

CAs SHOULD implement robust account security measures, including:

- Strong authentication requirements for ACME accounts
- Account activity monitoring and anomaly detection
- Rapid account revocation capabilities
- Regular account security reviews

## Subdomain Validation Risks

The ability to issue certificates for subdomains of validated FQDNs creates significant security risks, particularly in environments with subdomain delegation or where subdomains may be controlled by different entities.

Potential risks include:

- Subdomain takeover attacks where abandoned subdomains are claimed by attackers
- Unauthorized certificate issuance for subdomains controlled by different organizations
- Confusion about which entity has authority over specific subdomains

Organizations considering the use of subdomain validation MUST:

- Maintain strict control over subdomain delegation
- Implement monitoring for subdomain creation and changes
- Consider limiting subdomain validation to specific, controlled scenarios
- Provide clear governance policies for subdomain certificate authority

## Cross-CA Validation Reuse

The persistent nature of validation records raises concerns about potential reuse across different Certificate Authorities. While the issuer-domain-name parameter is designed to prevent such reuse, implementations MUST carefully validate that the issuer-domain-name in the DNS record matches the CA's disclosed Issuer Domain Name.

## Record Tampering and Integrity

DNS records are generally not authenticated end-to-end, making them potentially vulnerable to tampering. CAs SHOULD implement additional integrity checks where possible and consider the overall security posture of the DNS infrastructure when relying on persistent validation records.

# IANA Considerations

## ACME Validation Methods Registry

IANA is requested to register the following entry in the "ACME Validation Methods" registry:

- **Label**: dns-persistent-01
- **Identifier Type**: dns
- **ACME**: Y
- **Reference**: [This document]

# Implementation Considerations

## CA Implementation Guidelines

Certificate Authorities implementing this validation method should consider:

- Establishing clear policies for Issuer Domain Name disclosure in Certificate Policies and Certification Practice Statements
- Implementing robust multi-perspective validation infrastructure
- Developing procedures for handling validation record TTL variations
- Creating account security monitoring and incident response procedures
- Providing clear documentation for clients on proper record construction

## Client Implementation Guidelines

ACME clients implementing this validation method should consider:

- Implementing secure DNS record management practices
- Providing clear user interfaces for managing persistent validation records
- Implementing validation record monitoring and alerting
- Designing appropriate error handling for validation failures
- Considering the security implications of persistent records in their threat models

## DNS Provider Considerations

DNS providers supporting this validation method should consider:

- Implementing appropriate access controls for validation record management
- Providing audit logging for validation record changes
- Supporting reasonable TTL values for validation records
- Considering dedicated interfaces or APIs for ACME validation record management

# Examples

## Basic Validation Example

For validation of "example.com" by a CA using "authority.example" as its Issuer Domain Name:

1. CA provides challenge object:
```json
{
  "type": "dns-persistent-01",
  "url": "https://ca.example/acme/authz/1234/0",
  "status": "pending",
  "issuer-domain-name": "authority.example"
}
```

2. Client provisions DNS TXT record:
```
_validation-persist.example.com. IN TXT "authority.example; accounturi=https://ca.example/acct/123"
```

3. CA validates the record through multi-perspective DNS queries

## Wildcard Validation Example

For validation of "*.example.com":

1. Same challenge object format as above
2. Same DNS TXT record placement at "_validation-persist.example.com"
3. Validation authorizes both "example.com" and "*.example.com" certificates

--- back

# Acknowledgments
{:numbered="false"}

The author would like to acknowledge the CA/Browser Forum for developing the Baseline Requirements that motivated this specification, and the ACME Working Group for their ongoing work on certificate automation protocols.

Thanks to the contributors and reviewers who provided feedback on early versions of this document.
