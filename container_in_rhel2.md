 ---

# Podman Ecosystem (RHEL Container Tools)

Podman is not a single tool.

Red Hat provides an entire ecosystem of container tools.

```text
                  Container Tools

                        │

        ┌───────────────┼───────────────┐

     Podman          Buildah         Skopeo

        │

     Toolbox

        │

      Udica
```

Each tool has a specific responsibility.

---

# 1. Podman

Podman is used to **run and manage containers**.

Think of Podman as the replacement for Docker on RHEL.

Responsibilities

- Run containers
- Stop containers
- Remove containers
- Pull images
- Manage images
- Manage volumes
- Manage networks

Example

```bash
podman run ubi9
```

Think of Podman as the **container manager**.

---

# 2. Buildah

## Why Buildah Exists

Suppose you want to create your own container image.

Example

You have developed a Python application.

Now you want to package

- Python
- Required libraries
- Your application

into one image.

Podman can build images, but Red Hat created **Buildah** specifically for building images.

Think of it like this

```text
Buildah

↓

Creates Images

↓

Podman

↓

Runs Images
```

Buildah never runs containers.

Its job is only to build container images.

---

### Example

Instead of downloading

```text
ubi9
```

you create

```text
my-python-image
```

using Buildah.

Later Podman runs

```text
my-python-image
```

---

### Analogy

Imagine cooking.

Buildah

```text
Cooks the food
```

Podman

```text
Serves the food
```

---

# 3. Skopeo

## Why Skopeo Exists

Suppose an image exists on

```text
registry.redhat.io
```

and you want to

- inspect it
- copy it
- verify it

without downloading it.

Podman cannot efficiently do this.

Skopeo was created for image management between registries.

---

Example

```text
Registry A

↓

Copy Image

↓

Registry B
```

No container is created.

No image is downloaded locally (unless you choose to).

---

### Skopeo can

- Inspect remote images
- Copy images
- Delete images from registries
- Verify image signatures

Think of Skopeo as

```text
Image Transfer Tool
```

---

### Analogy

Podman

```text
Drives the car
```

Buildah

```text
Builds the car
```

Skopeo

```text
Ships the car
```

---

# 4. Toolbox

## Why Toolbox Exists

Containers are intentionally isolated.

Sometimes, as a developer or administrator, you want a safe environment to install packages and test software without modifying your host RHEL system.

Toolbox creates a container that behaves almost like your normal user environment.

Inside Toolbox you can install packages, test commands, or experiment without affecting the host OS.

---

Example

Instead of installing

```text
gcc

python

vim

git
```

directly on RHEL,

you install them inside a Toolbox container.

If you break the environment,

delete the Toolbox container and create a new one.

Your host remains clean.

---

Think of Toolbox as

```text
Temporary Linux Workspace
```

---

### Toolbox is mainly used by

- Developers
- Test engineers
- Linux administrators
- DevOps engineers

---

# 5. Udica

## Why Udica Exists

SELinux protects your system by enforcing security policies.

Containers also run under SELinux.

Sometimes a container needs access to

- a host directory
- a device
- a network resource

The default SELinux policy may deny that access.

Instead of disabling SELinux,

Udica generates a **custom SELinux policy** specifically for that container.

This lets the container access only what it needs while keeping SELinux enabled.

---

Example

Container needs

```text
/backup
```

SELinux blocks access.

Instead of

```text
setenforce 0
```

you generate a custom policy with Udica and allow access safely.

---

Think of Udica as

```text
SELinux Policy Generator for Containers
```

---

# Relationship Between All Tools

```text
                Create Image

                    │

                 Buildah

                    │

              Container Image

                    │

      ┌─────────────┴─────────────┐

      │                           │

   Skopeo                    Podman

Copy/Inspect Images      Run Containers

                                 │

                           Running Container

                                 │

                     Toolbox (Developer Shell)

                                 │

                     SELinux Policy

                                 │

                              Udica
```

---

# Which Tools are Important for RHCSA?

| Tool | RHCSA Importance | Purpose |
|------|------------------|---------|
| Podman | ⭐⭐⭐⭐⭐ | Run and manage containers |
| Buildah | ⭐⭐☆☆☆ | Build container images |
| Skopeo | ⭐⭐☆☆☆ | Copy and inspect images |
| Toolbox | ⭐☆☆☆☆ | Developer toolbox container |
| Udica | ⭐☆☆☆☆ | Generate SELinux policies for containers |

---

# RHCSA Exam Tip

For the RHCSA (EX200) exam, **Podman is the primary focus**.

You should be comfortable with:

- Pulling images
- Running containers
- Managing containers
- Managing images
- Using volumes
- Configuring rootless containers

Buildah, Skopeo, Toolbox, and Udica are part of the RHEL container ecosystem and are good to recognize conceptually, but the RHCSA exam emphasizes Podman administration rather than advanced image creation or SELinux policy generation.