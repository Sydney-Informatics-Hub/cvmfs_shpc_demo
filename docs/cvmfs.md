# CVMFS

**CVMFS** mounts a shared, read-only library of scientific software/containers over HTTP — nothing is downloaded until you touch it.

## 1. The mountpoint before anything is mounted

```bash
cd /cvmfs
ls
```

Nothing is listed. `/cvmfs` is managed by **autofs** — repositories only appear the moment something tries to access them. An empty `ls /cvmfs` is expected: nothing is "mounted" until it's touched.

## 2. Probe the client

```bash
cvmfs_config probe
```

```text
Probing /cvmfs/data.galaxyproject.org... OK
```

This attempts to mount and stat every repository listed in `CVMFS_REPOSITORIES` (`/etc/cvmfs/default.local`) and reports OK/FAIL per repo. Only `data.galaxyproject.org` is configured right now — the **singularity** repo (`singularity.galaxyproject.org`) has deliberately been left out so we can add it back in live.

## 3. The configuration files

CVMFS reads config in layers. Walk through each file in place:

### `/etc/cvmfs/default.conf`

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

### `/etc/cvmfs/default.local`

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

### `/etc/cvmfs/domain.d/galaxyproject.org.conf`

```bash
cat /etc/cvmfs/domain.d/galaxyproject.org.conf
```

```conf
CVMFS_SERVER_URL="http://cvmfs1-tacc0.galaxyproject.org/cvmfs/@fqrn@;http://cvmfs1-iu0.galaxyproject.org/cvmfs/@fqrn@;http://cvmfs1-psu0.galaxyproject.org/cvmfs/@fqrn@;http://galaxy.jrc.ec.europa.eu:8008/cvmfs/@fqrn@;http://cvmfs1-mel0.gvl.org.au/cvmfs/@fqrn@"
CVMFS_KEYS_DIR="/etc/cvmfs/keys/galaxyproject.org"
```

Domain-level config — anything ending in `.galaxyproject.org` inherits this: the Stratum 1 replica list and the key directory for signature verification. `@fqrn@` is substituted per-repo, so one config line covers every `*.galaxyproject.org` repo.

### `/etc/cvmfs/keys/galaxyproject.org/`

```bash
ls /etc/cvmfs/keys/galaxyproject.org/
```

```text
data.galaxyproject.org.pub
singularity.galaxyproject.org.pub
```

Public keys used to verify the cryptographic signature on each repository's root catalog. Both keys are already present — adding the singularity repo back in doesn't require fetching a new key, just telling CVMFS to use it.

### `/etc/auto.master.d/cvmfs.autofs`

```bash
cat /etc/auto.master.d/cvmfs.autofs
```

```text
/cvmfs /etc/auto.cvmfs
```

The autofs map entry that makes `/cvmfs` an automount point in the first place. `/etc/auto.cvmfs` is the CVMFS-provided automount helper — this is the piece that made `ls /cvmfs` come back empty in step 1: nothing is mounted until autofs is asked to resolve a path under it.

## 4. Add the singularity repo back in

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

## 5. Explore the singularity repo

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

**Next: [SHPC](shpc.md)**
