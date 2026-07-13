# CVMFS → SHPC → Shelley Demo

A live, follow-along tour of three tools that stack together:

- **[CVMFS](cvmfs.md)** mounts a shared, read-only library of scientific software/containers over HTTP — nothing is downloaded until you touch it.
- **[SHPC](shpc.md)** (Singularity Registry HPC) turns any container image into a normal `module load`-able command, including images already sitting in CVMFS.
- **[Shelley](shelley.md)** wraps SHPC into a single guided command, so building a module from a CVMFS container is one step instead of many.

Work through the pages in order — each one builds on the container mounted and module built in the previous page.

!!! note "Introductory slides (internal, USYD staff only)"
    [Slides](https://unisyd.sharepoint.com/:p:/t/SydneyInformaticsHub2/IQCSfH_qzK3iSLoXlMAUGBbyAYggeHNUwEnAS-mGAfDhe5g?e=ZbJAca) — access is restricted to specific people; if you weren't shared on it directly, this link won't open.

## Log into your BioShell VM

This demo runs on a **BioShell** VM. Before starting, connect over SSH:

```bash
ssh <username>@<ip_address>
```

!!! note
    Replace `<username>` and `<ip_address>` with the credentials provided for your BioShell session.

---

[![BioShell](assets/BioShell_logo_with_name.png){ width="200" }](https://sih.tools/bioshell-guide)
