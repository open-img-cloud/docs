# Security policy

This policy covers the images published on <https://images.openimages.cloud>, the build
pipeline in the [open-img-cloud](https://github.com/open-img-cloud) organization, the website
and `oic-sync`.

## Reporting a vulnerability

Do **not** describe a vulnerability in a public issue.

A private reporting channel is not set up yet: GitHub private vulnerability reporting is not
enabled on the organization's repositories and no security contact is published. Until it is,
open a public issue in [`open-img-cloud/.github`](https://github.com/open-img-cloud/.github/issues)
titled "Security contact request", without any detail, and a maintainer will arrange a private
channel.

Vulnerabilities in the software inside an image (kernel, OpenSSL, cloud-init...) belong to the
upstream distribution: report them there. The project republishes fixed images when the
distribution releases them; the daily upstream watchers pick up new releases.

## Supported versions

Only the latest published version of each catalog image is maintained; the version is listed on
its page on <https://openimages.cloud/images/> and in
[`index.json`](https://images.openimages.cloud/index.json). Images with status `archive` follow
an upstream that is end of life or in extended support, as noted on their page.

## Verifying what you run

Every image is signed with cosign keyless signatures bound to the open-img-cloud build
workflows: see [Verifying images](docs/verifying-images.md).
