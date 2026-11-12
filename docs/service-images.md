# OpenStack service images

Three catalog images are **OpenStack control plane components**: an OpenStack service boots
them, tenants never do. They are built with diskimage-builder (reusable workflow
`build-dib-image.yml`), signed and published like the other images, but:

- their version is the **OpenStack series** (`YYYY.N`, as in the OpenStack release names), not an OS release; use the
  image of your series;
- they do not get the shared cloud-init policy and have **no runtime test** on OpenStack yet;
- they are left out of the osism export, and `oic-sync` skips Octavia and Manila unless asked
  (see [oic-sync](oic-sync.md));
- they build in a privileged `ubuntu:24.04` container on GitHub-hosted runners, so the builder
  signature check is disabled for them (a documented deviation from section A of the standards).

Available series and files: catalog pages linked below, or
`https://images.openimages.cloud/<id>/latest/`. Verify them like any other image
([Verifying images](verifying-images.md)); the signing workflow is `build-dib-image.yml`, which
the common identity expression already matches.

The repository READMEs are the reference for each image; this page summarizes them.

## Octavia amphora

- Catalog: <https://openimages.cloud/images/octavia-amphora/>, repository:
  [open-img-cloud/octavia-amphora](https://github.com/open-img-cloud/octavia-amphora).
- The amphora-haproxy image used by [Octavia](https://docs.openstack.org/octavia/latest/) as
  load-balancer instances, built with Octavia's own `diskimage-create` from the
  `stable/<series>` branch with the matching upper-constraints.
- File name: `amphora-<series>-amd64-haproxy.qcow2`.
- No upstream watcher: the maintainers bump `VERSION` when a new series ships.

Import with the tag Octavia looks up (`amp_image_tag` in `[controller_worker]`, usually
`amphora`):

```sh
V=<series>
curl -fLO https://images.openimages.cloud/octavia-amphora/$V/amphora-$V-amd64-haproxy.qcow2
openstack image create \
  --disk-format qcow2 --container-format bare \
  --tag amphora \
  --file amphora-$V-amd64-haproxy.qcow2 \
  "amphora-$V-amd64-haproxy"
```

New load balancers use the newest image with that tag; existing ones move with
`openstack loadbalancer failover <lb_id>`.

## Manila service image

- Catalog: <https://openimages.cloud/images/manila-service-image/>, repository:
  [open-img-cloud/manila-service-image](https://github.com/open-img-cloud/manila-service-image).
- Service image of the Manila **generic driver** (NFS and CIFS share servers), built like
  upstream's default image from
  [`openstack/manila-image-elements`](https://opendev.org/openstack/manila-image-elements) at a
  pinned commit: Ubuntu 22.04 minimal, `nfs-kernel-server`, Samba, `qemu-guest-agent`.
- File name: `manila-service-image-<series>-amd64.qcow2`.
- Differences from upstream: **no password** for the `manila` user (upstream bakes one in); the
  SSH key is fetched from the metadata service (`169.254.169.254`) at boot. There is no cloud-init
  and no ConfigDrive support, so the metadata service must be reachable from the service network.

```sh
V=<series>
curl -fLO https://images.openimages.cloud/manila-service-image/$V/manila-service-image-$V-amd64.qcow2
openstack image create \
  --disk-format qcow2 --container-format bare \
  --property hw_qemu_guest_agent=yes \
  --file manila-service-image-$V-amd64.qcow2 \
  manila-service-image
```

In the generic back end of `manila.conf`: `service_image_name = manila-service-image`,
`service_instance_user = manila`, a key pair (`manila_service_keypair_name`,
`path_to_public_key`, `path_to_private_key`) and **no** `service_instance_password`. A complete
example is in the repository README; upstream reference:
[generic driver](https://docs.openstack.org/manila/latest/admin/generic_driver.html).

## Trove guest image

- Catalog: <https://openimages.cloud/images/trove-guest-image/>, repository:
  [open-img-cloud/trove-guest-image](https://github.com/open-img-cloud/trove-guest-image).
- Guest image for [Trove](https://docs.openstack.org/trove/latest/): Ubuntu 24.04 minimal with
  the Trove guest agent (from the pinned `stable/<series>` commit of `openstack/trove`) and
  Docker CE. The database runs in a container the guest agent pulls from the registry the operator
  configures (`docker_image`, `backup_docker_image`); no datastore image is embedded.
- File name: `trove-guest-image-<series>-amd64.qcow2`. The guest agent must match your Trove
  control plane: use the image of your series.
- Differences from upstream: no password for `ubuntu` and root (SSH keys only), pinned Trove,
  requirements and Docker versions, never a `dev_mode` image. Bigger than the 1 GB size target on
  purpose (Docker CE and the agent virtualenv).

```sh
V=<series>
curl -fLO https://images.openimages.cloud/trove-guest-image/$V/trove-guest-image-$V-amd64.qcow2
openstack image create trove-guest-image-$V \
  --private --disk-format qcow2 --container-format bare \
  --property hw_rng_model=virtio \
  --tag trove \
  --file trove-guest-image-$V-amd64.qcow2
# One image for every datastore version, selected by tag, for example:
openstack datastore version create 8.4 mysql mysql "" \
  --image-tags trove --active --default
```

In `trove.conf`, set `nova_keypair` (the only way into the instances) and
`management_networks`. Upstream references:
[building guest images](https://docs.openstack.org/trove/latest/admin/building_guest_images.html)
and [running Trove in production](https://docs.openstack.org/trove/latest/admin/run_trove_in_production.html).
Note that `hw_rng_model=virtio` requires a flavor that allows it.

## Testing status

None of the three has a runtime test yet (`runtime_smoke_login_user` is unset): the shared
contract (cloud-init, hostname from metadata, disk growth, IPv6) does not describe how a service
boots them. Dedicated checks are planned and described in the Manila and Trove READMEs.
