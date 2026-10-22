---
title: Local certificate authority
standards:
- nist-sc
---

## Context

There are situations at the MOC in which we want to secure access to a
device or service with TLS certificates, but for various reasons we are
not able to take advantage of existing certificate issuers.

### Data encryption in transit

[NIST SP 800-171 Requirement 3.13.8][nist-031308] requires that CUI (Controlled
Unclassified Information) be encrypted both at rest and in transit. Encryption
at rest is generally provided by the cryptographic features of our storage
appliances and servers. Encryption in transit requires co-operation between our
storage appliances and our storage clients, and in the MOC environment the only
viable option is NFS-over-TLS.

[nist-031308]: https://csf.tools/reference/nist-sp-800-171/r2/3-13/3-13-8/

In order for NFS-over-TLS to be effective, we need to establish a
certificate trust relationship between the storage clients and the
certificate presented by the storage server. One way to establish this is
to acquire a certificate from an existing trusted CA like Let's Encrypt,
but this is difficult for an internal machine that is often accessed by IP
address rather than name: it is not possible to obtain Let's Encrypt
certificates for private IP SANs ([Subject Alternative Names][san]). Using
an internal certificate authority means we can issue certificates for
arbitrary names or addresses.

[san]: https://support.dnsimple.com/articles/what-is-ssl-san/

### Securing device access

We generally access server BMCs ([Baseboard Management Controllers][bmc])
via IP address, which brings us to the same problem described in the
previous section -- it is not possible to acquire generally trusted
certificates for these addresses. This means that every device access is
preceded by negotiating the browser's untrusted certificate workflow.
Setting up a local CA and installing the CA certificate in our browsers
would make the process of accessing these devices more convenient and more
secure.

[bmc]: https://www.supermicro.com/en/glossary/baseboard-management-controller

## Decision

We will use [step-ca] as our local certificate authority, running in one of
our internal OpenShift clusters.

[step-ca]: https://smallstep.com/docs/step-ca/

step-ca exposes an API for issuing and renewing certificates, which lets us
automate certificate management instead of relying on someone to run commands
by hand. In particular, this lets us set up non-interactive processes for
automating the certificate renewal process, which would otherwise require
manual remediation for every device. Automating this process makes it much
easier to support certificates with short lifetimes, which is important in
situations in which certificate revocation is not an option. Short certificate
lifetimes limit the damage that can be caused by a stolen certificate.

An API-driven CA also avoids the coordination problems inherent in an
approach like OpenSSL plus a git repository, where the CA state is a set of
files that multiple people might edit at the same time. step-ca provides
centralized issuance and managed CA state, so there's no risk of conflicting
edits or duplicate serial numbers.

We can use step-ca policies to limit the scope of certificates that can be
issued by our CA -- e.g., by restricting it to issuing certificate with IP SANs
in specific network ranges, and limiting DNS SANs to specific domains.

step-ca also supports the ACME protocol, the same protocol used by Let's
Encrypt. This means we could eventually use standard ACME clients to
request and renew certificates from our internal CA, rather than writing
custom tooling against a step-ca-specific API.

### Multi-level certificate authority

Step CA sets up a two-tier certificate authority by default. Server
certificates are signed by an intermediate CA, which is itself signed by the
root CA. This is done in the interests of security -- once the intermediate CA
has been established, the root CA secret does not need to be exposed anywhere.
We install the root CA certificate in browsers and servers that need to trust
generated certificates, and we have servers present a certificate chain that
includes the intermediate certificate.

### What needs to be backed up?

The root certificate and key will be stored in our AWS SecretsManager store. It
is not necessary to back up the intermediate CA certificate; since the root CA
is used to establish trust, the intermediate CA certificate can always be
regenerated and certificates it signs will continue to work.

We should back up the step-ca database. This isn't an operational requirement
(using just the root CA certificate we can create a new authority that can
immediately start issuing trusted certificates), but is useful if we want to
preserve the history of issued certificates.

### Certificate installation and renewal

For this to be effective, it must be possible to automate certificate
provisioning and renewal. In this section, we look at how to accomplish this on
specific devices. Actually building this automation is outside the scope of
this ADR.

#### FlashBlade

- Certificates can be managed via the [REST API].
- There is an [ansible module] that can manage certificates on the
  FlashBlade.

[rest api]: https://code.purestorage.com/swagger/redoc/fb2.0-api-reference.html#tag/Certificates
[ansible module]: https://galaxy.ansible.com/ui/repo/published/everpure/flashblade/content/module/purefb_certs/

#### Dell iDRAC

- Use remote `racadm` to generate CSR and to upload signed certificate. See
  [`update-idrac-cert.sh`](update-idrac-cert.sh.txt) for an example. Activating
  the new configuration requires an iDRAC reset, which renders the iDRAC
  unavailable for several minutes (it does not impact the server itself).

#### Lenovo xClarity 2

- Use refish to generate CSR and to upload signed certificate. See
  [`update-xcc-cert.sh`](update-xcc-cert.sh.txt) for an example.

### Rejected options

#### OpenSSL

We can use the OpenSSL command line tool to operate a
[simple certificate authority][osca]. This has the advantage of not
requiring any infrastructure, but it introduces coordination problems if
more than one person is to be able to issue certificates, and it largely
eliminates the possibility of API-driven automated renewal.

[osca]: https://openssl-ca.readthedocs.io/en/latest/introduction.html

#### AWS

AWS offers a private certificate authority service, [AWS Private CA]. While
this would undoubtedly meet our needs from a technical perspective, it is a
much more expensive solution (there are monthly costs for each CA, for the
number of certificates issued each month, and for any uses of the [OCSP]
protocol).

[aws private ca]: https://aws.amazon.com/private-ca/
[ocsp]: https://en.wikipedia.org/wiki/Online_Certificate_Status_Protocol

## Consequences

- By running our own internal certificate authority we are able to properly
  secure access between our storage clients and the FlashBlade appliance,
  and between operator browsers and the devices that we manage.

- We will need to distribute the CA certificate to storage clients and
  browsers.

- We will need to install and operate the step-ca service.

## See also

- [Production considerations when running a certificate authority server](https://smallstep.com/docs/step-ca/certificate-authority-server-production/#safeguard-your-root-and-intermediate-keys)
- <https://smallstep.com/blog/build-a-tiny-ca-with-raspberry-pi-yubikey/>
