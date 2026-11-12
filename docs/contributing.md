# Contributing

## Reporting a problem or requesting an image

- A problem with one image (boot failure, missing package, wrong default): open an issue in that
  image's repository. The repository is linked from the image page on
  <https://openimages.cloud/images/> and listed in `index.json` (`repo`).
- A problem with the website, the registry or the build pipeline: open an issue in
  [`oic-website`](https://github.com/open-img-cloud/oic-website/issues),
  [`images-router`](https://github.com/open-img-cloud/images-router/issues) or
  [`.github`](https://github.com/open-img-cloud/.github/issues).
- A new image: open a request in
  [`.github`](https://github.com/open-img-cloud/.github/issues). Only images the project is
  allowed to redistribute can be added (for example, Red Hat Enterprise Linux is out; Rocky Linux
  and AlmaLinux are in the catalog instead).
- Security issues: see the [security policy](../SECURITY.md).

## Changing an existing image

Each image repository is small: `VERSION`, `build/` (build hook such as `customize.sh` or
`dib-build.sh`, and `detect-upstream.sh`), `.github/workflows/release.yml` and
`.github/workflows/watch.yml`. Fork, branch, open a pull request.

- A pull request can be tested without publishing anything: run `release.yml` with
  `workflow_dispatch` (`publish` defaults to false) on your branch.
- Keep changes to what a cloud image needs, and document deviations from the
  [image standards](https://github.com/open-img-cloud/.github/blob/main/docs/IMAGE_STANDARDS.md)
  at the top of the repository README, with a date and the reason.
- Do not ship a full replacement of `/etc/cloud/cloud.cfg`: distribution-specific cloud-init
  settings go in `build/config/99_oic-distro.cfg` (section C of the standards).

## Changing the shared platform

The reusable workflows and actions in [`.github`](https://github.com/open-img-cloud/.github) are
consumed at `@main` by every image repository: a merge there takes effect on the next build of
every image. Test a change from a branch first: point one image's `release.yml` at the branch
(`uses: ...@<branch>` and the `shared_actions_ref` input, which selects the checkout of the
composite actions) and run it as a dry run. If you add a build engine, update the cosign identity expression in the
standards (section A), in `oic-website/src/lib/images.ts`, and in every README that prints it.

GitHub Actions are pinned by commit SHA, with the version in a comment. The `.github` repository
has a Renovate configuration that keeps the pins current.

## Onboarding a new image (maintainers)

Requirements: sections A to F of the image standards are MUST, G to J SHOULD, K (runtime test)
MUST for general-purpose images. Steps:

1. **Repository.** Create `open-img-cloud/<id>` with `VERSION`, `build/`,
   `release.yml` calling one of the four reusable build workflows, `watch.yml` calling
   `upstream-watch.yml`, `LICENSE` (GPL-2.0) and a README. Use an existing repository of the
   same family as a model. If your global gitignore excludes `build/`, add `!/build/` to the
   repository `.gitignore` (see [CONVENTIONS.md](https://github.com/open-img-cloud/.github/blob/main/CONVENTIONS.md)).
   Set `runtime_smoke_login_user` in `release.yml` (leave it empty only for service images, and
   document a dedicated test).
2. **Watcher slot.** Give `watch.yml` a cron minute that no other repository uses (section G of
   the standards lists the current allocations).
3. **Bucket.** Create a public bucket named `<id>` with versioning on Infomaniak Object Storage,
   in the project that holds the other image buckets (maintainer access only).
4. **Router.** In `images-router/wrangler.toml`, add `<id>` to `ALLOWED_BUCKETS`, and
   `<repo>:watch.yml` to `SCHEDULED_WORKFLOWS` so the Worker dispatches the watcher daily. The
   Worker's dispatch token must also be allowed on the new repository.
5. **Website.** Add an entry to `oic-website/src/data/images.yaml`: `os_name`, `title`,
   `tagline`, `description`, `repo`, `latest_version`, `format`, `arch`, `tags`, `login_user`,
   `os_distro` (libosinfo name when one exists), optional `variants`, `logo`, `status`,
   `status_note`, `featured`. Images whose `os_distro` is not in the osism schema are left out of
   the osism export automatically.
6. **Organization profile.** Add a row to `.github/profile/README.md`.
7. **First release.** Merge, tag `v<VERSION>`, watch the release run (build, sign, publish,
   validate, runtime test), then check the image page and `index.json`.

Release secrets are set at the organization level, so a new repository inherits them; nothing
to configure per repository.

## This documentation

Plain Markdown, rendered by GitHub. Rules:

- Link to <https://openimages.cloud> and to `index.json` for anything that changes (versions,
  sizes, file names). Never hardcode current versions.
- Check every command, URL and property name against the code before writing it down.
- Links are checked by the `Links` workflow ([lychee](https://github.com/lycheeverse/lychee)) on
  every push, pull request and weekly.
