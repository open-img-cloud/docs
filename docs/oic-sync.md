# oic-sync

[`oic-sync`](https://github.com/open-img-cloud/oic-sync) keeps the catalog images in sync with
an OpenStack Glance or a Proxmox VE node (as VM templates). It reads [`index.json`](catalog-index.md), verifies the cosign signature of
each image, imports the versions the cloud does not have yet, gives the newest one a stable name
("Debian 13") and retires older ones. It keeps no local state: run it daily from a systemd timer, a
Kubernetes CronJob or a CI job; when everything is up to date it changes nothing.

Status: **alpha**. OpenStack and Proxmox VE (VM templates, tested on Proxmox VE 9.2) are
supported. License: Apache-2.0.
The repository README is the reference; this page summarizes it.

## Install

No release has been published yet: the PyPI package and the container image
`ghcr.io/open-img-cloud/oic-sync` will appear with the first tag (the release workflow is ready
but has not run). Until then, install from GitHub (Python 3.10 or newer):

```sh
# OpenStack: the backend needs openstacksdk, an optional extra
pipx install 'oic-sync[openstack] @ git+https://github.com/open-img-cloud/oic-sync.git'
# Proxmox VE, on the node itself: no extra needed
pipx install git+https://github.com/open-img-cloud/oic-sync.git
```

Credentials come from the standard
[`clouds.yaml`](https://docs.openstack.org/openstacksdk/latest/user/config/configuration.html)
(`--cloud`), or from `OS_*` environment variables when no cloud is named. An application
credential restricted to the image service is enough (plus read access to servers and volumes for
`--delete`).

## Usage

```sh
# What the index offers (no cloud needed)
oic-sync list
# Verify the cosign signatures of the index (no cloud needed)
oic-sync verify --image debian-13

# Compare a cloud with the index (read only)
oic-sync openstack status --cloud mycloud
# Show what a sync would do (read only, signatures verified)
oic-sync openstack sync --cloud mycloud --image debian-13 --image ubuntu-24.04 --dry-run
# Do it
oic-sync openstack sync --cloud mycloud --image debian-13 --image ubuntu-24.04 --keep 2
# Operator: public images, and remove versions beyond --keep that nothing uses
oic-sync openstack sync --cloud admin --visibility public --keep 3 --delete
```

Main options (full list: `oic-sync --help` and the repository README):

| Option | Meaning |
|---|---|
| `--image ID` | Image to handle, repeatable. Default: every image of the index except the service images (see below) |
| `--include-service-images` | Also handle the service images when no `--image` is given |
| `--variant NAME` | Only this variant of multi-variant images (`--variant uefi` for Alpine) |
| `--keep N` | Versions kept per image, the current one included (default 2) |
| `--delete` | Delete versions beyond `--keep`, only when no instance and no volume uses them |
| `--visibility` | `private` (default), `shared`, `community` or `public` |
| `--dry-run` | Read the cloud and verify signatures, change nothing |
| `--import-method` | `auto` (default), `web-download`, `glance-direct` or `upload` |
| `--property K=V` | Extra or overriding Glance property (the `oic_*` keys are reserved) |
| `--with-optional-properties` | Add `hw_rng_model=virtio` (flavors must allow `hw_rng:allowed`) |
| `--index-url URL` | Another index (`$OIC_INDEX_URL`), for example a mirror |
| `--cloud`, `--region` | clouds.yaml entry and region (`$OS_CLOUD`) |

`openstack status --check` exits with 1 when an image is not up to date, for monitoring. Exit
codes: 0 success, 1 at least one image failed, 2 invalid arguments or index.

Service and test images skipped by default: `octavia-amphora`, `manila-service-image`,
`trove-guest-image`, the test image `cirros` and the rescue image `cloud-rescue`. Name them with `--image`, or add `--include-service-images`.

## How it works

For each selected artifact:

1. **Find** existing images by their properties (`oic_id`, plus `oic_variant` for multi-variant
   images) in the current project. `oic_version` and `oic_sha256` tell which version each holds.
2. **Verify** the cosign bundle against the sha256 of the index, with the signer identity and
   issuer published in the index. A failed verification means no import.
3. **Import** hidden under a versioned name, with the Glance properties of the index and
   `min_disk` from the virtual size. With Glance `web-download`, the multihash Glance computes
   (sha512 by default) must equal the index sha512; otherwise the image is streamed from the CDN
   to Glance with sha256, sha512 and size checked on the fly. On any mismatch the new image is
   deleted.
4. **Promote** the verified image to the stable name and unhide it.
5. **Retire** older versions: renamed `Debian 13 (<version>)` and hidden, still usable by
   ID.
6. **Rotate**: keep the `--keep` newest; with `--delete`, delete older ones only when no server
   and no volume uses them.

Trust chain note: the signature covers the sha256. With `web-download`, Glance only checks the
sha512, which comes from the manifest over HTTPS but is not itself signed. Use
`--import-method upload` to check the signed digest on every byte.

### Terraform and OpenTofu

The stable name moves to each new version, so a data source that looks an image up by name will
pick the new image after a sync. Look images up by property instead:

```hcl
data "openstack_images_image_v2" "debian" {
  properties  = { oic_id = "debian-13" }
  most_recent = true
}
```

## Proxmox VE

`oic-sync proxmox` keeps one VM template per image version on a Proxmox VE node, ready for
`qm clone`. It runs on the node as root and talks to the local API through `pvesh`: no API token.
Downloads go to a file storage with the `import` content type, enabled once:

```sh
# list the content types the storage already has, plus import
pvesm set local --content iso,vztmpl,backup,import

oic-sync proxmox status
oic-sync proxmox sync --image debian-13 --image ubuntu-24.04 --dry-run
oic-sync proxmox sync --image debian-13 --image ubuntu-24.04 --vmid-base 9000 --disk-storage local-lvm
oic-sync proxmox sync --image debian-13 --keep 2 --delete

# Use a template (cloud images are small: grow the disk before the first boot)
qm clone 9000 120 --name web1 --full 0
qm set 120 --sshkeys ~/.ssh/id_ed25519.pub
qm resize 120 scsi0 +20G && qm start 120
```

Each import is checked by Proxmox itself against the signed sha256 (`download-url --checksum`).
Templates get the tags `oic` and `oic-<image id>`; the current version is named after the image
(`debian-13`), older ones `<image id>-<version>` with the tag `oic-retired`. With `--delete`, a
version beyond `--keep` is destroyed only when no linked clone uses it. Clones are never
touched. Options and details: the
[repository README](https://github.com/open-img-cloud/oic-sync#proxmox-ve).

## Run it periodically

Examples in the repository:
[systemd service and daily timer](https://github.com/open-img-cloud/oic-sync/tree/main/examples/systemd)
and [Kubernetes CronJob](https://github.com/open-img-cloud/oic-sync/blob/main/examples/kubernetes/cronjob.yaml).
The index is rebuilt at least once a day, so a daily run is enough.
