---
title: "Automated Certificate Management Environment (ACME) Challenge for Persistent DNS TXT Record Validation"
abbrev: "ACME Persistent DNS Challenge"
category: std

docname: draft-sheurich-acme-dns-persist-latest
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
  github: "sheurich/draft-sheurich-acme-dns-persist"
  latest: "https://sheurich.github.io/draft-sheurich-acme-dns-persist/draft-sheurich-acme-dns-persist.html"

author:
 -
   fullname: "Shiloh Heurich"
   organization: Fastly
   email: "sheurich@fastly.com"

normative:
  RFC1034:
  RFC1035:
  RFC8555:
  RFC8657:
  RFC8659:

informative:
  CABF-BR:
    title: "Baseline Requirements for the Issuance and Management of Publicly-Trusted Certificates"
    author:
      org: "CA/Browser Forum"
    date: 2024
    target: "https://cabforum.org/baseline-requirements-documents/"

--- abstract

This document specifies "dns-persist-01", a new validation method for the Automated Certificate Management Environment (ACME) protocol. This method allows a Certification Authority (CA) to verify control over a domain by confirming the presence of a persistent DNS TXT record containing CA and account identification information. This method is particularly suited for environments where traditional challenge methods are impractical, such as IoT deployments, multi-tenant platforms, and scenarios requiring batch certificate operations. The validation method is designed to fulfill requirements specified in the CA/Browser Forum Baseline Requirements for persistent DNS TXT record validation.

--- middle

# Introduction {#introduction}

The Automated Certificate Management Environment (ACME) protocol {{RFC8555}} defines mechanisms for automating certificate issuance and domain validation. The existing challenge methods, "http-01" and "dns-01", require real-time interaction between the ACME client and the domain's infrastructure during the validation process. While effective for many use cases, these methods present challenges in certain deployment scenarios.

Examples include:

- Internet of Things (IoT) deployments where devices may not be able to host an HTTP service or coordinate DNS updates in real-time.
- Edge compute and multi-tenant hosting platforms where the entity managing the DNS zone is distinct from the tenant subscribing to the certificate.
- Organizations that wish to pre-validate domains and batch issuance operations offline or at a later time.
- Scenarios requiring wildcard certificates where domain control is proven once and reused over an extended period.
- Environments with strict change management processes where DNS modifications require approval workflows.

This document defines a new ACME challenge type, "dns-persist-01". This method proves control over a Fully Qualified Domain Name (FQDN) by confirming the presence of a persistent DNS TXT record containing CA and account identification information.

The record format is based on the "issue-value" syntax from {{RFC8659}}, incorporating an issuer-domain-name and a mandatory accounturi parameter {{RFC8657}} that uniquely identifies the applicant's account. This design provides strong binding between the domain, the CA, and the specific account requesting validation.

## Relationship to CA/Browser Forum Requirements {#relationship-to-cabf}

This validation method is designed to align with industry practices for persistent DNS TXT record validation, such as those described in the CA/Browser Forum Baseline Requirements {{CABF-BR}}. The following key requirements are incorporated directly into this specification:

1. Use of the "_validation-persist" DNS label as the Authorization Domain Name prefix
2. Multi-Perspective Validation requirements (see Section 4.1)
3. Validation Data Reuse periods based on TTL values (see Section 4.2)
4. Explicit account binding through the accounturi parameter

Certification Authorities implementing this method MUST comply with this specification and MAY additionally need to comply with other applicable industry requirements depending on their trust program participation.

# Conventions and Definitions {#conventions-and-definitions}

{::boilerplate bcp14}

**Authorization Domain Name**: The domain name formed by prepending the DNS TXT Record Persistent DCV Domain Label to the domain name being validated.

**DNS TXT Record Persistent DCV Domain Label**: The label "_validation-persist" as specified in this document. This label is consistent with industry practices for persistent domain validation.

**Issuer Domain Name**: A domain name disclosed by the CA in Section 4.2 of the CA's Certificate Policy and/or Certification Practices Statement to identify the CA for the purposes of this validation method.

**Validation Data Reuse Period**: The period during which a CA may rely on validation data, as defined by the CA's practices and applicable requirements.

# The "dns-persist-01" Challenge {#dns-persist-01-challenge}

The "dns-persist-01" challenge allows an ACME client to demonstrate control over an FQDN by proving it can provision a DNS TXT record containing specific, persistent validation information. The validation information links the FQDN to both the Certificate Authority performing the validation and the specific ACME account requesting the validation.

When an ACME client accepts a "dns-persist-01" challenge, it proves control by provisioning a DNS TXT record at the Authorization Domain Name. Unlike the existing "dns-01" challenge, this record is designed to persist and may be reused for multiple certificate issuances over an extended period.

## Challenge Object {#challenge-object}

The challenge object for "dns-persist-01" contains the following fields:

- **type** (required, string): The string "dns-persist-01"
- **url** (required, string): The URL to which a response can be posted
- **status** (required, string): The status of this challenge
- **issuer-domain-name** (required, string): The Issuer Domain Name that the client MUST include in the DNS TXT record

Example challenge object:

~~~json
{
  "type": "dns-persist-01",
  "url": "https://ca.example/acme/authz/1234/0",
  "status": "pending",
  "issuer-domain-name": "authority.example"
}
~~~

# Challenge Response and Verification {#challenge-response-and-verification}

To respond to the challenge, the ACME client provisions a DNS TXT record for the Authorization Domain Name being validated. The Authorization Domain Name is formed by prepending the label "_validation-persist" to the domain name being validated.

For example, if the domain being validated is "example.com", the Authorization Domain Name would be "_validation-persist.example.com".

The RDATA of this TXT record MUST fulfill the following requirements:

1. The RDATA value MUST conform to the issue-value syntax as defined in {{RFC8659}}, Section 4.

2. The `issuer-domain-name` portion of the issue-value MUST be the Issuer Domain Name provided by the CA in the challenge object.

3. The issue-value MUST contain an accounturi parameter. The value of this parameter MUST be a unique URI identifying the account of the applicant which requested the validation, constructed according to {{RFC8657}}, Section 3.

4. The issue-value MAY contain a `policy` parameter. If present, this parameter modifies the validation scope. The `policy` parameter follows the `key=value` syntax. The policy parameter key and its defined values MUST be treated as case-insensitive. The following values for the `policy` parameter are defined with respect to subdomain and wildcard validation:

- `policy=specific-subdomains-only`: If this value is present, the CA MAY consider this validation sufficient for issuing certificates for the validated FQDN and for specific subdomains of the validated FQDN, as described in the "Subdomain Certificate Validation" section. This policy value explicitly does NOT authorize wildcard certificates.

- `policy=wildcard-allowed`: If this value is present, the CA MAY consider this validation sufficient for issuing certificates for the validated FQDN, for specific subdomains of the validated FQDN (as covered by wildcard scope or specific subdomain validation rules), and for wildcard certificates (e.g., `*.example.com`). See "Wildcard Certificate Validation" and "Subdomain Certificate Validation" sections.

CAs MUST parse the issue-value string by first separating semicolon-separated fields, then parsing each field as either a domain name (for the issuer-domain-name) or a key=value pair (for accounturi and policy parameters).

If the `policy` parameter is absent, the validation MUST only apply to the specific FQDN for which the record is set. If the `policy` parameter is present but contains a value not defined in this specification (i.e., any value other than `specific-subdomains-only` or `wildcard-allowed`), CAs MUST ignore the entire policy parameter and treat the validation as applying only to the specific FQDN. CAs MUST ignore unknown parameter keys not defined in this specification.

For example, if the ACME client is requesting validation for the FQDN "example.com" from a CA that uses "authority.example" as its Issuer Domain Name, and the client's account URI is "https://ca.example/acct/123", and wants to allow only specific subdomains, it might provision:

~~~
_validation-persist.example.com. IN TXT "authority.example; accounturi=https://ca.example/acct/123; policy=specific-subdomains-only"
~~~

If no policy parameter is included, the record defaults to FQDN-only validation:

~~~
_validation-persist.example.com. IN TXT "authority.example; accounturi=https://ca.example/acct/123"
~~~

The ACME server verifies the challenge by performing a DNS lookup for the TXT record at the Authorization Domain Name and checking that its RDATA conforms to the required structure and contains both the correct issuer-domain-name and a valid accounturi for the requesting account. The server also interprets any `policy` parameter values according to this specification.

## Multi-Perspective Validation {#multi-perspective-validation}

CAs performing validations using the "dns-persist-01" method MUST implement Multi-Perspective Issuance Corroboration. To count as corroborating, a Network Perspective MUST observe the same challenge information as the Primary Network Perspective.

## Validation Data Reuse and TTL Handling {#validation-data-reuse-and-ttl}

This validation method is explicitly designed for persistence and reuse. If the DNS TXT record has a Time-to-Live (TTL) that is less than the CA's defined validation data reuse period, then the CA MUST consider the validation data reuse period to be equal to the TTL or 8 hours, whichever is greater.

CAs MAY reuse validation data obtained through this method for the duration of their validation data reuse period, subject to the TTL constraints described above.

# Wildcard Certificate Validation {#wildcard-certificate-validation}

This validation method is suitable for validating Wildcard Domain Names (e.g., *.example.com). To authorize a wildcard certificate for a domain, a single DNS TXT record placed at the Authorization Domain Name for the base domain MUST be used. This TXT record MUST include the `policy=wildcard-allowed` parameter value.

When such a record is present (i.e., containing `policy=wildcard-allowed`), it can validate the base domain, specific subdomains, and wildcard certificates for that domain. For example, a TXT record at `_validation-persist.example.com` containing `policy=wildcard-allowed` can validate certificates for `example.com`, `www.example.com`, and `*.example.com`. If the `policy` parameter is absent or set to `specific-subdomains-only`, the validation is not sufficient for `*.example.com`.

# Subdomain Certificate Validation {#subdomain-certificate-validation}

If an FQDN has been successfully validated using this method, the CA MAY also consider this validation sufficient for issuing certificates for other FQDNs that are subdomains of the validated FQDN, under the following conditions:

- The persistent DNS TXT record MUST include either `policy=specific-subdomains-only` or `policy=wildcard-allowed`.

To determine which subdomains are permitted, the FQDN for which the persistent TXT record exists (the "validated FQDN") must appear as the exact suffix of the FQDN for which a certificate is requested (the "requested FQDN"). For example, if `dept.example.com` is the validated FQDN, a certificate for `server.dept.example.com` is permitted because `dept.example.com` is its suffix. This subdomain validation differs from the wildcard policy primarily in scope:

- With `policy=specific-subdomains-only` on the validated FQDN, certificates can be issued for the validated FQDN itself and its specific subdomains (per the suffix rule), but explicitly NOT for a wildcard covering those subdomains (e.g., `*.dept.example.com`).

- With `policy=wildcard-allowed` on the validated FQDN, certificates can be issued for the validated FQDN, its specific subdomains (per the suffix rule), AND for a wildcard covering those subdomains (e.g., `*.dept.example.com`). The "Wildcard Certificate Validation" section further details how `policy=wildcard-allowed` is typically used on a base domain to authorize certificates like `*.example.com`.

If the `policy` parameter is absent, or if it is present but does not authorize subdomain validation (e.g., a future policy value for a different purpose), this validation MUST NOT be considered sufficient for issuing certificates for subdomains.

**Security Warning**: Enabling subdomain validation via `policy=specific-subdomains-only` or `policy=wildcard-allowed` creates significant security implications. Organizations using this feature MUST carefully control subdomain delegation and monitor for unauthorized subdomains. These policy values serve as the explicit mechanism for domain owners to opt-in to broader validation scopes.

For example, validation of "dept.example.com" would authorize certificates for "server.dept.example.com" but not for "dept.example.org" due to the suffix rule. Without an appropriate policy parameter, validation would only authorize certificates for "dept.example.com" itself.

# Security Considerations {#security-considerations}

## Persistent Record Risks {#persistent-record-risks}

The persistence of validation records creates extended windows of vulnerability compared to traditional ACME challenge methods. If an attacker gains control of a DNS zone containing persistent validation records, they can potentially obtain certificates for the validated domains until the validation records are removed or modified.

Clients SHOULD protect validation records through appropriate DNS security measures, including:

- Using DNS providers with strong authentication and access controls
- Implementing DNS security extensions (DNSSEC) where possible
- Monitoring DNS zones for unauthorized changes
- Regularly reviewing and rotating validation records

## Account Binding Security {#account-binding-security}

The `accounturi` parameter provides strong binding between domain validation and specific ACME accounts. However, this binding depends on the security of the ACME account itself. If an ACME account is compromised, attackers may be able to obtain certificates for domains with persistent validation records associated with that account.

More specifically, if the ACME account key is compromised, an attacker can immediately use any pre-existing `dns-persist-01` authorizations for that account to issue certificates without requiring any additional access to DNS infrastructure. This represents a significant risk given the persistent nature of these validation records.

CAs SHOULD implement robust account security measures, including:

- Strong authentication requirements for ACME accounts
- Account activity monitoring and anomaly detection
- Rapid account revocation capabilities
- Regular account security reviews
- Account key rotation policies and procedures

Clients SHOULD protect their ACME account keys with the same level of security as they would protect private keys for high-value certificates.

## Subdomain Validation Risks {#subdomain-validation-risks}

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

## Cross-CA Validation Reuse {#cross-ca-validation-reuse}

The persistent nature of validation records raises concerns about potential reuse across different Certificate Authorities. While the issuer-domain-name parameter is designed to prevent such reuse, implementations MUST carefully validate that the issuer-domain-name in the DNS record matches the CA's disclosed Issuer Domain Name.

## Record Tampering and Integrity {#record-tampering-and-integrity}

DNS records are generally not authenticated end-to-end, making them potentially vulnerable to tampering. CAs SHOULD implement additional integrity checks where possible and consider the overall security posture of the DNS infrastructure when relying on persistent validation records.

Additionally, if an attacker could compromise the DNS infrastructure for a CA's `issuer-domain-name` (e.g., `authority.example`), they could potentially create confusion about CA identity, although this represents a systemic risk that extends beyond this specific validation method. CAs SHOULD protect their issuer domain names with appropriate DNS security measures.

# IANA Considerations {#iana-considerations}

## ACME Validation Methods Registry {#acme-validation-methods-registry}

IANA is requested to register the following entry in the "ACME Validation Methods" registry:

- **Label**: dns-persist-01
- **Identifier Type**: dns
- **ACME**: Y
- **Reference**: This document

# Implementation Considerations {#implementation-considerations}

## CA Implementation Guidelines {#ca-implementation-guidelines}

Certificate Authorities implementing this validation method should consider:

- Establishing clear policies for Issuer Domain Name disclosure in Certificate Policies and Certification Practice Statements
- Implementing robust multi-perspective validation infrastructure
- Developing procedures for handling validation record TTL variations
- Creating account security monitoring and incident response procedures
- Providing clear documentation for clients on proper record construction

## Client Implementation Guidelines {#client-implementation-guidelines}

ACME clients implementing this validation method should consider:

- Implementing secure DNS record management practices
- Providing clear user interfaces for managing persistent validation records
- Implementing validation record monitoring and alerting
- Designing appropriate error handling for validation failures
- Considering the security implications of persistent records in their threat models

## DNS Provider Considerations {#dns-provider-considerations}

DNS providers supporting this validation method should consider:

- Implementing appropriate access controls for validation record management
- Providing audit logging for validation record changes
- Supporting reasonable TTL values for validation records
- Considering dedicated interfaces or APIs for ACME validation record management

# Examples {#examples}

## Basic Validation Example (FQDN Only) {#basic-validation-example}

For validation of "example.com" by a CA using "authority.example" as its Issuer Domain Name, where the validation should only apply to "example.com":

1. CA provides challenge object:

~~~json
{
  "type": "dns-persist-01",
  "url": "https://ca.example/acme/authz/1234/0",
  "status": "pending",
  "issuer-domain-name": "authority.example"
}
~~~

2. Client provisions DNS TXT record (note the absence of a `policy` parameter for scope):

~~~
_validation-persist.example.com. IN TXT "authority.example; accounturi=https://ca.example/acct/123"
~~~

3. CA validates the record through multi-perspective DNS queries. This validation is sufficient only for "example.com".

## Specific Subdomain Validation Example {#specific-subdomain-validation-example}

For validation of "example.com" and its specific subdomains (e.g., "www.example.com", "api.example.com"), but NOT for "*.example.com", by a CA using "authority.example":

1. Same challenge object format as above.

2. Client provisions DNS TXT record including `policy=specific-subdomains-only`:

~~~
_validation-persist.example.com. IN TXT "authority.example; accounturi=https://ca.example/acct/123; policy=specific-subdomains-only"
~~~

3. CA validates the record. This validation authorizes certificates for "example.com" and specific subdomains like "www.example.com", but not for "*.example.com".

## Wildcard Validation Example {#wildcard-validation-example}

For validation of "*.example.com" (which also validates "example.com" and specific subdomains like "www.example.com") by a CA using "authority.example" as its Issuer Domain Name:

1. Same challenge object format as above.

2. Client provisions DNS TXT record at the base domain's Authorization Domain Name, including `policy=wildcard-allowed`:

~~~
_validation-persist.example.com. IN TXT "authority.example; accounturi=https://ca.example/acct/123; policy=wildcard-allowed"
~~~

3. CA validates the record through multi-perspective DNS queries. This validation authorizes certificates for "example.com", "*.example.com", and specific subdomains like "www.example.com".

--- back

# Acknowledgments {:numbered="false"}

The author would like to acknowledge the CA/Browser Forum for developing the Baseline Requirements that motivated this specification, and the ACME Working Group for their ongoing work on certificate automation protocols.

Thanks to the contributors and reviewers who provided feedback on early versions of this document.
