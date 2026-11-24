# Terraform and OpenTofu

The [`open-img-cloud/openimages`](https://registry.terraform.io/providers/open-img-cloud/openimages/latest)
provider reads the [catalog index](catalog-index.md) and exposes it as data sources: versioned
URL, checksums, sizes, minimum disk and RAM, and the Glance properties of each image. Chain them
into `openstack_images_image_v2`, a Proxmox VE download, or any resource that takes an image URL
and a checksum.

The provider only reads the public index: it needs no credentials and creates nothing. Sources
and examples: [terraform-provider-openimages](https://github.com/open-img-cloud/terraform-provider-openimages).
License: Apache-2.0.

## Install

```hcl
terraform {
  required_providers {
    openimages = {
      source  = "open-img-cloud/openimages"
      version = "~> 0.2"
    }
  }
}
```

The provider is published on the
[Terraform Registry](https://registry.terraform.io/providers/open-img-cloud/openimages/latest)
and the [OpenTofu Registry](https://search.opentofu.org/provider/open-img-cloud/openimages/latest),
under the same address. The full reference is the
[provider documentation](https://registry.terraform.io/providers/open-img-cloud/openimages/latest/docs).

## Import into OpenStack Glance

Glance downloads the image itself (`web_download`) and receives the catalog properties. The
postcondition checks that the SHA-512 Glance computed is the catalog's:

```hcl
data "openimages_image" "debian" {
  id = "debian-13"
}

resource "openstack_images_image_v2" "debian" {
  name             = "${data.openimages_image.debian.name} (${data.openimages_image.debian.version})"
  image_source_url = data.openimages_image.debian.url
  web_download     = true
  container_format = "bare"
  disk_format      = data.openimages_image.debian.disk_format
  min_disk_gb      = data.openimages_image.debian.min_disk_gb
  min_ram_mb       = data.openimages_image.debian.min_ram_mb
  properties       = data.openimages_image.debian.glance_properties

  lifecycle {
    postcondition {
      condition     = lookup(self.properties, "os_hash_value", "") == data.openimages_image.debian.sha512
      error_message = "The image Glance downloaded does not match the catalog SHA-512."
    }
  }
}
```

The `glance_properties` include the `oic_*` keys, so [oic-sync](oic-sync.md) recognizes images
created this way, and the reverse.

## Proxmox VE

With the [bpg/proxmox](https://registry.terraform.io/providers/bpg/proxmox/latest) provider,
the node downloads the versioned file and checks its SHA-256:

```hcl
data "openimages_image" "rocky" {
  id = "rocky-linux-9"
}

resource "proxmox_download_file" "rocky" {
  node_name          = "pve"
  datastore_id       = "local"
  content_type       = "import"
  url                = data.openimages_image.rocky.url
  file_name          = basename(data.openimages_image.rocky.url)
  checksum           = data.openimages_image.rocky.sha256
  checksum_algorithm = "sha256"
}
```

## Data sources

| Data source | Gives |
|---|---|
| `openimages_image` | One artifact: `url`, `sha256`, `sha512`, `size`, `virtual_size`, `min_disk_gb`, `min_ram_mb`, `disk_format`, `arch`, `name` (the stable name oic-sync uses), `glance_properties`, `optional_glance_properties`, `end_of_life`, `end_of_life_soon`, `security_until`, `bundle_url`, `manifest_url`, the cosign identity and issuer |
| `openimages_images` | The catalog ids, filtered by `tag` or `os_distro`; archived images only with `include_archived` |

`openimages_image` takes the catalog `id`, and optionally `variant` or `firmware` (multi-variant
images such as Alpine Linux) and `version` (default: the latest; older versions come from the
image's `versions.json`).

Pin a `version` for reproducible plans. Without it, a new release changes the URL and the
checksum, so the next plan replaces the Glance image.

## End of life and signatures

- Filter out images past their upstream end of life with `end_of_life` (see
  [Versions, retention and end of life](lifecycle.md)).
- Glance and Proxmox VE check the digest, not the cosign signature. To verify the signature, use
  the `bundle_url`, `cosign_certificate_identity_regexp` and `cosign_certificate_oidc_issuer`
  attributes with `cosign verify-blob` (see [Verifying images](verifying-images.md)), or import
  with [oic-sync](oic-sync.md), which verifies it.

## Internal mirror

For clouds without Internet access, point the provider at a [mirror](mirror.md) of the catalog:

```hcl
provider "openimages" {
  index_url = "https://images.example.internal/index.json"
}
```

`index_url` defaults to `$OPENIMAGES_INDEX_URL`, then to the public index.
