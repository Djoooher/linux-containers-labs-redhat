# Lab: Managing Container Images + Moving Images Between Registries (Podman + Skopeo on WSL2)

> Based on *Red Hat DO188 - Introduction to Containers with Podman*, Chapter 3:
> **Managing Images** + **Guided Exercise: Container Image Registries**.
> Environment: Windows + WSL2 (Ubuntu), rootless Podman.

## Table of contents

1. [Objectives](#objectives)
2. [Part A - Managing images](#part-a---managing-images)
3. [Part B - Guided exercise adapted to WSL](#part-b---guided-exercise-adapted-to-wsl)
4. [Cleanup](#cleanup)
5. [Cheat sheet](#cheat-sheet)

## Objectives

- Version images with tags (and understand why `latest` is risky)
- Search, pull, list, inspect, build, push and remove images
- Export/import containers (`export`/`import`) vs images (`save`/`load`)
- Log in to registries, copy an image between two registries with Skopeo, and test it as an unauthenticated user

> **Adaptation note:** the official guided exercise uses a classroom environment (an OpenShift cluster + an internal Quay registry at `registry.lab.example.com:8443`). That does not exist on a personal WSL, so Part B reproduces the **same workflow** with local registries:
>
> | Classroom | My WSL equivalent |
> |---|---|
> | RHOCP registry (source) | `registry.access.redhat.com` (Red Hat public registry) |
> | Internal registry `registry.lab.example.com:8443` (private, needs login) | Local registry `localhost:5000` with htpasswd auth |
> | "Make Public" button in Quay UI | Second local registry `localhost:5001` without auth |
> | `oc login` steps | Skipped (OpenShift is covered later in the course) |

---

## Part A - Managing images

### A.1 Image versioning and tags

Image name format:

```
[registry/namespace/]image-name[:tag]
```

- Tags usually map to the software version (semantic versioning: `MAJOR.MINOR.PATCH`).
- No tag = `latest` by default. **`latest` is bad practice**: it can silently move to a backward-incompatible version.
- One image can have several tags.

```bash
# Pull with an explicit tag (good)
podman pull registry.access.redhat.com/ubi9/ubi-minimal:9.5

# Pull without a tag -> Podman uses :latest (avoid in real projects)
podman pull quay.io/argoproj/argocd

# Add a second tag to a local image (does NOT duplicate data, same image ID)
podman image tag registry.access.redhat.com/ubi9/ubi-minimal:9.5 localhost/my-ubi:v1
podman images | grep -E "ubi-minimal|my-ubi"
```

### A.2 Searching images

```bash
# Search in all registries of unqualified-search-registries (registries.conf)
podman search nginx

# Search in one specific registry
podman search registry.access.redhat.com/ubi9
podman search --limit 5 docker.io/nginx
```

Add a private registry to the default search list in `~/.config/containers/registries.conf`:

```toml
unqualified-search-registries = ['registry.example.com:8443', 'registry.redhat.io', 'docker.io']
```

### A.3 Pulling and listing images

```bash
podman pull registry.access.redhat.com/ubi9/python-311
podman image ls        # same as: podman images
```

Where are images stored?

| User | Location |
|---|---|
| Rootless (normal user) | `~/.local/share/containers` |
| root | `/var/lib/containers` (only visible with `sudo podman images`) |

```bash
ls ~/.local/share/containers
sudo podman images     # separate storage: will not show your rootless images
```

### A.4 Building an image

```bash
mkdir -p ~/lab-images && cd ~/lab-images

cat > Containerfile <<'EOF'
FROM registry.access.redhat.com/ubi9/ubi-minimal:9.5
LABEL description="Demo image for DO188 lab"
CMD ["echo", "Hello from my custom image"]
EOF

podman build --file Containerfile --tag localhost/hello-lab:1.0 .
podman run --rm localhost/hello-lab:1.0
```

To build an image destined for Quay.io you would tag it as `quay.io/YOUR_QUAY_USER/IMAGE_NAME:TAG`.

### A.5 Pushing an image

Rule: **log in first, then push**.

```bash
podman login quay.io
podman tag localhost/hello-lab:1.0 quay.io/YOUR_QUAY_USER/hello-lab:1.0
podman push quay.io/YOUR_QUAY_USER/hello-lab:1.0
```

(No Quay account? Part B pushes to a local registry instead.)

### A.6 Inspecting images

```bash
# Full metadata (verbose JSON)
podman image inspect registry.access.redhat.com/ubi9/python-311

# Extract one field with Go templates
podman image inspect registry.access.redhat.com/ubi9/python-311 --format="{{.Config.Cmd}}"
podman image inspect registry.access.redhat.com/ubi9/python-311 --format="{{.Config.User}}"
podman image inspect registry.access.redhat.com/ubi9/python-311 --format="{{.Config.WorkingDir}}"
podman image inspect registry.access.redhat.com/ubi9/python-311 --format="{{.Architecture}} / {{.Os}}"
podman image inspect registry.access.redhat.com/ubi9/python-311 --format="{{.Config.Env}}"
```

What the main fields mean:

| Key | Meaning |
|---|---|
| `Config.User` | Default user in the container |
| `Config.ExposedPorts` | Ports the app exposes |
| `Config.Env` | Environment variables |
| `Config.Entrypoint` / `Config.Cmd` | Command(s) run when the container starts |
| `Config.WorkingDir` | Working directory |
| `Config.Labels` | Extra metadata |
| `Architecture` / `Os` | Platform the image targets |

To inspect a **remote** image without pulling it, use Skopeo:

```bash
skopeo inspect docker://registry.access.redhat.com/ubi9/python-311
```

### A.7 Removing images

```bash
podman image rm localhost/my-ubi:v1          # or: podman rmi ...
podman image rm -f <image>                   # force: stops and removes containers using it
podman rmi --all                             # remove ALL local images (careful)

# Dangling images (no tag, not referenced by another image)
podman image prune                           # asks for confirmation
podman image prune -a                        # also removes unused images
podman image prune -af                       # no prompt
```

If an image is used by a container, a normal `rm` fails: stop/remove the container first, or use `-f`.

### A.8 Export/import a container vs save/load an image

| | `export` / `import` | `save` / `load` |
|---|---|---|
| Works on | a **container** | an **image** |
| Layers | **squashed** into one | **kept** |
| Metadata (history, labels, CMD...) | **lost** | **kept** |
| Use case | filesystem snapshot / backup | moving images between machines |

**Container -> tar -> image:**

```bash
podman run -d --name web registry.access.redhat.com/ubi9/httpd-24
podman export -o web.tar web
podman import web.tar httpdcustom:2.4
podman images | grep httpdcustom
```

**Image -> tar -> image:**

```bash
podman save --output ubi-minimal.tar registry.access.redhat.com/ubi9/ubi-minimal:9.5
podman rmi registry.access.redhat.com/ubi9/ubi-minimal:9.5 localhost/my-ubi:v1 localhost/hello-lab:1.0 2>/dev/null
podman load --input ubi-minimal.tar          # image name is restored automatically
```

Compare the two to see the difference in metadata:

```bash
podman image inspect localhost/httpdcustom:2.4 --format="{{.Config.Cmd}}"          # empty (metadata lost)
podman image inspect registry.access.redhat.com/ubi9/ubi-minimal:9.5 --format="{{.Config.Cmd}}"   # preserved
```

---

## Part B - Guided exercise adapted to WSL

**Scenario:** the Dev and Test pipelines must use the same Python image as the platform one. Copy the Python image from the source registry to an internal registry, make it public, and test it as an unauthenticated user.

### B.1 Prepare the "internal registry" (private, with login)

Create the credentials (user `developer` / password `developer`, lab only):

```bash
mkdir -p ~/lab-registry/auth
podman run --rm --entrypoint htpasswd docker.io/library/httpd:2 \
  -Bbn developer developer > ~/lab-registry/auth/htpasswd
```

Start the private registry on port 5000:

```bash
podman run -d --name registry-private -p 5000:5000 \
  -v ~/lab-registry/auth:/auth:Z \
  -e REGISTRY_AUTH=htpasswd \
  -e REGISTRY_AUTH_HTPASSWD_REALM="Lab Registry" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  docker.io/library/registry:2

curl -i http://localhost:5000/v2/      # expect: 401 Unauthorized
```

### B.2 Log in to the registries

```bash
# Source registry: Red Hat public registry (no login required)
# Destination registry: our private local registry
podman login -u developer -p developer --tls-verify=false localhost:5000
# Login Succeeded!
```

> `--tls-verify=false` is needed because the local registry uses plain HTTP. Never do this on a real registry.

### B.3 Copy the image with Skopeo

Source = `registry.access.redhat.com/ubi9/python-311` (equivalent of the RHOCP `python:3.11-ubi9` image).
Destination = `localhost:5000/developer/python:3.11-ubi9`.

```bash
skopeo copy --dest-tls-verify=false \
  docker://registry.access.redhat.com/ubi9/python-311:latest \
  docker://localhost:5000/developer/python:3.11-ubi9
```

Verify the image arrived (list tags in the destination):

```bash
skopeo list-tags --tls-verify=false docker://localhost:5000/developer/python
```

### B.4 "Make the image public"

A plain local registry has no visibility toggle (in Quay you click *Settings -> Make Public*). To simulate the **public** state, start a second registry **without authentication** and copy the image there:

```bash
podman run -d --name registry-public -p 5001:5000 docker.io/library/registry:2

skopeo copy --src-tls-verify=false --dest-tls-verify=false \
  docker://localhost:5000/developer/python:3.11-ubi9 \
  docker://localhost:5001/developer/python:3.11-ubi9
```

### B.5 Test as an unauthenticated user

Log out of everything first:

```bash
podman logout --all
# Removed login credentials for all registries
```

**Private registry -> must fail:**

```bash
podman pull --tls-verify=false localhost:5000/developer/python:3.11-ubi9
# Error: ... unauthorized: authentication required
```

**Public registry -> must work:**

```bash
podman pull --tls-verify=false localhost:5001/developer/python:3.11-ubi9
podman run --rm localhost:5001/developer/python:3.11-ubi9 python3 --version
# Python 3.11.x
```

That's the point of the exercise: without login, the private repo refuses the pull, while the public one allows it.

---

## Cleanup

```bash
podman rm -f registry-private registry-public web 2>/dev/null
podman rmi --all
rm -rf ~/lab-registry ~/lab-images web.tar ubi-minimal.tar
```

---

## Cheat sheet

| Goal | Command |
|---|---|
| Search images | `podman search <term>` |
| Pull / list | `podman pull <image>` / `podman image ls` |
| Add a tag | `podman image tag <src> <new>` |
| Build | `podman build --file Containerfile --tag <name:tag> .` |
| Login / push | `podman login <registry>` / `podman push <image>` |
| Inspect (local) | `podman image inspect <image> --format="{{.Config.Cmd}}"` |
| Inspect (remote) | `skopeo inspect docker://<image>` |
| Copy between registries | `skopeo copy docker://<src> docker://<dst>` |
| Remove image | `podman rmi <image>` / `podman rmi --all` |
| Remove dangling | `podman image prune` (`-a` all unused, `-f` no prompt) |
| Container -> tar / tar -> image | `podman export -o f.tar <ctr>` / `podman import f.tar name:tag` |
| Image -> tar / tar -> image | `podman save -o f.tar <img>` / `podman load -i f.tar` |

## Key takeaways

- Pin **explicit tags**; avoid `latest`.
- `podman tag` adds a name to the same image, it does not copy data.
- `export/import` = flat snapshot, metadata lost. `save/load` = full image, metadata kept.
- Skopeo copies between registries without pulling to local storage.
- Private repos need login; public ones can be pulled anonymously.
- Never commit `auth.json`, htpasswd files or real credentials to Git.

## Suggested `.gitignore`

```
auth.json
htpasswd
*.tar
*.env
```
