# Lab: Container Image Registries (Podman + Skopeo on WSL2)

> Based on *Red Hat OpenShift Development I: Introduction to Containers with Podman*, Chapter 3 - Container Image Registries.
> Environment: Windows + WSL2 (Ubuntu), rootless Podman.

## Objectives

- Understand what a container registry is and which ones exist
- Read and build a fully qualified image name
- Configure which registries Podman searches (`registries.conf`)
- Inspect and copy images with **Skopeo** (without pulling them locally)
- Authenticate to a registry and understand where credentials are stored

---

## 0. Setup

```bash
sudo apt update
sudo apt install -y podman skopeo
podman --version
skopeo --version
```

Check that rootless Podman works:

```bash
podman info --format '{{.Host.Security.Rootless}}'   # expected: true
podman run --rm quay.io/podman/hello
```

---

## 1. Theory in 2 minutes

A **registry** is a server that stores and distributes container images.

| Registry | Hosts | Auth needed? |
|---|---|---|
| `registry.access.redhat.com` | Red Hat images (UBI, etc.) | No |
| `registry.redhat.io` | Red Hat images (RHEL-based products) | **Yes** (Red Hat account) |
| `quay.io` | Public/custom images, free public repos | Only to push / private repos |
| `docker.io` | Docker Hub | Only to push / private repos |

Search images in the [Red Hat Ecosystem Catalog](https://catalog.redhat.com/software/containers/explore): it shows the Containerfile, packages, security scan and tags.

### Anatomy of an image name

```
registry.access.redhat.com / ubi9 / nodejs-18 : latest
 \_______________________/   \___/   \_______/   \____/
        registry URL        user/org  repository   tag
```

A name without the registry URL (e.g. `ubi9/python-39`) is **unqualified**. Podman then looks at `registries.conf` to decide where to search. **Best practice: always use the fully qualified name** to avoid pulling an unintended (possibly malicious) image.

---

## 2. Pull images and compare names

```bash
# Fully qualified (recommended)
podman pull registry.access.redhat.com/ubi9/ubi:latest

# Check what you have
podman images
```

Other useful UBI images from the course:

| Image | Provides |
|---|---|
| `registry.access.redhat.com/ubi9` | UBI 9 base image (RHEL 9 based) |
| `registry.access.redhat.com/ubi9/python-39` | Python 3.9 runtime |
| `registry.access.redhat.com/ubi9/nodejs-18` | Node.js 18 runtime |
| `registry.access.redhat.com/ubi9/go-toolset` | Go runtime |

> UBI images are free to use and redistribute, no Red Hat subscription needed. Some tags may be deprecated over time: if one is missing, pick another tag in the Ecosystem Catalog.

---

## 3. Configure registries with `registries.conf`

Podman reads, in order of priority:

1. `~/.config/containers/registries.conf` (user, overrides the global one)
2. `/etc/containers/registries.conf` (global)

### 3.1 Create a user config

```bash
mkdir -p ~/.config/containers
cat > ~/.config/containers/registries.conf <<'EOF'
# Search the Red Hat registry first, then Docker Hub
unqualified-search-registries = ['registry.access.redhat.com', 'docker.io']
EOF
```

Now an unqualified pull is resolved using that order:

```bash
podman pull ubi9/python-39
```

If several registries match, Podman asks you to choose:

```bash
podman pull httpd
# ? Please select an image:
#   ▸ registry.access.redhat.com/httpd:latest
#     docker.io/library/httpd:latest
```

### 3.2 Block a registry

```bash
cat >> ~/.config/containers/registries.conf <<'EOF'

[[registry]]
location = "docker.io"
blocked = true
EOF
```

Test it (should fail):

```bash
podman pull docker.io/library/alpine
```

Remove the block when done (edit the file and delete the `[[registry]]` section).

> On Windows without WSL, the global file lives inside the Podman VM: `podman machine ssh` then `ls /etc/containers/registries.conf`.

---

## 4. Skopeo: inspect and copy without local storage

Skopeo uses the `transport:image` format: `docker://`, `dir:`, `oci:`.

### 4.1 Inspect a remote image (no pull)

```bash
skopeo inspect docker://registry.access.redhat.com/ubi9/nodejs-18
```

Handy filters:

```bash
# Only the tags list
skopeo list-tags docker://registry.access.redhat.com/ubi9/ubi

# Only selected fields (needs jq)
skopeo inspect docker://registry.access.redhat.com/ubi9/nodejs-18 | jq '{Name, Digest, Created, Architecture, Os}'
```

### 4.2 Copy an image to a local directory

```bash
mkdir -p ~/images
skopeo copy \
  docker://registry.access.redhat.com/ubi9/nodejs-18 \
  dir:$HOME/images/nodejs-18

ls ~/images/nodejs-18     # manifest.json + layer blobs
```

### 4.3 Copy an image between registries (optional, needs a Quay.io account)

```bash
podman login quay.io
skopeo copy \
  docker://registry.access.redhat.com/ubi9/nodejs-18 \
  docker://quay.io/<YOUR_USER>/nodejs-18
```

Other things Skopeo can do: sign images with OpenPGP, convert formats (Docker -> OCI).

---

## 5. Authentication

`registry.redhat.io` requires login. Without it:

```bash
podman pull registry.redhat.io/rhel9/httpd-24
# Error: ... unable to retrieve auth token: invalid username/password: unauthorized
```

Two options: use an image from `registry.access.redhat.com`, or log in:

```bash
podman login registry.redhat.io
# Username / Password (the password is not echoed, just type it and press Enter)
podman pull registry.redhat.io/rhel9/httpd-24
```

### Where are the credentials stored?

```bash
echo $XDG_RUNTIME_DIR                       # e.g. /run/user/1000
cat ${XDG_RUNTIME_DIR}/containers/auth.json
```

> If `XDG_RUNTIME_DIR` is empty in your WSL, Podman falls back to `/run/user/<UID>`: check with `podman info --format '{{.Store.RunRoot}}'`.

The `auth` field is only **base64-encoded, not encrypted**. Demonstration with a **fake** value:

```bash
echo -n 'user:hunter2' | base64          # dXNlcjpodW50ZXIy
echo -n dXNlcjpodW50ZXIy | base64 -d     # user:hunter2
```

Skopeo reads the same `auth.json`, so one login works for both tools.

Log out when you are done:

```bash
podman logout registry.redhat.io
```

---

## 6. Cleanup

```bash
podman rmi --all
rm -rf ~/images
rm ~/.config/containers/registries.conf   # only if you want to restore defaults
```

---

## Cheat sheet

| Goal | Command |
|---|---|
| Pull an image | `podman pull <registry>/<org>/<repo>:<tag>` |
| List local images | `podman images` |
| Login / logout | `podman login <registry>` / `podman logout <registry>` |
| Inspect remote image | `skopeo inspect docker://<image>` |
| List remote tags | `skopeo list-tags docker://<repo>` |
| Copy between registries | `skopeo copy docker://<src> docker://<dst>` |
| Copy to local dir | `skopeo copy docker://<src> dir:<path>` |

## Key takeaways

- Always use **fully qualified image names**.
- `registries.conf` controls the search order and can **block** registries.
- Skopeo works on **remote** images without needing a daemon or local storage.
- Credentials live in `auth.json` (base64, not encrypted): **never commit it to Git**.

## Security note for this repo

Do **not** commit `auth.json`, tokens or real passwords. Add this to `.gitignore`:

```
auth.json
*.env
images/
```
