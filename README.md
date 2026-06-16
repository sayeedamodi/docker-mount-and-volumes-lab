# 🐳 Docker Storage: Images, Bind Mounts & Volumes

> A hands-on lab built by experimenting and breaking things. Every concept here was discovered by running real commands and seeing what happens.

---

## Table of Contents

- [The Big Picture](#the-big-picture)
- [Core Concepts (Read This First)](#core-concepts-read-this-first)
- [Experiment 1 — Images are Immutable Snapshots](#experiment-1--images-are-immutable-snapshots)
- [Experiment 2 — Bind Mounts](#experiment-2--bind-mounts)
- [Experiment 3 — Named Volumes](#experiment-3--named-volumes)
- [Experiment 4 — Bind Mount vs Named Volume](#experiment-4--bind-mount-vs-named-volume)
- [The Secret: First-Copy Behaviour](#the-secret-first-copy-behaviour)
- [Final Mental Model](#final-mental-model)
- [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)
- [Interview Cheat Sheet](#interview-cheat-sheet)

---

## The Big Picture

When you run a Docker container, something must supply the files at every path inside it.

By default, that source is the **image**. But Docker lets you **override** any path at runtime by mounting something else there — a folder from your laptop, or a Docker-managed volume.

The application inside the container never knows the difference. It just sees `/app`. Docker decides what backs it.

```
Source options for a path like /app:
─────────────────────────────────────
  [1] Image filesystem    (default, no flags)
  [2] Host folder         (bind mount: -v /host/path:/app)
  [3] Named volume        (docker volume:  -v mydata:/app)
```

This repo walks you through all three — with real commands you can run right now.

---

## Core Concepts (Read This First)

### What is a Docker Image?

A Docker **image** is a frozen snapshot of a filesystem. Think of it like a ZIP file of your app and everything it needs to run.

- Built with `docker build`
- Read-only — you cannot modify it after building
- Can be run as many times as you want
- Each run creates a new **container**

### What is a Container?

A **container** is a running instance of an image. Docker adds a thin writable layer on top of the image for that container's own changes (new files, edited files).

Key fact: **when the container is deleted, that writable layer is gone forever.**

```
Container lifecycle:
─────────────────────────────────────────
  docker run image    → container starts, writable layer created
  (container runs)    → you can write files, they go to writable layer
  docker stop         → container stops, writable layer still exists
  docker rm           → container deleted, writable layer destroyed ❌
```

### Why Does This Matter?

If your app writes data (logs, uploads, a database) to the container's writable layer, you **lose that data** when the container is removed. This is why mounts and volumes exist — to give your data somewhere persistent to live.

---

## Experiment 1 — Images are Immutable Snapshots

**Goal:** Prove that editing your source code does not change an already-built image.

### Setup

```
01-image-is-immutable/
├── Dockerfile
└── app.py
```

**`app.py`**
```python
print("Hello World this is v1")
```

**`Dockerfile`**
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY app.py .
CMD ["python", "app.py"]
```

### Steps

```bash
# Step 1: Build image and run it
cd 01-image-is-immutable
docker build -t mount-test-1 .
docker run mount-test-1
# Output: Hello World this is v1

<img width="350" height="43" alt="Screenshot 2026-06-16 183447" src="https://github.com/user-attachments/assets/e4ca8e13-a94f-4442-8f9c-a95c468c93cc" />

```

Now edit `app.py` to say v2, build a *second* image, and run both:

```bash
# Edit app.py -> print("Hello World this is v2")

# Step 2: Build a new image with the updated code
docker build -t mount-test-2 .

# Step 3: Run both images
docker run mount-test-1   # → Hello World this is v1
docker run mount-test-2   # → Hello World this is v2
```
<img width="350" height="43" alt="Screenshot 2026-06-16 183447" src="https://github.com/user-attachments/assets/b3d4bc88-9ea0-46bb-a725-9971568c6b7f" />

<img width="274" height="46" alt="Screenshot 2026-06-16 184023" src="https://github.com/user-attachments/assets/1c6dcc2d-422a-4a5d-93b2-847f11152e53" />

### What Happened


`mount-test-1` still prints v1. You edited `app.py` after building it, but that doesn't matter — the image already has its own frozen copy of `app.py` from build time.

```
WRONG mental model:
  Image → points to your files → builds on demand

CORRECT mental model:
  Image → frozen copy of your files at build time
  Your files → completely separate after that
```

> **Rule:** Every time you want to ship new code, you must `docker build` a new image. The old image is untouched.

### Why This Is a Good Thing

Immutability means your images are **reproducible**. `mount-test-1` will print v1 today, tomorrow, and in three years — on any machine. That's the whole point of Docker.

---

## Experiment 2 — Bind Mounts

**Goal:** Skip rebuilding. Use a bind mount so the container reads your files directly from your laptop.

### What is a Bind Mount?

A bind mount says: "instead of using the image's files at `/app`, use this folder from my host machine."

```bash
docker run -v /absolute/host/path:/container/path image
# or, using $(pwd) for the current directory:
docker run -v $(pwd):/app image
```

The container's `/app` now shows exactly what's on your disk, live.

### Setup

```
02-bind-mount/
├── Dockerfile
└── app.py       ← same Dockerfile as before
```

### Steps

```bash
cd 02-bind-mount

# Step 1: Run with a bind mount (no rebuild needed)
docker run --rm -v $(pwd):/app mount-test-2
# Output: Hello World this is v2

<img width="165" height="41" alt="Screenshot 2026-06-16 183738" src="https://github.com/user-attachments/assets/47c95d9e-f5b5-485a-8f04-44340d168a55" />


# Step 2: Edit app.py on your laptop (no docker build!)
# Change it to: print("Hello World this is from mount point")
<img width="236" height="67" alt="Screenshot 2026-06-16 184036" src="https://github.com/user-attachments/assets/b40c363c-64db-4e62-a570-41d18d79e93d" />

# Step 3: Run again — no rebuild
docker run --rm -v $(pwd):/app mount-test-2
# Output: Hello World this is from mount point

<img width="311" height="94" alt="Screenshot 2026-06-16 184048" src="https://github.com/user-attachments/assets/0c406fec-2432-4988-8b7f-6c8c78cc095e" />

```

### What Happened

You changed `app.py` on your laptop and the container immediately saw the new version — **without a single `docker build`.**

The `-v $(pwd):/app` flag mounted your current directory over the image's `/app`. The image's `/app` is still there, it's just hidden behind your mount.

```
Without bind mount:
  Image /app  ──────────────→  Container sees: v2 code

With bind mount (-v $(pwd):/app):
  Host folder ──────────────→  Container sees: latest code from your disk
  Image /app                   (hidden, shadowed by the mount)
```

### When to Use Bind Mounts

✅ **Local development** — edit code on your laptop, container picks it up instantly  
✅ **Injecting config** — pass a config file from host to container  
✅ **Reading host files** — let the container process files from your machine  

❌ **Production** — tied to the host's filesystem, not portable  
❌ **Data that needs to outlive the host** — if you move to a different machine, the data stays behind

### The `--rm` Flag

Notice the `--rm` above. It auto-deletes the container after it exits so you don't accumulate dead containers. Use it any time you're just testing.

---

## Experiment 3 — Named Volumes

**Goal:** Prove that data written inside a named volume survives container deletion.

### What is a Named Volume?

A **named volume** is a piece of storage that Docker manages for you. Unlike a bind mount, you don't point it at a specific folder on your laptop — Docker decides where it lives.

```bash
docker volume create mydata           # create a volume
docker run -v mydata:/app image       # mount it into a container
docker volume rm mydata               # delete it (and all its data)
```

Volumes survive container deletion. You can attach the same volume to a new container and all the data is still there.

### Setup

```
03-named-volume/
├── Dockerfile
└── app.py
```

### Steps

```bash
# Step 1: Create a named volume
docker volume create mydata

# Step 2: Run a container interactively and mount the volume
docker run -it -v mydata:/app mount-test-2 bash

# Step 3: Inside the container — create some files
mkdir data
echo "first"  > data/a.txt
echo "second" > data/b.txt
ls data        # a.txt  b.txt
exit           # container is now stopped and will be removed
```
<img width="421" height="221" alt="image" src="https://github.com/user-attachments/assets/85a8fb6e-afc9-4eeb-bb73-72529bb43af9" />


Now start a **completely new container** — the old one is gone:

```bash
# Step 4: New container, same volume
docker run -it -v mydata:/app mount-test-2 bash

# Step 5: Check if the files are still there
cd /app/data
ls             # a.txt  b.txt  ← still here!
cat a.txt      # first
cat b.txt      # second
exit
```

### What Happened

The files you created in the first container are still there in the second container. The **volume**, not the container, owns the data.

```
Container 1 → writes to mydata:/app/data → container deleted
Container 2 → mounts same mydata         → data still there ✅
```

### When to Use Named Volumes

✅ **Databases** — MySQL, Postgres, MongoDB data must survive restarts  
✅ **User uploads** — files users upload to your app  
✅ **Logs** — persistent log storage across deploys  
✅ **Production** — Docker manages the storage, it works on any machine  

❌ **Local dev with live code editing** — use a bind mount instead; volumes don't give you instant host-to-container sync in the same way  

### Useful Volume Commands

```bash
docker volume create mydata       # create
docker volume ls                  # list all volumes
docker volume inspect mydata      # see where Docker stores the data
docker volume rm mydata           # delete volume (and its data!)
docker volume prune               # delete all unused volumes
```

---

## Experiment 4 — Bind Mount vs Named Volume

**Goal:** Understand the critical difference between how each type of mount handles existing image data.

### What Happens to the Image Files?

In both cases, when you mount something to a path that already exists in the image, **the image's files are hidden**. The mount takes over completely.

But *how* each mount type gets populated is very different.

### Bind Mount Behaviour

```bash
docker run -v $(pwd):/app image
```

- Docker mounts your host folder
- Image files at `/app` are immediately hidden
- **Nothing is copied** from the image to the host
- If your host folder is empty, the container sees an empty `/app`

```
Host folder (might be empty)
         │
         ▼
  Container /app       ← your host folder, as-is
  Image /app (hidden)
```

### Named Volume Behaviour

```bash
docker run -v mydata:/app image
```

This is where it gets interesting. Named volumes have two modes:

**First run (volume is empty):**

```
Image /app  ──── copied into ────→  Volume (mydata)
                                         │
                                         ▼
                                   Container /app
```

Docker sees the volume is empty and seeds it with the image's files. **This only happens once.**

**Every run after that (volume has data):**

```
Volume (mydata, already has data)
         │
         ▼
  Container /app       ← volume contents
  Image /app (hidden)
```

Docker mounts the volume as-is. No copying. The image files are hidden.

### Side-by-Side Comparison

| Feature | Bind Mount | Named Volume |
|---|---|---|
| Storage lives on | Your host machine | Docker-managed location |
| Image files hidden? | Yes | Yes |
| First-copy from image? | ❌ Never | ✅ Once (if empty) |
| Data persists? | Yes (on host disk) | Yes (in volume) |
| Great for local dev? | ✅ Yes | ❌ Not ideal |
| Great for production? | ❌ Risky | ✅ Yes |
| Portable across machines? | ❌ No | ✅ Yes |

---

## The Secret: First-Copy Behaviour

This is the thing that trips up almost everyone learning Docker storage. Read it slowly.

**Named volumes seed themselves from the image — but only once, and only when empty.**

### Why Does This Matter?

Imagine you have a container for Postgres. The image ships with a default `/var/lib/postgresql/data` directory structure inside it.

```bash
# First run — volume is empty, Docker copies image files into it
docker run -v pgdata:/var/lib/postgresql/data postgres

# Second run — volume already has data, Docker just mounts it
docker run -v pgdata:/var/lib/postgresql/data postgres
```

On the first run, Postgres initialises its database into the volume. On every subsequent run, it finds that data already there. This is why Postgres (and most databases in Docker) work correctly with named volumes out of the box.

**Bind mounts don't do this.** If you used a bind mount and your host folder was empty, Postgres would find an empty data directory and try to initialise again — or fail.

### The Gotcha

If your volume already has old data and you update your image with new default files, **the volume will not update**. It keeps whatever was seeded into it the first time.

```bash
# ⚠️ This will NOT update the volume with new image defaults
docker run -v mydata:/app new-version-of-image

# To start fresh, delete the volume first
docker volume rm mydata
docker run -v mydata:/app new-version-of-image   # now seeds from new image
```

---

## Final Mental Model

The application inside the container does not know or care where `/app` comes from. Docker decides at runtime.

```
┌─────────────────────────────────────────────────┐
│                                                 │
│  No mount (default)                             │
│                                                 │
│  Image /app ──────────────► Container /app      │
│                                                 │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│                                                 │
│  Bind Mount   (-v $(pwd):/app)                  │
│                                                 │
│  Host Folder ─────────────► Container /app      │
│  Image /app (hidden)                            │
│                                                 │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│                                                 │
│  Named Volume   (-v mydata:/app)                │
│                                                 │
│  Volume (mydata) ─────────► Container /app      │
│  Image /app (hidden)                            │
│                                                 │
│  * If volume was empty on first run:            │
│    image /app was copied into it first          │
│                                                 │
└─────────────────────────────────────────────────┘
```

Same image. Same container. Different storage backend. The application runs identically in all three cases — it just reads and writes to `/app` without knowing what backs it.

---

## Quick Reference Cheat Sheet

### Building and Running

```bash
docker build -t my-image .             # build image from Dockerfile in current dir
docker run my-image                    # run container from image (default: image filesystem)
docker run --rm my-image               # auto-delete container on exit
docker run -it my-image bash           # interactive terminal inside container
```

### Bind Mounts

```bash
docker run -v $(pwd):/app my-image         # mount current dir to /app
docker run -v /absolute/path:/app image    # mount absolute path
docker run -v $(pwd)/config:/etc/config image   # mount a specific subfolder
```

### Named Volumes

```bash
docker volume create mydata            # create a named volume
docker volume ls                       # list all volumes
docker volume inspect mydata           # see details (including where data lives)
docker volume rm mydata                # delete volume (data gone!)
docker volume prune                    # delete all unused volumes

docker run -v mydata:/app my-image     # mount named volume to /app
```

### Useful Combinations

```bash
# Run, mount, and auto-clean up
docker run --rm -v $(pwd):/app my-image

# Interactive shell with volume mounted
docker run -it --rm -v mydata:/app my-image bash

# Production-style: named volume, detached, restart on crash
docker run -d --restart unless-stopped -v pgdata:/var/lib/postgresql/data postgres
```

### Inspecting What's Mounted

```bash
docker inspect container_name | grep -A 10 Mounts    # see all mounts for a container
docker volume inspect mydata                           # see where volume data lives on host
```

---

## Interview Cheat Sheet

These are the exact questions interviewers ask, and what they want to hear.

**Q: What is a Docker image vs a container?**

> An image is an immutable snapshot of a filesystem, built with `docker build`. A container is a running instance of that image — Docker adds a thin writable layer on top for that container's changes. When the container is deleted, the writable layer is lost.

**Q: What happens to data written inside a container?**

> It goes into the container's writable layer, which is deleted when the container is removed. To persist data, you use a bind mount or a named volume.

**Q: What is a bind mount?**

> A bind mount maps a path on the host machine to a path inside the container. The host folder replaces (shadows) the image's files at that path. Changes on the host are immediately visible inside the container, no rebuild needed. Best for local development.

**Q: What is a named volume?**

> A Docker-managed piece of storage identified by a name. Unlike bind mounts, you don't control where on the host it lives — Docker handles that. Data in a volume persists after the container is deleted. Best for production databases and persistent app data.

**Q: What is the difference between a bind mount and a named volume?**

> Both hide the image's files at the mounted path. The key difference is the **first-copy behaviour**: named volumes, when empty, are seeded from the image on the first run. Bind mounts never do this — if the host folder is empty, the container sees an empty path. Also: bind mounts are tied to a specific host location, volumes are portable.

**Q: How do you persist a database in Docker?**

> Use a named volume:
> ```bash
> docker run -v pgdata:/var/lib/postgresql/data postgres
> ```
> The database data goes into the `pgdata` volume. Even if you delete the container, the data is still in the volume. The next container that mounts `pgdata` picks up exactly where the last one left off.

---


*Built by learning hands-on and writing down everything that was confusing. If something was unclear, that's a bug — open an issue.*
