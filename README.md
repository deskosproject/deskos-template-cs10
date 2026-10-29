# DeskOS template for CentOS Stream 10

This repository builds your own DeskOS golden image for CentOS Stream 10. DeskOS is not a
distribution you consume: it is a compiler. The published DeskOS image exists so
you can try it; this repository is where you make your own, from your resources,
with your desktop.

Edit four YAML files, run three commands, and you have an image and an ISO. The
GitHub workflow is there if you would rather not build it yourself.

## 1. Get the resources

Use this template, public or private, or just copy the folder: **you do not need
GitHub to build this.** The repository is a folder of YAML plus one optional
workflow.

## 2. Make it yours

Everything is in `resources/deskos.yaml`, as several YAML documents separated by
`---`, plus the wallpaper. The compiler does not care about directories; this is one
file because one file is easier to read.

| In `resources/deskos.yaml` | What to change |
|---|---|
| `Workstation` | the name, `displayName` and the profiles it selects |
| `PackageSet` | the packages every user of the machine gets |
| `FlatpakSet` | Flatpak applications: LibreOffice from Flathub by default; change the id or add more |
| `GnomeProfile` | wallpaper, dark or light, favorite applications, window buttons |
| `UpdatePolicy` | how the image updates: daily or weekly, on battery or not |

`resources/background.svg` is the wallpaper the desktop document names. Replace it
with your own image and keep the name, or update the path. The file must stay inside
`resources/`, and the format is PNG, JPG, JPEG or SVG.

Anything else Core supports is added the same way, by writing a resource and
listing it in `resources/profiles/organization-baseline.yaml`: a repository, a
Flatpak channel, a binary, a CA certificate, a lock on a setting. The catalogue is
[the resource reference](https://github.com/deskosproject/deskos-core/blob/main/docs/resources.md).

## 3. Build it locally

Install the compiler once, then run the three commands:

```bash
curl -fsSLO https://github.com/deskosproject/deskos-core/releases/latest/download/deskosctl-linux-amd64
install -D -m 0755 deskosctl-linux-amd64 "$HOME/.local/bin/deskosctl"

deskosctl validate ./resources
deskosctl render ./resources --workstation my-workstation --output ./dist
sudo podman build -t localhost/my-workstation:test ./dist
```

`deskosctl plan ./resources --workstation my-workstation` prints what the image will
contain, and where each value came from, without building anything.

A `localhost/` image is enough to boot the machine and to build the ISO. It is **not
an update source**: the installed machine follows the reference baked into the ISO,
and no registry serves `localhost/...`.

So a registry is part of the deliverable, not an optional extra. Push the image
somewhere your machines can reach:

```bash
podman tag localhost/my-workstation:test registry.example.com/your-org/my-workstation:v1
podman push registry.example.com/your-org/my-workstation:v1
```

That can be any OCI registry you control: GitHub is optional, the registry is not.
The image there is what installed machines pull their updates from, and the ISO you
build in section 5 should name that reference.

## 4. Or let GitHub build the image

The workflow is **optional**. Use it if you want GitHub to build the image for you,
or ignore it and stop after section 3.

It needs nothing: it pushes with the `GITHUB_TOKEN` that GitHub gives every run,
under the `packages: write` permission it declares. **Push to `main`, or Actions,
build, Run workflow.** The first run takes about ten minutes, most of it pulling
the base and installing packages, and the image appears as `:latest` and
`:<commit>`.

Packages on `ghcr.io` are **private by default**, even in a public repository. To
let anyone pull yours, open the package settings and change the visibility.

## 5. Build the ISO

**From the image you built locally.** No pull and no digest:

```bash
image=localhost/my-workstation:test
```

**From a registry.** Pull it and resolve the digest, so a later push cannot change
what you install:

```bash
image=ghcr.io/<your-user>/<your-repo>:latest
sudo podman pull "$image"
digest="$(sudo podman image inspect --format '{{.Digest}}' "$image")"
image="ghcr.io/<your-user>/<your-repo>@$digest"
echo "$image"
```

Then build the ISO. The kickstart asks for language, keyboard, timezone, network,
the disk and LUKS, and erases **the disk you select**. Those screens come from
Anaconda.

You can pre-answer them, or change which ones appear, by editing the `config.toml`
above. The options DeskOS uses, and the ones to avoid, are in
[Installer ISO](https://github.com/deskosproject/deskos-core/blob/main/docs/installer-iso.md).

**Keep the `config.toml`.** Without it, bootc-image-builder writes its own
kickstart, which runs `clearpart --all` and `autopart`: the install asks nothing and
erases every disk the installer sees.

```bash
mkdir -p output
cat > config.toml <<'EOF'
[customizations.installer.modules]
enable = [
  "org.fedoraproject.Anaconda.Modules.Localization",
  "org.fedoraproject.Anaconda.Modules.Timezone",
]

[customizations.installer.kickstart]
contents = """
graphical
rootpw --lock
reboot
"""
EOF

sudo podman run --rm --privileged --pull=missing \
    --security-opt label=type:unconfined_t \
    -v ./output:/output -v ./config.toml:/config.toml:ro \
    -v /var/lib/containers/storage:/var/lib/containers/storage \
    quay.io/centos-bootc/bootc-image-builder:latest \
    build --type anaconda-iso --config /config.toml \
    --chown "$(id -u):$(id -g)" "$image"
```

The ISO lands in `output/bootiso/install.iso`. Boot it; GNOME Initial Setup creates
the first account.

## 6. What the installed machine updates from

It follows the reference the ISO was built from, recorded at install, and
`bootc upgrade` pulls that reference.

- **A `localhost/` image**: no registry serves it, so the machine does not update.
  Rebuild the ISO, or rebuild the image and `bootc switch` to a reference that
  exists.
- **A digest** (`@sha256:...`): the machine follows that digest. A new push does not
  reach it; update with a new ISO, or `bootc switch` on the machine.
- **A tag**: the machine follows the tag, and moving it is what offers the update.
  Use a version tag and move it on purpose; `:latest` is convenient for a lab and
  hard to audit for a fleet.

## 7. If the registry is private

`bootc-image-builder` records the image reference for later updates and nothing
else: **no credentials**. The ISO embeds the image, so installing does not contact
the registry, and the first `bootc upgrade` does.

The installed machine reads `/etc/ostree/auth.json` (containers auth format, mode
0600), or `/run/ostree/auth.json` for one boot:

```json
{ "auths": { "registry.example.com": { "auth": "base64-of-user:token" } } }
```

Create the value with `printf '%s' 'user:token' | base64`. For `ghcr.io` the user is
a GitHub account and the token a personal access token with `read:packages`; for
your registry, the same account the workflow pushes with, with read only.

Deliver the file from provisioning, never inside the image or the ISO. Anyone who
can read it has the token: scope it to read, rotate it, and prefer a registry token
over an account password.

## What this template does not do

- No signature, no provenance, no CVE scan and no SBOM. DeskOS's own
  [pipeline](https://github.com/deskosproject/deskos-core/tree/main/.github/workflows)
  is the reference for all of them.
- No tests beyond `deskosctl validate`.
- No ISO in the workflow: where it is stored and who signs it are your decisions.

## Where to read more

- [Installing DeskOS](https://github.com/deskosproject/deskos-core/blob/main/docs/install.md)
- [Resources](https://github.com/deskosproject/deskos-core/blob/main/docs/resources.md):
  every kind you can write.
- [Runtime](https://github.com/deskosproject/deskos-core/blob/main/docs/runtime.md):
  updates, GNOME, Flatpaks and `/etc`.
- [Platforms](https://github.com/deskosproject/deskos-core/blob/main/docs/platforms.md):
  what is verified on CentOS Stream 10, and what is not.
