# Getting started

This page shows the generic steps. Every image page on <https://openimages.cloud/images/>
generates the same commands with the real file name, version, checksum, Glance properties and
login user filled in: copy them from there rather than typing versions by hand.

## 1. Find the image

- Browse the catalog: <https://openimages.cloud/images/>. Each image has a page at
  `https://openimages.cloud/images/<id>/` (for example
  <https://openimages.cloud/images/debian-13/>).
- Or read the index, which lists the latest version of every recommended image:

  ```sh
  curl -fsS https://images.openimages.cloud/index.json \
    | jq -r '.images[] | "\(.id)\t\(.latest_version)\t\(.login_user // "-")"'
  ```

Files are served from `https://images.openimages.cloud`:

| Path | Meaning | Cache |
|---|---|---|
| `/<id>/<version>/<file>` | Versioned file, never changes | `public, max-age=31536000, immutable` |
| `/<id>/latest/<file>` | Alias of the newest release | `public, max-age=300` |
| `/<id>/<version>/` and `/<id>/latest/` | HTML listing of the directory | `public, max-age=300` |
| `/<id>/versions.json` | Every published version (see [catalog index](catalog-index.md)) | `public, max-age=300` |

Use versioned URLs in automation: a checksum taken from one release matches the versioned file
forever, while `latest/` moves at the next release.

Some images ship several variants in one version directory (for example Alpine Linux `bios` and
`uefi`, Alpaquita `glibc` and `musl`). Each variant has its own file, `MANIFEST-<variant>.json`
and `SHA256SUMS-<variant>`.

## 2. Download and verify

Next to every image file the registry publishes `<file>.md5`, `<file>.sha1`, `<file>.sha256`
and `<file>.bundle` (the cosign signature), plus `MANIFEST.json` (or
`MANIFEST-<variant>.json`) and `SHA256SUMS` (or `SHA256SUMS-<variant>`) for the version. The
sha512 is in the manifest and the index, not in a sidecar file.

```sh
URL=https://images.openimages.cloud/<id>/<version>/<file>
curl -fLO "$URL"
curl -fLO "$URL.bundle"

# Signature: proves the file was built by the open-img-cloud build workflows
cosign verify-blob "$(basename "$URL")" \
  --bundle "$(basename "$URL").bundle" --new-bundle-format \
  --certificate-identity-regexp '^https://github.com/open-img-cloud/\.github/\.github/workflows/build-(libguestfs|dib|nix-flake|freebsd)-image\.yml@' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com

# Checksum only (detects a corrupted download, proves nothing about who built it)
curl -fsSL "$URL.sha256" | sha256sum -c -
```

Details and the meaning of each check: [Verifying images](verifying-images.md).

## 3. Import into OpenStack Glance

```sh
openstack image create \
  --disk-format qcow2 --container-format bare \
  --file <file> \
  --min-disk <GiB> \
  --property os_distro=<os_distro> --property os_version=<os_version> \
  --property os_admin_user=<login_user> --property architecture=x86_64 \
  --property os_type=linux \
  --property hw_qemu_guest_agent=yes \
  --property hw_disk_bus=scsi --property hw_scsi_model=virtio-scsi \
  --property hw_vif_multiqueue_enabled=true \
  --property os_require_quiesce=yes \
  --property oic_id=<id> --property oic_version=<version> --property oic_sha256=<sha256> \
  <name>
```

- The complete property set of each artifact is published in `index.json`
  (`images[].artifacts[].glance_properties`) and in the snippet of each image page. Use it as is
  instead of the list above, which is only an example.
- UEFI artifacts also need `hw_firmware_type=uefi` and `hw_machine_type=q35` (already included
  in their `glance_properties`).
- `--min-disk` is the virtual size of the image rounded up to the next GiB (`virtual_size` in
  the index or the manifest).
- `hw_rng_model=virtio` is supported by every image but left optional
  (`optional_glance_properties` in the index): Nova refuses to boot it unless the flavor allows
  it (`hw_rng:allowed=True`).
- The `oic_*` properties let tools such as [oic-sync](oic-sync.md) recognize the image and
  detect newer versions. They are harmless otherwise.

To keep images up to date automatically, use [oic-sync](oic-sync.md) or the
[osism export](catalog-index.md#osism-openstack-image-manager-export).

Log in with the image's default user (`login_user` in the index, shown on the image page) and
the key pair you passed to Nova. Password login over SSH is disabled.

## 4. Import into Proxmox VE

The image pages generate this sequence. It downloads the file into the `import` content of a
file-based storage (Proxmox checks the sha256), creates a VM, imports the disk and turns the VM
into a template:

```sh
# The storage ("local" here) must have the "import" content type enabled.
pvesh create /nodes/$(hostname)/storage/local/download-url \
  --content import --filename <file> \
  --url https://images.openimages.cloud/<id>/<version>/<file> \
  --checksum <sha256> --checksum-algorithm sha256
qm create 9000 --name <id> --memory 2048 --cores 2 \
  --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-single \
  --serial0 socket --vga serial0 --agent enabled=1
qm set 9000 --scsi0 local-lvm:0,import-from=local:import/<file>,discard=on
qm set 9000 --ide2 local-lvm:cloudinit --boot order=scsi0 \
  --ciuser <login_user> --sshkeys ~/.ssh/id_ed25519.pub --ipconfig0 ip=dhcp
qm template 9000
```

For a UEFI artifact, add `--bios ovmf --machine q35 --efidisk0 local-lvm:0,efitype=4m` to
`qm create`. Adjust the VM id, storage names and bridge to your cluster.

The images read the Proxmox cloud-init drive: their cloud-init datasource list is
`ConfigDrive, OpenStack, NoCloud, None` (see [image contract](image-contract.md)). Images that
do not use cloud-init (NixOS) document their own metadata mechanism in their repository.

Download into the `import` content is not wrapped in a cosign check: verify the signature on a
workstation first (step 2) if you need provenance, or compare the sha256 with a verified
manifest.
