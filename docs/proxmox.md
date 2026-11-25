# Proxmox VE

This tutorial takes a Proxmox VE node from nothing to a running VM built from an
openimages.cloud image: a cloud-init template first, then VMs cloned from it. It ends with the
three ways to keep the templates current without doing it by hand: oic-sync, Terraform and
Ansible.

| You want | Use |
|---|---|
| One template, now, to try an image | [A template by hand](#1-a-template-by-hand) |
| Templates that follow every new image version | [oic-sync](#keep-templates-current-with-oic-sync), on the node, with a daily timer |
| VMs described as code | [Terraform or OpenTofu](#terraform-and-opentofu) with the bpg/proxmox provider |
| oic-sync rolled out to many nodes | [Ansible](#ansible), role `openimages.cloud.oic_sync` |

Tested on Proxmox VE 9.2 (Debian 13 with DHCP and a fixed address, Alpine Linux UEFI, a
custom user-data, the Terraform example with OpenTofu and bpg/proxmox 0.123). Commands run
on the node, as root.

## 0. Prepare a storage for imports

Proxmox VE downloads disk images into a file storage that has the `import` content type. It is
not enabled by default. Look at the content types of the storage, then set them again with
`import` added:

```sh
pvesm status --content import        # empty: no storage has it yet
grep -A3 '^dir: local' /etc/pve/storage.cfg
pvesm set local --content iso,vztmpl,backup,import   # keep the types the storage already had
```

The VM disks go to any storage that holds VM images: `local-lvm` on a default install,
`local-zfs` on a ZFS install, or a directory storage. Replace `local-lvm` in the commands below
with yours.

## 1. A template by hand

### Pick an image

Every image has an id (`debian-13`, `rocky-linux-9`, `alpine-linux`...), listed in the
[catalog](https://openimages.cloud/images/). The [catalog index](catalog-index.md) gives the
versioned URL, the sha256, the default user and the minimum RAM. This reads them on the node
(`python3` is installed on every Proxmox VE node):

```sh
IMG=debian-13
VARIANT=            # only for images with variants: alpine-linux (bios, uefi), alpaquita-linux (glibc, musl)
read -r URL SHA USER RAM FW < <(curl -fsS https://images.openimages.cloud/index.json | python3 -c '
import json, sys
img = next(i for i in json.load(sys.stdin)["images"] if i["id"] == sys.argv[1])
a = next(a for a in img["artifacts"] if sys.argv[2] in ("", a.get("variant")))
print(a["url"], a["sha256"], img["login_user"], img["min_ram"], a["firmware"])
' "$IMG" "$VARIANT")
FILE=$(basename "$URL")
echo "$FILE, user $USER, at least $RAM MiB, firmware $FW"
```

The [command generator](https://openimages.cloud/get/) (target Proxmox VE) gives the same
commands with the values filled in, if you prefer to copy them.

### Download it

The node downloads the versioned file and checks its sha256 before keeping it:

```sh
pvesh create /nodes/$(hostname)/storage/local/download-url \
  --content import --filename "$FILE" --url "$URL" \
  --checksum "$SHA" --checksum-algorithm sha256
```

This checks the file against the index, not its signature. To check who built the image,
verify the cosign signature first (see [Verifying images](verifying-images.md)); oic-sync does
it for you.

### Create the template

```sh
VMID=9000
qm create $VMID --name "$IMG" --memory $(( RAM > 2048 ? RAM : 2048 )) --cores 2 \
  --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-single \
  --serial0 socket --vga serial0 --agent enabled=1 --ostype l26
qm set $VMID --scsi0 "local-lvm:0,import-from=local:import/$FILE,discard=on"
qm set $VMID --ide2 local-lvm:cloudinit --boot order=scsi0 --ciuser "$USER"
qm template $VMID
```

What each part does:

- `--scsihw virtio-scsi-single` and `scsi0`: the disk bus the images are tested on;
- `--serial0 socket --vga serial0`: the images print their console on the serial port, so
  `qm terminal` and the web console show the boot;
- `--agent enabled=1`: every OS image runs `qemu-guest-agent` (see the
  [image contract](image-contract.md)), so Proxmox VE shows the VM IP addresses and shuts it
  down cleanly;
- `ide2 ... cloudinit`: the cloud-init drive Proxmox VE writes the user, SSH keys and network
  into. The images read it (datasources `ConfigDrive, OpenStack, NoCloud, None`);
- `--ciuser "$USER"`: the image's default user. Another name works too: cloud-init creates it.

For a FreeBSD image, use `--ostype other` instead of `l26`.

#### UEFI images

Most images boot with SeaBIOS, the default. The `uefi` variant of `alpine-linux`
(`FW=uefi` above) needs OVMF and an EFI disk. Add these to `qm create`:

```sh
--bios ovmf --machine q35 --efidisk0 local-lvm:0,efitype=4m,pre-enrolled-keys=0
```

The images are not signed for Secure Boot: keep `pre-enrolled-keys=0`.

## 2. A VM from the template

```sh
qm clone 9000 120 --name web1 --full 1
qm set 120 --sshkeys ~/.ssh/id_ed25519.pub --ipconfig0 ip=dhcp
qm resize 120 scsi0 +20G
qm start 120
```

- `--full 1` copies the disk: the VM no longer depends on the template. A linked clone
  (`--full 0`) is faster and smaller, but keeps the template alive (oic-sync never deletes a
  template that has linked clones).
- The image disks are small (`min_disk_gb` in the index, a few GB at most): grow the disk
  before the first boot. On the Linux images, cloud-init grows the partition and the file
  system at boot.
- For a fixed address:
  `--ipconfig0 ip=192.0.2.10/24,gw=192.0.2.1 --nameserver 192.0.2.53 --searchdomain example.org`.
  With `ip=dhcp`, the VM uses the DNS servers the DHCP server gives: `--nameserver` is not
  applied to a DHCP interface.
- Proxmox VE upgrades the packages at first boot by default (`ciupgrade`). It needs DNS and
  Internet access from the VM; without them, the first boot waits on the package mirrors and
  cloud-init reports an error for that step. `qm set 120 --ciupgrade 0` skips it: the images
  are rebuilt from current upstream releases, so they are recent anyway.

Check that it came up:

```sh
qm agent 120 ping && qm guest cmd 120 network-get-interfaces   # IP addresses, from the guest agent
qm terminal 120                                                 # serial console, Ctrl+O to leave
ssh "$USER"@<address>
```

### Your own cloud-init

`--ciuser`, `--sshkeys` and `--ipconfig0` cover the usual cases. For packages, files or
commands at first boot, pass your own user-data. It goes to a storage with the `snippets`
content type, and replaces the user part that Proxmox VE generates (put the user and SSH key
in it):

```sh
pvesm set local --content iso,vztmpl,backup,import,snippets
cat > /var/lib/vz/snippets/web.yaml <<'EOF'
#cloud-config
users:
  - name: debian
    sudo: ALL=(ALL) NOPASSWD:ALL
    ssh_authorized_keys: [ssh-ed25519 AAAA... you@example.org]
packages: [nginx]
EOF
qm set 120 --cicustom "user=local:snippets/web.yaml"
```

`qm cloudinit dump 120 user` shows the user-data Proxmox VE generates, not the snippet: check
the snippet file itself, and `/var/log/cloud-init.log` in the VM.

### Images without cloud-init

NixOS, Flatcar, Talos Linux and OPNsense read their configuration differently (NixOS module,
Ignition, Talos machine config, OPNsense first boot). The steps above still apply to download
the image and create the VM; the README of each image repository explains how to configure it
on Proxmox VE.

## Keep templates current with oic-sync

A template made by hand stays at the version you downloaded. [oic-sync](oic-sync.md) does the
steps above for you, on the node, for every image you list: it verifies the cosign signature,
creates one template per new version, names the newest after the image (`debian-13`), and
rotates the old ones out without touching clones.

```sh
apt install pipx && pipx install oic-sync
oic-sync proxmox sync --image debian-13 --image ubuntu-24.04 --vmid-base 9000 --dry-run
oic-sync proxmox sync --image debian-13 --image ubuntu-24.04 --vmid-base 9000
```

The [Proxmox VE guide of oic-sync](https://openimages.cloud/oic-sync/proxmox/) covers the
options, the daily timer and the rotation; oic-sync is also packaged as `.deb` (see
[oic-sync](oic-sync.md)).

## Terraform and OpenTofu

With the [openimages provider](terraform.md) and
[bpg/proxmox](https://registry.terraform.io/providers/bpg/proxmox/latest), the catalog gives
the URL, the sha256, the minimum disk and RAM, and the default user; the node downloads the
image and the VM imports it directly, without a template:

```hcl
terraform {
  required_providers {
    openimages = { source = "open-img-cloud/openimages", version = "~> 0.2" }
    proxmox    = { source = "bpg/proxmox" }
  }
}

data "openimages_image" "debian" {
  id = "debian-13"
}

resource "proxmox_download_file" "debian" {
  node_name          = "pve"
  datastore_id       = "local"
  content_type       = "import"
  url                = data.openimages_image.debian.url
  file_name          = basename(data.openimages_image.debian.url)
  checksum           = data.openimages_image.debian.sha256
  checksum_algorithm = "sha256"
}

resource "proxmox_virtual_environment_vm" "web1" {
  name          = "web1"
  node_name     = "pve"
  scsi_hardware = "virtio-scsi-single"

  agent {
    enabled = true
  }

  cpu {
    cores = 2
  }

  memory {
    dedicated = max(data.openimages_image.debian.min_ram_mb, 2048)
  }

  disk {
    datastore_id = "local-lvm"
    import_from  = proxmox_download_file.debian.id
    interface    = "scsi0"
    size         = max(data.openimages_image.debian.min_disk_gb, 20)
    discard      = "on"
  }

  initialization {
    datastore_id = "local-lvm"
    ip_config {
      ipv4 {
        address = "dhcp"
      }
    }
    user_account {
      username = data.openimages_image.debian.glance_properties["image_original_user"]
      keys     = [trimspace(file("~/.ssh/id_ed25519.pub"))]
    }
  }

  network_device {
    bridge = "vmbr0"
  }

  operating_system {
    type = "l26"
  }

  serial_device {}
}
```

- Without `version` in the data source, the next plan after a new release downloads the new
  file and replaces the VM. Pin `version = "..."` for VMs that must stay, and change it when
  you decide to move.
- For the UEFI variant of Alpine (`variant = "uefi"`), add `bios = "ovmf"`, `machine = "q35"`
  and an `efi_disk { datastore_id = "local-lvm", type = "4m", pre_enrolled_keys = false }`
  block.
- `proxmox_download_file` needs a user with `Datastore.AllocateTemplate`, `Sys.Audit` and
  `Sys.Modify` on the node (see the bpg/proxmox documentation).
- The proxmox provider reads its connection from the environment: `PROXMOX_VE_ENDPOINT`
  (`https://<node>:8006/`) and `PROXMOX_VE_API_TOKEN` (`user@realm!tokenid=secret`), plus
  `PROXMOX_VE_INSECURE=true` for the default self-signed certificate.
- Replace `pve`, `local-lvm` and `vmbr0` with your node name, storage and bridge.

## Ansible

The role `openimages.cloud.oic_sync` of the [Ansible collection](ansible.md) installs oic-sync
from its signed package on each node and enables its daily timer:

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

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| The download or the import refuses the storage | The storage lacks the `import` content type ([step 0](#0-prepare-a-storage-for-imports)). |
| The download fails on the checksum | The file changed in transit or the sha256 is from another version: read URL and sha256 from the index together, as above. |
| The VM boots but SSH refuses the key | No cloud-init drive (`ide2 ... cloudinit`) or no `--sshkeys`; check with `qm cloudinit dump <vmid> user`. |
| First boot slow, `cloud-init status` says error on `package_update_upgrade_install` | No DNS or Internet from the VM for the first-boot upgrade: fix the network, or `--ciupgrade 0`. |
| No IP address shown in Proxmox VE | `--agent enabled=1` missing, or the VM is still booting; `qm agent <vmid> ping`. |
| Nothing on the console | `--serial0 socket --vga serial0` missing: the images write to the serial port. |
| Kernel panic or out of memory at boot | Less RAM than the image's `min_ram`: give it at least that much. |
| UEFI image drops to the EFI shell | `--bios ovmf` without an EFI disk, or Secure Boot keys enrolled: `efitype=4m,pre-enrolled-keys=0`. |
