# Run your own mirror

`oic-sync mirror` copies the signed catalog to a directory or an S3 bucket of your own, for clouds
that should not, or cannot, download from `images.openimages.cloud`: internal networks, air-gapped
sites, or simply to keep the images next to the cloud. The mirror has the layout of the CDN, so
its base URL is a drop-in replacement for `https://images.openimages.cloud`.

The [oic-sync README](https://github.com/open-img-cloud/oic-sync#mirror) is the reference; a
guide with the same content is on the website:
[openimages.cloud/oic-sync/mirror](https://openimages.cloud/oic-sync/mirror/).

## Install

```sh
pipx install oic-sync
# S3 bucket destination: the s3 extra (boto3)
pipx install 'oic-sync[s3]'
```

The Debian and RPM packages and the container image of each
[release](https://github.com/open-img-cloud/oic-sync/releases) include both extras. The
[Ansible collection](https://github.com/open-img-cloud/ansible-collection-openimages) has an
`oic_mirror` role that installs oic-sync, creates a user and the directory, and sets up a daily
systemd timer and, optionally, an nginx server block.

## Mirror to a directory

```sh
# What would be downloaded (signatures verified, nothing written)
oic-sync mirror --dest /srv/oic-mirror --base-url https://mirror.example.net/oic \
  --image debian-13 --image ubuntu-24.04 --dry-run
# Do it
oic-sync mirror --dest /srv/oic-mirror --base-url https://mirror.example.net/oic \
  --image debian-13 --image ubuntu-24.04
# Every image except the service ones, two versions each, delete what falls out of --keep
oic-sync mirror --dest /srv/oic-mirror --base-url https://mirror.example.net/oic \
  --keep 2 --prune
```

Serve the directory with any web server at `--base-url`, hiding the dot files (temporary
downloads and the lock):

```nginx
server {
    listen 443 ssl;
    server_name mirror.example.net;
    location ~ /\. { deny all; }
    location /oic/ {
        alias /srv/oic-mirror/;
    }
}
```

## Mirror to an S3 bucket

With `--dest s3://bucket/prefix`, the mirror lives in AWS S3 or any S3-compatible object storage
(Ceph RGW, MinIO, OpenStack Swift with s3api...), and the bucket serves it: no web server.

```sh
export AWS_ACCESS_KEY_ID=... AWS_SECRET_ACCESS_KEY=...   # or a profile in ~/.aws
oic-sync mirror --dest s3://images/oic --s3-endpoint-url https://s3.example.net \
  --base-url https://s3.example.net/images/oic --keep 2 --prune
```

- Credentials, region and endpoint come from the usual AWS settings; `--s3-endpoint-url`
  overrides the endpoint.
- Each qcow2 is downloaded and verified in a local staging directory (`--staging-dir`), then
  uploaded: the bucket never holds an unverified image. Plan local space for the largest image
  (2 GiB).
- Objects get the cache headers of the CDN and carry their sha256 and sha512 as metadata, so
  later runs compare them instead of downloading images back.
- The bucket needs read access for your clients on the prefix; oic-sync does not change bucket
  policies.

## Use the mirror

```sh
export OIC_INDEX_URL=https://mirror.example.net/oic/index.json
oic-sync verify --image debian-13
oic-sync openstack sync --cloud mycloud --image debian-13
oic-sync proxmox sync --image debian-13 --vmid-base 9000
```

Glance, Proxmox VE and people can download from the mirror URLs directly; the checksums and
cosign bundles next to each qcow2 verify exactly as on the CDN
(see [Verifying images](verifying-images.md)).

## How the copy stays safe

1. The cosign bundle of each qcow2 is verified against the sha256 of the index, with the signer
   identity built into oic-sync, before anything is downloaded.
2. The file is downloaded to a temporary file, and its size, sha256 and sha512 are checked. A
   mismatch deletes it and fails the image; an interrupted download resumes.
3. Only then is it moved into place (or uploaded).
4. `index.json`, `versions.json` and `latest.json` are written last: an image whose new version
   fails keeps its previous entry, so the mirror never announces an incomplete version.

A lock stops two runs on the same mirror. Clients verify signatures against the signer built into
oic-sync, not the one the mirror index announces, so a tampered mirror cannot vouch for foreign
images. Plain `http://` base URLs are accepted for internal mirrors, since integrity rests on the
signed digests; HTTPS is still better, as someone on the path could serve an older signed
version.

## Run it daily

The catalog is rebuilt once a day. A systemd service and timer, as a user that owns the
directory:

```ini
# /etc/systemd/system/oic-sync-mirror.service
[Service]
Type=oneshot
User=oic-mirror
ExecStart=/usr/local/bin/oic-sync mirror --dest /srv/oic-mirror \
  --base-url https://mirror.example.net/oic --keep 2 --prune

# /etc/systemd/system/oic-sync-mirror.timer
[Timer]
OnCalendar=daily
RandomizedDelaySec=1h
Persistent=true

[Install]
WantedBy=timers.target
```

Exit codes: 0 every image mirrored, 1 at least one image failed (the others are still updated),
2 invalid arguments or index. The mirror host needs the CDN and the Sigstore trust root
(`tuf-repo-cdn.sigstore.dev`).
