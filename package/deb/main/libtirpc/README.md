# libtirpc Debian security backports

This package retains Debian 13's libtirpc 1.3.6 ABI and applies the upstream fixes
for CVE-2026-85231 and CVE-2026-85977 through the existing `dpkg/patch@v1` pipeline.
The candidate DHI revision is `1.3.6+ds-1+dhi2`.

- `65d27459fe569a678cb5cc67d308bcf12c49cb40` bounds the three RPCSEC_GSS
  unwrap decode calls to 16 MiB. The upstream diff omits the privacy decode despite its stated intent; the
  backport adds the same bound to that remaining call.
- `ccc1acf867d88dc0bdbe58568f621607d44977f5` bounds each of the three RPC
  list decoders to 1,024 entries. The backport adjusts declaration context for
  1.3.6; the upstream limits and encode/free behavior are retained.

For package maintenance, use the repository's normal DHI build path with
`package/deb/main/libtirpc/debian-13/1.yaml` and authenticated DHI access.
DHI helper steps may require supplier repository access. An access failure is a
build blocker. A standard Debian source build can validate the patches and
packages, but does not validate supplier helper integration.

The recipe pins the Debian source checksums, applies both patches, records the
CVE fixes in the Debian changelog and builds normal scanner-visible packages.
Do not strip package metadata or suppress findings to represent these backports.

Qualification must include the normal Debian package build with GSS enabled on
each required architecture, RPCSEC_GSS oversized-input rejection, and acceptance
of 1,024 versus rejection of 1,025 entries in all three list decoders. Stop on
patch application, build or regression failures; investigate against the pinned
source rather than weakening the limits. Report failures to the package
maintainer with the source revision and build logs.

A repaired package must then be consumed by a rebuilt image, scanned and runtime
tested before release. A source backport alone does not clear an image finding.
If compatibility regresses, retain the failed candidate and evidence and restore
the previous package/image through the normal release process; reverting also
restores the unresolved CVEs and requires the normal risk review.

## Local validation (2026-10-26)

A Linux AMD64 derivative of DHI Cilium 1.20.4 was tested with supplier
`cilium-envoy-1.20=1.20.4-1` and these rebuilt libtirpc packages. The exact
backports matched the built Debian source archive. This was a standard Debian
13 `dpkg-buildpackage` build with GSS enabled, not a successful execution of the
supplier's private helper pipeline.

| Check | Result |
| --- | --- |
| GSS-enabled Debian package build | Passed |
| Three RPC list decoders: accept 1,024 and reject 1,025 entries | Six checks passed |
| Built GSS library: reject oversized integrity body, checksum and privacy body | Three checks passed |
| Two-node Kind connectivity and HTTP L7 policy | Four selected tests, 73 actions passed |
| Agent rolling restart | One selected test, 43 actions passed; eight workload UIDs/IPs unchanged |
| Repaired shared library in both agent pods | Loading and four proxylib API symbols passed |

Runtime validation used Kubernetes 1.35.1 and the matching upstream Cilium
1.20.4 chart, Kubernetes IPAM, VXLAN, kube-proxy, and embedded Envoy
(`envoy.enabled=false`). Agent UID/GID 0 and the chart's normal capabilities
were required; agents were not privileged. The stock BPF mount init container
was privileged. Hubble flow checks and the full connectivity suite were not run.
The separate shared-library loading check does not establish Go parser-specific
behavior.

The derivative's AMD64 manifest was
`sha256:cbbaed05ba64d17cb8dbf474c90a9cb0ae52c4bfe1afb489c807eb4832ce5d83`;
its config identity was
`sha256:b1857816edbfd302b8a0b73d55079e96b4f2a0c4aad216a8fd64c212df126c9b`.
This is a local qualification artifact, not a published supplier image.

Grype 0.119.0 reported no Go findings; direct module inspection confirmed
`golang.org/x/net v0.60.0`. Trivy 0.74.0 covered OS packages only. Both libtirpc
CVEs remained in raw scanner output despite the verified backports. No findings
were suppressed and supplier VEX was not automatically inherited by the local
derivative. No zero-Critical/High or policy-clearance claim is made.

Before release, complete supplier build integration and ARM64 qualification,
publish the repaired package, verify its consumption by rebuilt images, and
scan and disposition those exact released artifacts. The Cilium development
variant and downstream chart integration were not runtime-qualified here.
