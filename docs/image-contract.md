# Image contract and quality checks

What every general-purpose image in the catalog guarantees, and how it is checked. The public
summary is <https://openimages.cloud/quality/>; the normative rules are the
[image standards](https://github.com/open-img-cloud/.github/blob/main/docs/IMAGE_STANDARDS.md)
in `open-img-cloud/.github` (sections A to K). When this page and the standards disagree, the
standards win.

Service images (Octavia, Manila, Trove) follow a different contract: see
[OpenStack service images](service-images.md).

## What you get

- **Official upstream source.** Each image starts from what the distribution publishes (its cloud
  image, or its sources for images built from scratch), never from a third-party base.
- **One cloud-init policy** for every cloud-init distribution, installed as
  `/etc/cloud/cloud.cfg.d/99_oic-policy.cfg` by the libguestfs build workflow
  ([template](https://github.com/open-img-cloud/.github/blob/main/templates/cloud.cfg.d/99_oic-policy.cfg)):

  ```yaml
  datasource_list: ['ConfigDrive', 'OpenStack', 'NoCloud', 'None']
  disable_root: true
  ssh_pwauth: false
  preserve_hostname: false
  mount_default_fields: [~, ~, 'auto', 'defaults,nofail', '0', '2']
  ```

  ConfigDrive comes first because it is read locally, without waiting for DHCP. NoCloud covers
  local seeds and the Proxmox cloud-init drive.
- **A documented default user** with passwordless administrator access (sudo, or doas) and SSH
  key authentication only. Root login over SSH is disabled. The user name is `login_user` in
  `index.json` and on each image page.
- **Serial console** on `ttyS0` and the **QEMU guest agent** where the distribution packages it
  (exceptions are documented in the image repository).
- **Hostname from metadata**, root disk grown to the size of the flavor or VM disk.
- **Clean first boot:** machine id, SSH host keys, logs and package caches are removed during the
  build.
- **Format:** sparse, compressed qcow2, `x86_64`. ARM64 is not published.

NixOS does not ship cloud-init; it uses its own metadata fetcher with the same outcome (section D
of the standards). Each image repository documents its own deviations at the top of its README.

## How a release is checked

| Stage | Check | Applies to |
|---|---|---|
| Upstream download | Compared with the checksum list or hash the upstream publishes (SHA-256 or SHA-512) when the image repository configures one; otherwise the build looks for `<url>.sha256` and only warns if none exists | libguestfs builds |
| Builder | The builder container signature is verified with cosign before the build (images that build in `ubuntu:24.04` opt out and document it) | libguestfs, DIB, Nix builds |
| Boot test before signing | The qcow2 boots in QEMU/KVM with a ConfigDrive seed; CI logs in over SSH and checks cloud-init did not fail. A failure stops the release before anything is published | libguestfs builds |
| Signature | cosign keyless signature and provenance manifest | all |
| Publication checks | Downloaded back through the public CDN: manifest fields, sidecar files, cache headers, size threshold, cosign signature | all |
| Runtime test on OpenStack | Imported and booted on Infomaniak Public Cloud, region `dc3-a` or `dc4-a` at random, then the contract below | images that set `runtime_smoke_login_user` |
| Weekly audit | Every Monday, the `latest/` artifact of every catalog entry and variant goes through the publication checks again; a failure opens a "Catalog audit" issue in `open-img-cloud/.github` | all |

### Runtime contract (section K of the standards)

All checks are required (`strict` and `require_ipv6` are on by default):

1. SSH login with the injected key as the documented default user.
2. cloud-init finished without error (status, warnings and tracebacks printed on failure).
3. DNS resolution inside the guest.
4. sshd does not offer password authentication.
5. Hostname taken from the metadata.
6. Root partition and filesystem (or ZFS pool) grown to at least 80% of the flavor disk.
7. Passwordless root for the default user (sudo or doas).
8. The global IPv6 address Neutron assigned to the port is configured, and the guest reaches the
   IPv6 Internet.

Timeouts are generous on purpose (SSH 7 minutes, IPv6 address 4 minutes, DNS and IPv6 egress 2
minutes) because DHCP leases over 2 minutes were measured in one region: a failure means the
image is wrong, not that the cloud was slow.

### Known limits

- The runtime test runs **after** publication: a failing release is visible until it is fixed and
  republished. Running it before `latest/` moves is on the roadmap
  (<https://openimages.cloud/quality/>).
- Images built with diskimage-builder, Nix or the FreeBSD workflow have no QEMU boot test before
  signing; they rely on the post-publication runtime test when they set a login user.
- Service images have no runtime test yet (see [service images](service-images.md)).
