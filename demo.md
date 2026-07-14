# CVMFS → SHPC → Shelley Demo

A live, follow-along tour of three tools that stack together:

- **CVMFS** mounts a shared, read-only library of scientific software/containers over HTTP — nothing is downloaded until you touch it.
- **SHPC** (Singularity Registry HPC) turns any container image into a normal `module load`-able command, including images already sitting in CVMFS.
- **Shelley** wraps SHPC into a single guided command, so building a module from a CVMFS container is one step instead of many.

---

## Part 1: CVMFS

### 1. The mountpoint before anything is mounted

```bash
cd /cvmfs
ls
```

Nothing is listed. `/cvmfs` is managed by **autofs** — repositories only appear the moment something tries to access them. An empty `ls /cvmfs` is expected: nothing is "mounted" until it's touched.

### 2. Probe the client

```bash
cvmfs_config probe
```

```text
Probing /cvmfs/data.galaxyproject.org... OK
```

This attempts to mount and stat every repository listed in `CVMFS_REPOSITORIES` (`/etc/cvmfs/default.local`) and reports OK/FAIL per repo. Only `data.galaxyproject.org` is configured right now — the **singularity** repo (`singularity.galaxyproject.org`) has deliberately been left out so we can add it back in live.

### 3. The configuration files

CVMFS reads config in layers. Walk through each file in place:

#### `/etc/cvmfs/default.conf`

```bash
cat /etc/cvmfs/default.conf
```

Shipped by the `cvmfs` package — global defaults for **all** repositories (cache location/quota, timeouts, retry/backoff, key directory). **Never edit this file directly**; override values in `default.local` instead.

```conf
CVMFS_CACHE_BASE=/var/lib/cvmfs
CVMFS_QUOTA_LIMIT=4000
CVMFS_SHARED_CACHE=yes
CVMFS_KEYS_DIR=/etc/cvmfs/keys
CVMFS_TIMEOUT=5
CVMFS_TIMEOUT_DIRECT=10
CVMFS_USE_GEOAPI=no
```

!!! note
    `CVMFS_USE_GEOAPI=no` is the package default — fine generically, but **not what we want for Galaxy**: the galaxyproject.org domain has Stratum 1 replicas spread across the globe (TACC, IU, PSU, JRC in the EU, GVL in Australia). Without GeoAPI, the client just walks the server list in the order it's written rather than picking the closest one. `default.local` overrides this back to `yes`.

#### `/etc/cvmfs/default.local`

```bash
cat /etc/cvmfs/default.local
```

```conf
CVMFS_REPOSITORIES=data.galaxyproject.org
CVMFS_HTTP_PROXY='DIRECT'
CVMFS_QUOTA_LIMIT=4096
CVMFS_USE_GEOAPI=yes
```

Site-local overrides layered on top of `default.conf` — this is where **which repos to mount** and proxy/quota settings actually live for this machine.

!!! note
    `CVMFS_HTTP_PROXY='DIRECT'` because this box is on Nectar, which has no Squid proxy set up — every client talks straight to the Stratum 1 servers. Compare with Nirin, which does have Squid proxies available and sets a proxy IP address so its hosts route through those instead of going direct.

#### `/etc/cvmfs/domain.d/galaxyproject.org.conf`

```bash
cat /etc/cvmfs/domain.d/galaxyproject.org.conf
```

```conf
CVMFS_SERVER_URL="http://cvmfs1-tacc0.galaxyproject.org/cvmfs/@fqrn@;http://cvmfs1-iu0.galaxyproject.org/cvmfs/@fqrn@;http://cvmfs1-psu0.galaxyproject.org/cvmfs/@fqrn@;http://galaxy.jrc.ec.europa.eu:8008/cvmfs/@fqrn@;http://cvmfs1-mel0.gvl.org.au/cvmfs/@fqrn@"
CVMFS_KEYS_DIR="/etc/cvmfs/keys/galaxyproject.org"
```

Domain-level config — anything ending in `.galaxyproject.org` inherits this: the Stratum 1 replica list and the key directory for signature verification. `@fqrn@` is substituted per-repo, so one config line covers every `*.galaxyproject.org` repo.

#### `/etc/cvmfs/keys/galaxyproject.org/`

```bash
ls /etc/cvmfs/keys/galaxyproject.org/
```

```text
data.galaxyproject.org.pub
singularity.galaxyproject.org.pub
```

Public keys used to verify the cryptographic signature on each repository's root catalog. Both keys are already present — adding the singularity repo back in doesn't require fetching a new key, just telling CVMFS to use it.

#### `/etc/auto.master.d/cvmfs.autofs`

```bash
cat /etc/auto.master.d/cvmfs.autofs
```

```text
/cvmfs /etc/auto.cvmfs
```

The autofs map entry that makes `/cvmfs` an automount point in the first place. `/etc/auto.cvmfs` is the CVMFS-provided automount helper — this is the piece that made `ls /cvmfs` come back empty in step 1: nothing is mounted until autofs is asked to resolve a path under it.

### 4. Add the singularity repo back in

```bash
sudo sed -i 's/CVMFS_REPOSITORIES=data.galaxyproject.org/CVMFS_REPOSITORIES=data.galaxyproject.org,singularity.galaxyproject.org/' /etc/cvmfs/default.local

cat /etc/cvmfs/default.local
```

If the line was correctly updated we can run:

```bash
cvmfs_config probe
```

We expect two repositories to be available now:

```text
Probing /cvmfs/data.galaxyproject.org... OK
Probing /cvmfs/singularity.galaxyproject.org... OK
```

`ls /cvmfs` now shows both repos mounted.

### 5. Explore the singularity repo

```bash
ls /cvmfs/singularity.galaxyproject.org/
```

```text
1  2  3  a  all  b  c  d  e  f  g  h  i  j  k  l  m  n  o  p  q  r  s  t  u  v  w  x  y  z
```

The single-character directories are a two-level alphabetical index meant to help you find tooling without listing everything at once. For example, `s/a/` holds symlinks for every tool starting `sa...`, pointing back into `all`:

```bash
ls -la /cvmfs/singularity.galaxyproject.org/s/a | head -6
```

```text
sabre:1.000--h577a1d6_6 -> ../../all/sabre:1.000--h577a1d6_6
sabre:1.000--h5bf99c6_2 -> ../../all/sabre:1.000--h5bf99c6_2
sabre:1.000--h7132678_3 -> ../../all/sabre:1.000--h7132678_3
```

It helps, but only if you already know the tool's first two letters — `s/` alone has 25 subdirectories and `s/a/` alone has 777 entries. In practice, `all` + `grep` is faster than walking the index by hand:

```bash
ls /cvmfs/singularity.galaxyproject.org/all | wc -l
```

```text
122777
```

**122,777 containers** in one directory — every Bioconda/Biocontainers image Galaxy has ever pulled in. Too many to browse by eye, so you search for what you want:

```bash
ls /cvmfs/singularity.galaxyproject.org/all | grep '^samtools:'
```

```text
samtools:0.1.12--0
samtools:0.1.12--1
samtools:0.1.13--0
...
samtools:1.9--h91753b0_8
```

138 versions of `samtools` alone. Naming convention is `<tool>:<version>--<build>`, matching the Bioconda/Biocontainers tag scheme — the same naming SHPC and Shelley key off of next.

---

## Part 2: SHPC (Singularity Registry HPC)

SHPC allows the installation of software containers in the form of "container modules", for transparent usage of containerised applications. An automated process generates a system module for an application, hiding the specificities of the Singularity syntax behind shell functions that take the same name as the corresponding executables.

### 1. Load the module

```bash
cd
module avail
module load shpc
module avail
```

The second `module avail` shows `singularity` got pulled in too — `shpc` depends on it, so loading `shpc` auto-loads `singularity` as well.

### 2. Where SHPC gets its registry from

```bash
cat /opt/shpc/singularity-hpc/shpc/settings.yml
```

```yaml
registry: [https://github.com/singularityhub/shpc-registry]
```

SHPC reads recipe entries (maintainer, description, available tags/aliases) from the community-maintained `shpc-registry` repo on GitHub. The actual container image still comes from wherever the recipe's `docker:` field points — `quay.io/biocontainers` in our case.

### 3. Search and inspect an entry

```bash
shpc show -f samtools
```

```text
ghcr.io/autamus/samtools
quay.io/biocontainers/bioconductor-rsamtools
quay.io/biocontainers/msamtools
quay.io/biocontainers/perl-bio-samtools
quay.io/biocontainers/samtools
```

`-f` fuzzy-matches on the name, so a few unrelated tools show up alongside the one we want: `quay.io/biocontainers/samtools`.

```bash
shpc show quay.io/biocontainers/samtools
```

```yaml
url: https://biocontainers.pro/tools/samtools
maintainer: '@vsoch'
description: shpc-registry automated BioContainers addition for samtools
latest:
  1.23.1--ha83d96e_0:
    sha256:23cda33a3a42125872766df9aaf1d2db67cdb8c85314b793465188435af31ba6
tags:
  1.10--h2e538c0_3:
    sha256:84a8d0c0acec87448a47cefa60c4f4a545887239fcd7984a58b48e7a6ac86390
  ...
  1.23.1--ha83d96e_0:
    sha256:23cda33a3a42125872766df9aaf1d2db67cdb8c85314b793465188435af31ba6
docker: quay.io/biocontainers/samtools
aliases:
  bgzip: /usr/local/bin/bgzip
  htsfile: /usr/local/bin/htsfile
  samtools: /usr/local/bin/samtools
  tabix: /usr/local/bin/tabix
  ...
```

Every tag is pinned by sha256, and `aliases` lists the binaries inside the container that SHPC will expose as individual commands.

### 4. Install pointing at the CVMFS copy, not a fresh download

By default `shpc install` pulls the image straight from the registry (`docker://...`). We already have this exact image in CVMFS, so point at it instead:

```bash
ls /cvmfs/singularity.galaxyproject.org/all/samtools*1.23.1--ha83d96e_0
```

```text
/cvmfs/singularity.galaxyproject.org/all/samtools:1.23.1--ha83d96e_0
```

```bash
shpc install quay.io/biocontainers/samtools:1.23.1--ha83d96e_0 /cvmfs/singularity.galaxyproject.org/all/samtools:1.23.1--ha83d96e_0 --keep-path
```

```text
Module quay.io/biocontainers/samtools:1.23.1--ha83d96e_0 was created.
```

The syntax names the tool twice — once as the registry identifier (for metadata/aliases), once as the literal CVMFS path (the actual image) — but this is what lets SHPC build a module against an already-mounted image instead of fetching a new copy.

### 5. What `install` creates

```bash
tree ~/shpc/
```

```text
shpc/
├── modules
│   └── quay.io/biocontainers/samtools/1.23.1--ha83d96e_0/module.lua
└── wrappers
    └── quay.io/biocontainers/samtools/1.23.1--ha83d96e_0/
        ├── 99-shpc.sh
        └── bin/  (bgzip, htsfile, samtools, samtools-run, samtools-shell, tabix, ...)
```

Two parallel trees, both keyed by `<registry>/<org>/<tool>/<version>`:

- **`modules/`** — the Lmod `module.lua` file that `module load` reads
- **`wrappers/`** — one shell script per alias, plus every real binary inside the container — these are what end up on `$PATH` once the module is loaded

#### The module file

```bash
cat ~/shpc/modules/quay.io/biocontainers/samtools/1.23.1--ha83d96e_0/module.lua
```

Here are the key parts of this module file:

- Container path (most critical)
- All tools are exposed as commands
- Environment variables users can control
- Wrapper directory location — needs to exist and be readable by all users for global access
- Conflict declarations — prevents loading multiple versions simultaneously

#### The wrapper file

```bash
cat ~/shpc/wrappers/quay.io/biocontainers/samtools/1.23.1--ha83d96e_0/bin/htsfile
```

```bash
#!/bin/bash

script=`realpath $0`
wrapperDir=`dirname $script`/..

singularity ${SINGULARITY_OPTS} exec ${SINGULARITY_COMMAND_OPTS} -B $wrapperDir/99-shpc.sh:/.singularity.d/env/99-shpc.sh   /cvmfs/singularity.galaxyproject.org/all/samtools:1.23.1--ha83d96e_0 /usr/local/bin/htsfile "$@"
```

Every alias script is a thin `singularity exec` wrapper, and the image path it runs against is the **CVMFS path**, not a locally cached copy. `htsfile`, `samtools`, `bgzip`, etc. all become ordinary commands on `$PATH` once the module is loaded, running straight out of CVMFS — no image download, no local disk usage for the container itself.

### 6. Make the module visible and inspect a wrapper

Once installed, the module is not immediately visible in the tool paths.

```bash
module avail
```

You need to add the module path to Lmod so that it is visible when available modules are inspected:

```bash
module use ~/shpc/modules/quay.io/biocontainers
module avail
```

Load the module and check it is the correct version:

```bash
module load samtools/1.23.1--ha83d96e_0/module
samtools --version
```

### 7. Installing a tag that isn't in the registry

CVMFS has far more versions mounted than the shpc-registry recipe knows about — `samtools:1.0--0` exists under `/cvmfs/singularity.galaxyproject.org/all/`, but predates the range of tags the registry maintainer added:

```bash
shpc install quay.io/biocontainers/samtools:1.0--0 /cvmfs/singularity.galaxyproject.org/all/samtools:1.0--0 --keep-path
```

```text
quay.io/biocontainers/samtools:1.0--0 is not a known identifier. Valid tags are:
1.10--h2e538c0_3
1.11--h6270b1f_0
...
1.23.1--ha83d96e_0
```

SHPC validates against the tags listed in the recipe's `container.yaml` — it won't take just any path. To install an untracked tag, add a **local registry** with a `container.yaml` that includes it:

```bash
# Seed a local registry from the upstream recipe, in the same nested layout shpc expects
mkdir -p shpc/registry/local/quay.io/biocontainers/samtools
curl -fsSL \
  https://raw.githubusercontent.com/singularityhub/shpc-registry/main/quay.io/biocontainers/samtools/container.yaml \
  -o shpc/registry/local/quay.io/biocontainers/samtools/container.yaml

# Register it with shpc — local registry takes priority, upstream stays as fallback
shpc config add registry shpc/registry/local/
shpc config get registry
```

```text
['shpc/registry/local/', 'https://github.com/singularityhub/shpc-registry']
```

Get the checksum for the untracked tag and add it to the local recipe:

```bash
sha256sum /cvmfs/singularity.galaxyproject.org/all/samtools:1.0--0
```

```text
be88ae82e3ba7c650be087fb29bd80505982d092fb69f0913c17892fd7422cb9  /cvmfs/singularity.galaxyproject.org/all/samtools:1.0--0
```

Edit `shpc/registry/local/quay.io/biocontainers/samtools/container.yaml` and add the new tag under `tags:`:

```yaml
  1.0--0:
    sha256:be88ae82e3ba7c650be087fb29bd80505982d092fb69f0913c17892fd7422cb9
```

Install again:

```bash
shpc install quay.io/biocontainers/samtools:1.0--0 /cvmfs/singularity.galaxyproject.org/all/samtools:1.0--0 --keep-path
```

```text
Module quay.io/biocontainers/samtools:1.0--0 was created.
```

!!! tip
    The recipe is really just metadata plus a checksum allowlist — as long as a tag/sha256 is declared somewhere in a registry shpc knows about, it will build a module against any image, including ones the upstream shpc-registry never heard of. This is exactly how you'd onboard a locally-built or CVMFS-only container.

### 8. Load it and use it

```bash
module load samtools/1.0--0/module
```

```text
The following have been reloaded with a version change:
  1) samtools/1.23.1--ha83d96e_0/module => samtools/1.0--0/module
```

Swaps straight over the module loaded earlier — Lmod handles the version switch, no need to unload first.

```bash
samtools --version
```

```text
samtools 1.0
Using htslib 1.0
Copyright (C) 2014 Genome Research Ltd.
```

That's the exact ancient container we hand-registered a minute ago, invoked as an ordinary command, sourced live from CVMFS with nothing downloaded to local disk.

---

## Part 3: Shelley

Shelley automates everything we just did by hand in Part 2 §7. It's a CVMFS/SHPC-aware CLI (with a TUI mode) that turns *find the container → check the registry → patch a local recipe → install* into a single `shelley build` call.

### 1. Interactive mode

```bash
shelley interactive
```

Opens a full-screen TUI with its own command grammar:

```text
Available Commands
Command                Description                                     Example
find <tool> [-v|-vv]   Find a tool; -v all versions, -vv adds paths    find fastqc
search <terms>         Search for tools by description                 search quality control
build <tool[/ver]>     Build an Lmod module for a tool                  build samtools/1.21
help                   Show this help table                            help
exit                   Exit interactive mode
```

`exit` (or Ctrl-C) drops back to the shell.

### 2. Look up a tool

```bash
shelley find samtools
```

Renders a description panel (homepage, operations, inputs/outputs — pulled from the same shpc-registry metadata), then a version table:

```text
Available Versions
Versions    Buildable  Installed
1.23.1      ✓          ✗
1.23        ✓          ✗
1.22.1      ✓          ✗
1.22        ✓          ✗
1.21        ✓          ✓
+ 35 more              shelley find samtools -v
```

```text
Buildable ✗: Versions not in the shpc registry may still be built, but can take a few minutes longer.
Installed ✓: This version is already available to module load on this system.
```

Shelley is surfacing exactly the two facts that took a full manual walkthrough to establish in Part 2: whether a version has a registry recipe already, and whether it's already built as a module here. `1.21` shows **Installed ✓** — that's the module we built by hand earlier.

### 3. See every CVMFS build, and where they're located on CVMFS

```bash
shelley find samtools -vv
```

`-vv` adds the CVMFS path per tag and switches to a paginated table — **136 total** builds available for samtools (vs. the ~19 the shpc-registry recipe knows about):

```text
Available Builds (136 total) — page 1 of 14
Versions             Buildable  Installed  Container Path
1.23.1--ha83d96e_0   ✓          ✗          /cvmfs/singularity.galaxyproject.org/all/samtools:1.23.1--ha83d96e_0
1.21--h50ea8bc_0     ✓          ✓          /cvmfs/singularity.galaxyproject.org/all/samtools:1.21--h50ea8bc_0
1.21--h96c455f_1     ✓          ✓          /cvmfs/singularity.galaxyproject.org/all/samtools:1.21--h96c455f_1
...
1.5--0               ✗          ✗          /cvmfs/singularity.galaxyproject.org/all/samtools:1.5--0
```

Every row is a build Shelley found by scanning `/cvmfs/singularity.galaxyproject.org/all` directly — the same directory (and the same "too many to browse" problem) from Part 1 §5, now with a paged, filterable view instead of raw `ls | grep`.

### 4. Build a version that isn't in the registry — one command

```bash
shelley build samtools:1.5--2
```

Under the hood this runs the same manual sequence from Part 2 §7 — diffing container layers, extracting the filesystem, subtracting out common base-image layers (`alpine`, `busybox`, `ubuntu`, miniconda, etc. — "removed N shared paths") to isolate what's unique to this container — but as one command:

```text
Generating diff for /cvmfs/singularity.galaxyproject.org/all/samtools:1.5--2
...
⚠️  quay.io/biocontainers/samtools:1.5--2 is not in the upstream shpc-registry. A local entry has been created in /apps/local.

✅ Module Built Successfully!
Tool: samtools
Version: 1.5--2
Module Path: /apps/Modules/modulefiles/samtools/1.5--2.lua

To load this module:
module load samtools/1.5--2
```

!!! tip
    Compare this to Part 2 §7: create a directory, `curl` the upstream recipe, register the local registry, `sha256sum` the image, hand-edit the YAML, then `shpc install`. Shelley collapses all of that into one command — it detects the tag is untracked, generates the local registry entry itself (in `/apps/local`, the site-wide equivalent of the `~/shpc/registry/local/` we built by hand), computes the checksum, and builds the module.

One more difference worth noting: this module lands at `/apps/Modules/modulefiles/samtools/1.5--2.lua` — the **site-wide** Environment Modules tree (same place `R`, `nextflow`, `snakemake`, etc. already live) — not the personal `~/shpc/modules/` tree `shpc install` wrote to in Part 2. Shelley builds modules for the whole system to use, not just the current user.

### 5. Load it

```bash
module avail
```

```text
--- /apps/Modules/modulefiles ---
   R/4.3.3   ansible/2.16.3   jupyter/2026.07   nextflow/26.04.4   nf-core/4.0.2   plink/1.90b7.7--h7b50bb2_0   rstudio/2026.06.0   samtools/1.5--2   samtools/1.21--h96c455f_1   snakemake/7.32.4
```

`samtools/1.5--2` now sits alongside the site's existing modules.

```bash
module load samtools/1.5--2
```

```text
The following have been reloaded with a version change:
  1) samtools/1.0--0/module => samtools/1.5--2
```

End to end: a container version nobody had registered anywhere, sitting in CVMFS since who-knows-when, built into a real system module and loaded — with one command instead of many.
