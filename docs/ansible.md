# Ansible

The [`openimages.cloud`](https://github.com/open-img-cloud/ansible-collection-openimages)
collection brings the catalog to Ansible: import images into OpenStack Glance from a playbook,
install [oic-sync](oic-sync.md) with its timers, and run an internal [mirror](mirror.md). The
repository README is the reference; this page summarizes it. License: GPL-3.0-or-later.

## Which role?

| Need | Role |
|---|---|
| Proxmox VE templates kept current | `oic_sync`: installs oic-sync and its daily timer on the node |
| OpenStack images that follow the catalog every day (rotation, deletion of unused versions) | `oic_sync`, on any Debian 12/13 or EL9/EL10 host |
| OpenStack images created as part of a playbook (tenant bootstrap, lab, CI) | `glance_image`: imports what is missing and renames older versions |
| Clouds without access to the CDN | `oic_mirror` on a host that can reach it, then point the clients at the mirror |

`glance_image` and oic-sync use the same Glance properties and names: images imported by one
are recognized by the other.

## Install

```sh
# Until the collection is published on Ansible Galaxy:
ansible-galaxy collection install git+https://github.com/open-img-cloud/ansible-collection-openimages.git
```

Requirements: ansible-core 2.15 or newer; for `glance_image` and `glance_info`, the
`openstack.cloud` collection 2.5.0 or newer (installed as a dependency) and `openstacksdk` on the
host that runs them; optionally cosign 2.4 or newer there, for signature verification.

## Import images into Glance

```yaml
- name: Import openimages.cloud images
  hosts: localhost
  connection: local
  gather_facts: false
  roles:
    - role: openimages.cloud.glance_image
      vars:
        glance_image_cloud: mycloud          # clouds.yaml entry; empty uses OS_* variables
        glance_image_images: [debian-13, ubuntu-24.04, alpine-linux]
        glance_image_variants:
          alpine-linux: [uefi]
```

For each missing version, the role downloads the qcow2 and checks the sha256 of the index,
verifies the cosign signature when cosign is found (`glance_image_verify_signature: required` to
make it mandatory), uploads it with `min_disk`, `min_ram` and every Glance property of the index,
compares the size and the multihash Glance computed with the index (the new image is deleted on
any mismatch), then gives it the stable name ("Debian 13") and renames older versions. It returns
the image ids in `glance_image_ids`.

It never deletes or hides images: for rotation, use oic-sync. `glance_image_eol` (`warn`,
`skip` or `fail`) decides what happens to images past their upstream end of life.

## Install oic-sync

```yaml
- name: Keep openimages.cloud templates on Proxmox VE
  hosts: proxmox
  become: true
  roles:
    - role: openimages.cloud.oic_sync
      vars:
        oic_sync_proxmox_images: [debian-13, ubuntu-24.04]
        oic_sync_proxmox_extra_args: --vmid-base 9000 --keep 2 --delete
```

The role installs the `.deb` or `.rpm` of the oic-sync release (`oic_sync_version`), checks its
sha256 and, when cosign is installed, its signature, writes its configuration and enables
`oic-sync-proxmox.timer` or `oic-sync-openstack.timer` (`oic_sync_openstack_cloud` and
`oic_sync_openstack_images`).

## Run an internal mirror

```yaml
- name: Mirror openimages.cloud for the internal clouds
  hosts: mirror
  become: true
  roles:
    - role: openimages.cloud.oic_mirror
      vars:
        oic_mirror_base_url: http://mirror.example.net/oic
        oic_mirror_images: [debian-13, ubuntu-24.04, rocky-linux-10]
        oic_mirror_keep: 2
        oic_mirror_prune: true
```

The role runs `oic-sync mirror` daily as a dedicated user, into `/srv/oic-mirror` or an S3
bucket. Each image is verified before it appears in the mirror. Clients then use
`OIC_INDEX_URL=http://mirror.example.net/oic/index.json` (or `glance_image_index_url`, or the
provider's `index_url`).

## Modules

| Module | Gives |
|---|---|
| `openimages.cloud.index_info` | What the index offers, without any cloud |
| `openimages.cloud.glance_info` | The versions of catalog images present in a project, by `oic_id` |

Look images up by `oic_id`, not by name: the name moves to each new version.

```yaml
- name: Current Debian 13 image of the project
  openimages.cloud.glance_info:
    cloud: mycloud
    oic_ids: [debian-13]
  register: found
```

`ansible-doc openimages.cloud.index_info` and `ansible-doc openimages.cloud.glance_info` list the
options and return values.
