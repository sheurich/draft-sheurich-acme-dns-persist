---
title: "Automated Certificate Management Environment (ACME) Challenge for Persistent DNS TXT Record Validation"
abbrev: "ACME Persistent DNS Challenge"
category: std
docname: draft-sheurich-acme-dns-persist-latest
number:
date: 2025-06-23
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

author:
 -
   name: "Shiloh Heurich"
   org: Fastly
   email: "sheurich@fastly.com"
 -
   name: "Henry Birge-Lee"
   org: "Princeton University"
   email: "birgelee@princeton.edu"

informative:
  CABF-BR:
    title: "Baseline Requirements for the Issuance and Management of Publicly-Trusted Certificates"
    author:
      org: "CA/Browser Forum"
    date: 2024
    target: "https://cabforum.org/baseline-requirements-documents/"
  draft-sheth-identifiers-dns:
    target: https://datatracker.ietf.org/doc/draft-sheth-identifiers-dns/
    title: "Best Practices for Persistent References in DNS"
    author:
      -
        ins: S. Sheth
        name: S. Sheth
        org: Verisign Labs
      -
        ins: A. Kaizer
        name: A. Kaizer
        org: Verisign Labs
    date: 2025-04-22

normative:

--- abstract

This document specifies "dns-persist-01", a new validation method for the Automated Certificate Management Environment (ACME) protocol. This method allows a Certification Authority (CA) to verify control over a domain by confirming the presence of a persistent DNS TXT record containing CA and account identification information. This method is particularly suited for environments where traditional challenge methods are impractical, such as IoT deployments, multi-tenant platforms, and scenarios requiring batch certificate operations. The validation method is designed with a strong focus on security and robustness, incorporating widely adopted industry best practices for persistent domain control validation. This design aims to make it suitable for Certification Authorities operating under various policy environments, including those that align with the CA/Browser Forum Baseline Requirements.

--- middle

# Introduction {#introduction}

The Automated Certificate Management Environment (ACME) protocol {{!RFC8555}} defines mechanisms for automating certificate issuance and domain validation. The existing challenge methods, "http-01" and "dns-01", require real-time interaction between the ACME client and the domain's infrastructure during the validation process. While effective for many use cases, these methods present challenges in certain deployment scenarios.

Examples include:

- Internet of Things (IoT) deployments where devices may not be able to host an HTTP service or coordinate DNS updates in real-time.
- Edge compute and multi-tenant hosting platforms where the entity managing the DNS zone is distinct from the tenant subscribing to the certificate.
- Organizations that wish to pre-validate domains and batch issuance operations offline or at a later time.
- Scenarios requiring wildcard certificates where domain control is proven once and reused over an extended period.
- Environments with strict change management processes where DNS modifications require approval workflows.

This document defines a new ACME challenge type, "dns-persist-01". This method proves control over a Fully Qualified Domain Name (FQDN) by confirming the presence of a persistent DNS TXT record containing CA and account identification information.

The record format is based on the "issue-value" syntax from {{!RFC8659}}, incorporating an issuer-domain-name and a mandatory accounturi parameter {{!RFC8657}} that uniquely identifies the applicant's account. This design provides strong binding between the domain, the CA, and the specific account requesting validation.

## Robustness and Alignment with Industry Best Practices {#robustness-and-alignment}

This validation method is designed to provide a robust and persistent mechanism for domain control verification within the ACME protocol. Its technical design incorporates widely adopted security principles and best practices for domain validation, ensuring high assurance regardless of the specific CA policy environment. These principles include, but are not limited to:

1. The use of a well-defined, unique DNS label (e.g., "_validation-persist") for persistent validation records, minimizing potential conflicts.
2. Consideration of DNS TTL values when determining the effective validity period of an authorization, balancing persistence with responsiveness to DNS changes (see {{validation-data-reuse-and-ttl-handling}}).
3. Explicit binding of the domain validation to a specific ACME account through a unique identifier, establishing clear accountability and enhancing security against unauthorized use.

Certification Authorities operating under various trust program requirements will find this technical framework suitable for their domain validation needs, as its design inherently supports robust and auditable validation practices.

# Conventions and Definitions {#conventions-and-definitions}

{::boilerplate bcp14-tagged}

DNS TXT Record Persistent DCV Domain Label
: The label "_validation-persist" as specified in this document. This label is consistent with industry practices for persistent domain validation.

Authorization Domain Name
: The domain name at which the validation TXT record is provisioned. It is formed by prepending the DNS TXT Record Persistent DCV Domain Label to the FQDN being validated.

Issuer Domain Name
: A domain name disclosed by the CA in Section 4.2 of the CA's Certificate Policy and/or Certification Practices Statement to identify the CA for the purposes of this validation method.

Validation Data Reuse Period
: The period during which a CA may rely on validation data, as defined by the CA's practices and applicable requirements.

persistUntil
: An optional parameter in the validation record that specifies the timestamp after which the validation record should no longer be considered valid by CAs. The value MUST be a base-10 encoded integer representing a UNIX timestamp (the number of seconds since 1970-01-01T00:00:00Z ignoring leap seconds).

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

1. The RDATA value MUST conform to the issue-value syntax as defined in {{!RFC8659}}, Section 4.

2. The `issuer-domain-name` portion of the issue-value MUST be the Issuer Domain Name provided by the CA in the challenge object.

3. The issue-value MUST contain an accounturi parameter. The value of this parameter MUST be a unique URI identifying the account of the applicant which requested the validation, constructed according to {{!RFC8657}}, Section 3.

4. The issue-value MAY contain a `policy` parameter. If present, this parameter modifies the validation scope. The `policy` parameter follows the `key=value` syntax. The policy parameter key and its defined values MUST be treated as case-insensitive. The following value for the `policy` parameter is defined with respect to subdomain and wildcard validation:

- `policy=wildcard`: If this value is present, the CA MAY consider this validation sufficient for issuing certificates for the validated FQDN, for specific subdomains of the validated FQDN (as covered by wildcard scope or specific subdomain validation rules), and for wildcard certificates (e.g., `*.example.com`). See {{wildcard-certificate-validation}} and {{subdomain-certificate-validation}}.

If the `policy` parameter is absent, or if its value is anything other than `wildcard`, the CA MUST proceed as if the policy parameter were not present (i.e., the validation applies only to the specific FQDN). CAs MUST ignore any unknown parameter keys.

5. The issue-value MAY contain a `persistUntil` parameter. If present, the value MUST be a base-10 encoded integer representing a UNIX timestamp (the number of seconds since 1970-01-01T00:00:00Z ignoring leap seconds). CAs MUST NOT consider this validation record valid for new validation attempts after the specified timestamp. However, this does not affect the reuse of already-validated data.

For example, if the ACME client is requesting validation for the FQDN "example.com" from a CA that uses "authority.example" as its Issuer Domain Name, and the client's account URI is "https://ca.example/acct/123", it might provision:

~~~ dns
_validation-persist.example.com. IN TXT "authority.example; accounturi=https://ca.example/acct/123"
~~~

The ACME server verifies the challenge by performing a DNS lookup for TXT records at the Authorization Domain Name. It then iterates through the returned records to find one that conforms to the required structure and contains both the correct `issuer-domain-name` and a valid `accounturi` for the requesting account. See {{handling-of-multiple-records}} for detailed requirements. The server also interprets any `policy` parameter values according to this specification.

## Handling of Multiple Records {#handling-of-multiple-records}

A DNS query for the Authorization Domain Name may return multiple TXT records. When multiple records are present, a CA MUST iterate through them to find one that meets the validation requirements. The validation is successful if at least one record satisfies all of the following conditions:

1. The record conforms to all requirements specified in this document
2. The `issuer-domain-name` in the record matches the CA's Issuer Domain Name
3. The `accounturi` parameter exactly matches the URI of the ACME account making the request
4. Any `persistUntil` parameter, if present, has not expired

Malformed records or records intended for other CAs MUST be ignored. If no single record meets all requirements, the validation attempt fails. This approach allows multiple CAs to have persistent validation records for the same domain simultaneously.

## Just-in-Time Validation {#just-in-time-validation}

When processing a new authorization request, a CA MAY perform an immediate DNS lookup for `_validation-persist` TXT records at the Authorization Domain Name corresponding to the requested domain identifier.

If one or more such records exist, the CA MUST evaluate them according to the requirements specified in {{handling-of-multiple-records}}. If at least one record meets all validation requirements, the CA MAY transition the authorization directly to the "valid" status without requiring a challenge response.

If no DNS TXT record meets the validation requirements, or if the records are absent, the CA MUST proceed with the standard authorization flow by returning a "pending" authorization with an associated `dns-persist-01` challenge object.

This mechanism enables efficient reuse of persistent validation records while maintaining the security properties of the validation method.

# Wildcard Certificate Validation {#wildcard-certificate-validation}

This validation method is suitable for validating Wildcard Domain Names (e.g., *.example.com). To authorize a wildcard certificate for a domain, a single DNS TXT record placed at the Authorization Domain Name for the base domain MUST be used. This TXT record MUST include the `policy=wildcard` parameter value.

When such a record is present (i.e., containing `policy=wildcard`), it can validate the base domain, specific subdomains, and wildcard certificates for that domain. For example, a TXT record at `_validation-persist.example.com` containing `policy=wildcard` can validate certificates for `example.com`, `www.example.com`, and `*.example.com`. If the `policy` parameter is absent, the validation is not sufficient for `*.example.com`.

# Subdomain Certificate Validation {#subdomain-certificate-validation}

If an FQDN has been successfully validated using this method, the CA MAY also consider this validation sufficient for issuing certificates for other FQDNs that are subdomains of the validated FQDN, under the following conditions:

* The persistent DNS TXT record MUST include `policy=wildcard`.

To determine which subdomains are permitted, the FQDN for which the persistent TXT record exists (referred to as the "validated FQDN") must appear as the exact suffix of the FQDN for which a certificate is requested (referred to as the "requested FQDN"). For example, if `dept.example.com` is the validated FQDN, a certificate for `server.dept.example.com` is permitted because `dept.example.com` is its suffix.

When `policy=wildcard` is present, the validation authorizes certificates for the *validated FQDN itself*, for *any specific subdomain* of the validated FQDN (per the suffix rule), and for wildcard certificates covering the validated FQDN (e.g., `*.example.com` if `example.com` is the validated FQDN). This policy grants broad scope of validation for both specific subdomains and wildcard certificates.

If the `policy` parameter is absent, or if it is present but its value does not explicitly authorize subdomain validation (e.g., an unrecognized or future policy value), this validation MUST NOT be considered sufficient for issuing certificates for subdomains.

See {{subdomain-validation-risks}} for important security implications of enabling subdomain validation.

Example: Scope of 'wildcard' Policy

For a persistent TXT record provisioned at `_validation-persist.example.com` with a `policy` of `wildcard`, a CA may issue certificates for `example.com`, `www.example.com`, `app.example.com`, and `*.example.com`.

# Security Considerations {#security-considerations}

## Persistent Record Risks {#persistent-record-risks}

The persistence of validation records creates extended windows of vulnerability compared to traditional ACME challenge methods. If an attacker gains control of a DNS zone containing persistent validation records, they can potentially obtain certificates for the validated domains until the validation records are removed or modified.

Clients SHOULD protect validation records through appropriate DNS security measures, including:

- Using DNS providers with strong authentication and access controls
- Implementing DNS security extensions (DNSSEC) where possible
- Monitoring DNS zones for unauthorized changes
- Regularly reviewing and rotating validation records

## Account Binding Security {#account-binding-security}

The `accounturi` parameter provides strong binding between domain validation and specific ACME accounts. However, this binding depends on the security of the ACME account itself.

The security of this method is fundamentally bound to the security of the ACME account's private key. If this key is compromised, an attacker can immediately use any pre-existing `dns-persist-01` authorizations associated with that account to issue certificates, without needing any further access to the domain's DNS infrastructure. This elevates the importance of secure key management for ACME clients far above that required for transient challenge methods, as the window of opportunity for an attacker is tied to the lifetime of the persistent authorization, not a momentary challenge.

CAs SHOULD implement robust account security measures, including:

- Strong authentication requirements for ACME accounts
- Account activity monitoring and anomaly detection
- Rapid account revocation capabilities
- Regular account security reviews
- Account key rotation policies and procedures

Clients SHOULD protect their ACME account keys with the same level of security as they would protect private keys for high-value certificates.

### Account Key Rotation

The `accounturi` parameter is a stable identifier for the ACME account that persists across key rotations. When a client rotates their account key following the procedures defined in {{!RFC8555}}, Section 7.3.5, the `accounturi` remains unchanged. Therefore, existing DNS TXT records containing the `accounturi` parameter do not require modification when performing account key rotations.

## Subdomain Validation Risks {#subdomain-validation-risks}

Enabling subdomain validation via `policy=wildcard` creates significant security implications. Organizations using this feature MUST carefully control subdomain delegation and monitor for unauthorized subdomains. This policy value serves as the explicit mechanism for domain owners to opt-in to broader validation scopes.

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

Additionally, CAs MUST protect their `issuer-domain-name` with robust security measures. Using DNSSEC is a recommended mechanism for this purpose. An attacker who compromises the DNS for a CA's `issuer-domain-name` could disrupt validation or potentially impersonate the CA in certain scenarios. While this is a systemic DNS security risk that extends beyond this specification, it is amplified by any mechanism that relies on DNS for identity.

## DNS Security Measures {#dns-security-measures}

To enhance the security and integrity of the validation process, CAs and clients should consider implementing advanced DNS security measures.

### DNSSEC

DNS Security Extensions (DNSSEC) provide cryptographic authentication of DNS data. This is a critical security measure that ensures the validation records retrieved by a CA are authentic and have not been tampered with.

For CAs operating within the public WebPKI, the use of DNSSEC is a vital best practice for ensuring the integrity of domain validation. For private or closed PKI environments, DNSSEC is strongly recommended but may not be required, depending on the trust model and risk profile of the specific deployment.

### Multi-Perspective Validation

Multi-Perspective Issuance Corroboration (MPIC) is a technique to validate domain control from multiple network vantage points. This is a critical defense against localized network attacks, such as BGP hijacking and DNS spoofing, which could otherwise lead to certificate mis-issuance.

For CAs subject to requirements like the CA/Browser Forum Baseline Requirements, MPIC is essential for robust domain validation. However, for private PKI systems where the network topology is well-known and such localized attacks are not part of the threat model, MPIC may be considered optional.

## Validation Data Reuse and TTL Handling {#validation-data-reuse-and-ttl-handling}

This validation method is explicitly designed for persistence and reuse. The period for which a CA may rely on validation data is its `Validation Data Reuse Period` (as defined in {{conventions-and-definitions}}). However, if the DNS TXT record's Time-to-Live (TTL) is shorter than this period, the CA MUST treat the record's TTL as the effective validation data reuse period for that specific validation.

CAs MAY reuse validation data obtained through this method for the duration of their validation data reuse period, subject to the TTL constraints described in this section. The `persistUntil` parameter indicates when the DNS validation record should no longer be considered valid for new validation attempts. If a `persistUntil` parameter is present in the DNS TXT record, the CA MUST NOT successfully complete a validation attempt after the date and time specified in that parameter. This restriction does not preclude reuse of data that has already been validated.

## persistUntil Parameter Considerations

The `persistUntil` parameter provides domain owners with direct control over the validity period of their validation records. CAs and clients should be aware of the following considerations:

- Domain owners should set expiration dates for validation records that balance security and operational needs. To avoid unexpected validation failures during certificate renewal, domain owners are advised to:
  - Align `persistUntil` values with certificate lifetimes or planned maintenance intervals
  - Monitor or set reminders for `persistUntil` expirations
  - Document `persistUntil` practices in certificate management procedures
  - Automate updates to validation records with new `persistUntil` values during certificate renewal workflows
- CAs MUST properly parse and interpret the integer timestamp value as a UNIX timestamp (the number of seconds since 1970-01-01T00:00:00Z ignoring leap seconds) and apply the expiration correctly.
- CAs MUST reject or consider expired any validation record where the current time exceeds the `persistUntil` timestamp.

## Revocation and Invalidation of Persistent Authorizations {#revocation-and-invalidation}

The persistent nature of `dns-persist-01` authorizations means that a valid DNS TXT record can grant control for an extended period, potentially even if the domain owner's intent changes or if the associated ACME account key is compromised. Therefore, explicit mechanisms for revoking or invalidating these persistent authorizations are critical.

The primary method for an Applicant to invalidate a `dns-persist-01` authorization for a domain is to **remove the corresponding DNS TXT record** from the Authorization Domain Name. After the record is removed, new validation attempts for the domain will fail. Any existing authorization obtained via this method will remain valid until it expires as per the CA's Validation Data Reuse Period.

ACME Clients SHOULD provide clear mechanisms for users to:

* Remove the `_validation-persist` DNS TXT record.
* Monitor the presence and content of their `_validation-persist` records to ensure they accurately reflect desired authorization.

Certificate Authorities (CAs) implementing this method MUST:

* During a validation attempt, fail the validation if the corresponding DNS TXT record is no longer present or if its content does not meet the requirements of this specification (e.g., incorrect `issuer-domain-name`, missing `accounturi`, altered `policy`).

* Reject new validation attempts when the current time exceeds the timestamp specified in a `persistUntil` parameter, even if the DNS TXT record remains present and would otherwise satisfy all other validation requirements.

* Ensure their internal systems are capable of efficiently handling the validation failure when DNS records are removed or become invalid.

While this method provides a persistent signal of control, the fundamental ACME authorization object (as defined in {{!RFC8555}}) remains subject to its own lifecycle, including expiration. A persistent DNS record allows for repeated authorizations, but each authorization object issued by the CA will have a defined validity period, after which it expires unless renewed.


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
- Developing procedures for handling validation record TTL variations
- Creating account security monitoring and incident response procedures
- Providing clear documentation for clients on proper record construction

### Error Handling

When implementing the "dns-persist-01" validation method, Certificate Authorities SHOULD return appropriate ACME error codes to provide clear feedback on validation failures. Specifically:

- CAs SHOULD return a `malformed` error (as defined in {{!RFC8555}}) when the TXT record has invalid syntax, such as duplicate parameters, invalid timestamp format in the `persistUntil` parameter, missing mandatory `accounturi` parameter, or other syntactic violations of the record format specified in this document.

- CAs SHOULD return an `unauthorized` error (as defined in {{!RFC8555}}) when validation fails due to authorization issues, including:
  - The `accounturi` parameter in the DNS TXT record does not match the URI of the ACME account making the request
  - The `persistUntil` timestamp has expired, indicating that the validation record is no longer considered valid for new validation attempts
  - The `issuer-domain-name` in the DNS TXT record does not match the CA's Issuer Domain Name

These error codes help ACME clients distinguish between different types of validation failures and take appropriate corrective actions.

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

~~~ dns
_validation-persist.example.com. IN TXT "authority.example; accounturi=https://ca.example/acct/123"
~~~

3. CA validates the record through DNS queries. This validation is sufficient only for "example.com".


## Wildcard Validation Example {#wildcard-validation-example}

For validation of "*.example.com" (which also validates "example.com" and specific subdomains like "www.example.com") by a CA using "authority.example" as its Issuer Domain Name:

1. Same challenge object format as above.

2. Client provisions DNS TXT record at the base domain's Authorization Domain Name, including `policy=wildcard`:

~~~ dns
_validation-persist.example.com. IN TXT "authority.example; accounturi=https://ca.example/acct/123; policy=wildcard"
~~~

3. CA validates the record through DNS queries. This validation authorizes certificates for "example.com", "*.example.com", and specific subdomains like "www.example.com".

## Validation Example with persistUntil

For validation of "example.com" with an explicit expiration date:

1. Same challenge object format as above.

2. Client provisions DNS TXT record including `persistUntil`:

~~~ dns
_validation-persist.example.com. IN TXT "authority.example; accounturi=https://ca.example/acct/123; persistUntil=1721952000"
~~~

3. CA validates the record. This validation is sufficient only for "example.com" and will not be considered valid after the specified timestamp (2024-07-26T00:00:00Z).

## Wildcard Validation Example with persistUntil

For validation of "*.example.com" with an explicit expiration date:

1. Same challenge object format as above.

2. Client provisions DNS TXT record including `policy=wildcard` and `persistUntil`:

~~~ dns
_validation-persist.example.com. IN TXT "authority.example; accounturi=https://ca.example/acct/123; policy=wildcard; persistUntil=1721952000"
~~~

3. CA validates the record. This validation authorizes certificates for "example.com", "*.example.com", and specific subdomains, but will not be considered valid after the specified timestamp (2024-07-26T00:00:00Z).

--- back

# Acknowledgments
{:unnumbered}

The author would like to acknowledge the CA/Browser Forum for developing the Baseline Requirements that motivated this specification, and the ACME Working Group for their ongoing work on certificate automation protocols.

Thanks to the contributors and reviewers who provided feedback on early versions of this document.
