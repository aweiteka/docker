# Docker Image Signing

---

**Provenance** (from the French *provenir* meaning "to come from"): the origin or source of something

---

## Overview
As Docker application packaging becomes mainstream users need to ensure that their applications are transferred securely, have integrity and come from trusted sources. Image provenance includes:
* data integrity
* issuer trust and verification
* supporting distributed access to assemble an image
* attestation of the image assembly (commands/instructions to build it)

## Goals

1. **Verify offline:** Data integrity verification may be performed without connection to Internet.
1. **Inspect image assembly:** An application may be inspected to determine how it was assembled and who assembled each layer.
1. **Arbitrary metadata support:** Authors or maintainers may add arbirary metadata describing the applicatation to support their unique needs.
1. **Application certification support:** Provide mechanism for a certifying party to certify applications built by independent software vendors (ISVs).
1. **Apply Chain of trust model:** In the same way that a signed certificate can be traced to a trusted root CA issuer, layer ancestry built on a series of signed and trusted layers can be trusted.
1. **Retain simplicity:** The signing model should be transparent to the end-user, if desired, and straightforward for the maintainer. Users are able to easily build images and securly pull images without excessive security burdens.
1. **"Engine" delegation:** The docker daemon is configured by user to perform verification and signing. Docker commands are run as root or under sudo and passed to daemon.

## Non-goals
1. **Encrypting images**
1. **Secure data in flight:** A secured SSL/TLS connection is desired when an image layer is transferred. Secure transfer should be the default but insecure transfer supported. See [PR#3070](https://github.com/dotcloud/docker/pull/3070) Add support for certificates for repositories
1. **Separate transport from identity:** Image naming is arbitrary and independent of the registry location (URL). Images may be pushed to many registries or mirrors so layers are easily accessible. Consider the BitTorrent-style federation of file servers.
1. **Remote assembly:** Image layers may be safely pulled from multiple issuers when assembling application layers.

## Roles
* **Author/Maintainer**: The author/maintainer includes certification, policy and/or licensing metadata in the Dockerfile.
* **Image builder**: The image builder GPG signs the image using private key.

There are many author/maintainers but most do not build images for wide distribution. It is expected that in the majority of situations where trust verification is important, a known image builder will be distributing images. This may be Docker (via Stackbrew Trusted Builds), a cloud provider or a software company. Understanding this helps inform how key distribution may be performed.

## Image Build Signing
1. Optional Dockerfile `META` included in generate metadata. See [Generated Metadata](#generated-metadata).
1. Signature information added to metadata for public key verification and retrieval.
1. Signer public key added to image to simplify key distribution in trusted situations.
1. `--sign <gpg_email_or_ID>` argument used to pass signature information to build process. Docker signs with detached signature and ASCII armor creating a <image>.asc file. Example: `gpg -a -b <image>`

## Image Transfer
Image and detached signature file are pushed and pulled with image so the signature may be checked.

## Image Verification

Image pull and verification follows current Linux package installation and verification.

`docker pull <image> [-y]`

`docker verify <image>`

* GPG signature checked during docker pull
* GPG verification may be disabled in a configuration file, similar to a repository file.
* User is prompted for public key installation unless `-y` flag is passed.


## Dockerfile
The Dockerfile creates the manifest metadata stored with the image. The dockerfile is also stuffed into the image (path TBD) as a human-readable record of how the image was built.

```
FROM        fedora
MAINTAINER  Aaron Weitekamp <aweiteka@gmail.com>
META        [
            "docs" :    "<http://readthedocs.org/>",
            "license" : "<file://LICENSE>",
            "product" : "ID123456789",
            "policy"  : "Publication or distribution policy statement",
            "foo" :     "bar"
            ]

...

```
### META Syntax
* Values enclosed with `<>` denote links
* `file://` copies local file from build directory into the image. For example, `file://LICENSE` will copy the file LICENSE in the Dockerfile build directory to the image. This allows offline viewing of author/maintainer extended metadata such as the full text of a license or certification.

## Generated Metadata

```
$ sudo docker inspect fedora:20
[{
    "id": "b7de3133ff989df914ae9382a1e8bb6771aeb7b07c5d7eeb8ee266b1ccff5709",
    "parent": "ef52fb1fe61037a1b531698c093205f214ade751c781e30ce4f9a7d33020a0f2",
    "created": "2014-04-24T16:04:14.925132357Z",
    "container": "9565c6517a0e5760ddc5b9d21483b05748b2f04f6711219c7e1ac15de64be7b5",
    ...
    "metadata": {
        "docs": "<https://readthedocs.org/>",
        "license": "<http://example.com/>",
        "product": "ID123456789",
        "policy": "Publication or distribution policy statement with URL",
        "foo": "bar",
    },
    "author": "Lokesh Mandvekar \u003clsm5@redhat.com\u003e - ./buildcontainers.sh",
    "signature": {
        "name": "Aaron Weitekamp",
        "email": "\u003caweiteka@redhat.com\u003e",
        "fingerprint": "04AA 2FAA 4762 69AB 3407  FA45 FD5E B4DB 4807 17ED",
        "pubkeys": {
            "id": 0x480717ED,
            "server": "pgp.mit.edu",
        }
        "sigfile": "fedora20_image.asc"
    },
    ...
}]
```

## Implementation Notes

### GPG Public Keys
On Linux systems public keys are stored using two methods. User keys are managed using native GPG tools in a keyring database. Public keys may be retrieved keyserver using the command `gpg --keyserver pgp.mit.edu --recv-key 0x37017186`. Software vendors typically store their public keys as flat files in /etc/pki/. In some cases a package management tool such as rpm is used to install a key file.

#### Native GPG
```
$ gpg --list-keys
/home/aaron/.gnupg/pubring.gpg
------------------------------
pub   2048R/480717ED 2013-07-26
uid                  Aaron Weitekamp (work key) <aweiteka@redhat.com>
sub   2048R/834C1E4D 2013-07-26
```

**Pros**
* Native GPG implementation with mature tooling
* Web of trust verification based on GPG configuration

**Cons**
* user must add software vendors to their personal keyring.
* not clear how the docker daemon accesses the user public keyring.
* Web of trust signing is not widely used outside of software development community.

#### Flat files
```
# /etc/pki/rpm-gpg/
RPM-GPG-KEY-adobe-linux
RPM-GPG-KEY-fedora-20-primary
RPM-GPG-KEY-fedora-20-secondary
```

**Pros**
* Simple. Model currently used by some software vendors.

**Cons**
* Cannot leverage full GPG toolset.
* Too simple


#### Key Distribution Options
Each method for distributing public keys has drawbacks. Options:

* [DANE protocol](http://tools.ietf.org/html/draft-wouters-dane-openpgp-02) may be used to securely download keys. It is unclear what infrastructure is needed and how widely it has been adopted.
* Signer specifies public keyserver where key may be downloaded. A docker configuration file contains list of trusted keyservers. If list matches signing party keyserver, use it to download public key.
* Web of trust: GPG keys may be signed as trusted from one person to another. The usefullness of these signatures improves if the web of trust can be traced back to the validating party. See [discussion](http://www.linuxfoundation.org/news-media/blogs/browse/2014/02/pgp-web-trust-delegated-trust-and-keyservers). The GPG configuration file `~/.gnupg/gpg.conf` provides support for specifying the level of trust required.

### GPG Signing and Private Keys
GPG signing requires a public/private keypair in the GPG keyring. A keypair may be generated with `gpg --gen-key`. It is not clear how the docker daemon (running as root) accesses this private key during the build and signing process.

## Application Certification Model
The Docker ecosystem enables opportunities for ISVs to distribute certified applications built on platform base layer(s). In this model each layer is signed by the author/maintainer. A certifying party then builds and signs a "metadata-only" layer that certifies the application. The metadata provides attestation that the application is certified by the party and references related policy information.

## GnuPG
GPG is a suite of cryptographic tools compatible with OpenPGP. It relies on a web of trust relationship between a public key and its owner. It implements a decentralized public key infrastructure (PKI) trust model where certificate authority (CA) organizations are not used to validate a public key. Root CA systems like X509 require issuers (maintainers) to either have a root-signed certificate or self-sign. Self-signed X509 certificates do not provide enough verification of issuer trust. GPG is developer friendly and used by large organizaions. For example, GPG is used to sign RPM packages from Red Hat, who publishes their [public key](https://access.redhat.com/site/security/team/key/).

## Standards and resources
* OpenPGP [RFC4880](http://tools.ietf.org/html/rfc4880)
* [GnuPG](https://www.gnupg.org/)
* <http://godoc.org/code.google.com/p/go.crypto/openpgp>

## Open Questions (FIXME)
* What signing algorithm? At least SHA-256, preferrably SHA-512
* Should Dockerfile `META` keys be standardized while still allowing for extension? What metadata is a first-class object vs part of the extended data structure?


## Current Metadata Output
```
# docker inspect fedora:20
[{
    "id": "b7de3133ff989df914ae9382a1e8bb6771aeb7b07c5d7eeb8ee266b1ccff5709",
    "parent": "ef52fb1fe61037a1b531698c093205f214ade751c781e30ce4f9a7d33020a0f2",
    "created": "2014-04-24T16:04:14.925132357Z",
    "container": "9565c6517a0e5760ddc5b9d21483b05748b2f04f6711219c7e1ac15de64be7b5",
    "container_config": {
        "Hostname": "9565c6517a0e",
        "Domainname": "",
        "User": "",
        "Memory": 0,
        "MemorySwap": 0,
        "CpuShares": 0,
        "AttachStdin": false,
        "AttachStdout": false,
        "AttachStderr": false,
        "PortSpecs": null,
        "ExposedPorts": null,
        "Tty": false,
        "OpenStdin": false,
        "StdinOnce": false,
        "Env": [
            "HOME=/",
            "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
        ],
        "Cmd": [
            "/bin/sh",
            "-c",
            "#(nop) ADD file:a117c9572f1c87cbe6a730cd1c223dc6fb8e8f63ecb441d7ce2e09abeb93fac0 in /"
        ],
        "Dns": null,
        "Image": "ef52fb1fe61037a1b531698c093205f214ade751c781e30ce4f9a7d33020a0f2",
        "Volumes": null,
        "VolumesFrom": "",
        "WorkingDir": "",
        "Entrypoint": null,
        "NetworkDisabled": false,
        "OnBuild": []
    },
    "docker_version": "0.10.0",
    "author": "Lokesh Mandvekar \u003clsm5@redhat.com\u003e - ./buildcontainers.sh",
    "config": {
        "Hostname": "9565c6517a0e",
        "Domainname": "",
        "User": "",
        "Memory": 0,
        "MemorySwap": 0,
        "CpuShares": 0,
        "AttachStdin": false,
        "AttachStdout": false,
        "AttachStderr": false,
        "PortSpecs": null,
        "ExposedPorts": null,
        "Tty": false,
        "OpenStdin": false,
        "StdinOnce": false,
        "Env": [
            "HOME=/",
            "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
        ],
        "Cmd": null,
        "Dns": null,
        "Image": "ef52fb1fe61037a1b531698c093205f214ade751c781e30ce4f9a7d33020a0f2",
        "Volumes": null,
        "VolumesFrom": "",
        "WorkingDir": "",
        "Entrypoint": null,
        "NetworkDisabled": false,
        "OnBuild": []
    },
    "architecture": "amd64",
    "os": "linux",
    "Size": 372663898
}]
```
