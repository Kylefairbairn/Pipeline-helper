# Parallel Jenkins NetBoot repository

A parent job reads `config/boards.yaml`, merges defaults with each board, and starts
concurrent builds of the same single-board job. Both jobs load their Jenkinsfiles
from Git branch `master`. Board builds get separate Jenkins workspaces.

## Files

| File | Purpose |
| --- | --- |
| `Jenkinsfile` | Parent: validate YAML, release checkout executor, fan out builds |
| `config/boards.yaml` | Shared defaults and per-board overrides |
| `child-pipeline/Jenkinsfile` | Concurrent single-board deployments, per-board lock |
| `scripts/validate_params.py` | Validate shell-bound parameter values |
| `scripts/deploy-board.sh` | Cache check, download, publish, prepare, install, reboot, verify |
| `scripts/deploy_board.py` | Python entry point delegating to the Bash backend |
| `scripts/render_uenv.py` | Render deployment variables while preserving U-Boot variables |
| `templates/uEnv.txt.template` | AM625 TFTP + NFS example; adapt to actual U-Boot |
| `server/common.sh` | Shared server validation |
| `server/publish-wic-release.sh` | flock-protected, atomic WIC extraction |
| `server/prepare-board-rootfs.sh` | Separate writable root for each board and release |
| `server/install-tools.sh` | Install deployment tools on boot server |
| `server/exports.example` | Restricted lab NFS export template |
| `docs/setup.md` | Jenkins, server, board and troubleshooting instructions |
| `.gitignore` | Ignore generated deployment files |

## Start here

1. Read `docs/setup.md` and configure your boot server and Yocto image.
2. Update addresses, versions and boards in `config/boards.yaml`.
3. Update Artifactory, SSH user and boot server in `child-pipeline/Jenkinsfile`.
4. Create parent and child Jenkins jobs against the same repository and `*/master`.
5. Run one board first, then add the remaining boards to YAML.

This is a complete starter repository, not a deployment validated on your hardware.
The U-Boot import flow, console, interface, kernel NFS support, Yocto network/fstab
configuration, and SSH provisioning must match your board. This repo downloads raw
`.wic` files, not `.wic.gz` or `.wic.zst`.

`SCRIPT_IMPL` is retained for compatibility, but both options use one Bash backend.
The Python selection is an entry point, not a second independent implementation.
Python 3 is required on Jenkins agents in either mode; no third-party Python packages.

Health checks confirm a new boot ID, assignment markers, and an NFS root mount.
They do not prove application health or that the exact kernel image was booted.

## Concurrency and storage

Keep `disableConcurrentBuilds()` in the parent only. Lockable Resources locks each
board for the whole deployment. Server `flock` rechecks and publishes shared releases
under a per-release lock. Parallel cache misses may download the same WIC more than
once, but cannot overwrite a completed release. The version checksum is immutable
when publishing; cache-hit builds trust the existing cache.

Writable roots use `boards/BOARD_ID/roots/PRODUCT/TYPE/VERSION`. Retrying a version
preserves its root and local modifications; a fresh-state test requires a new version
or explicit offline cleanup. Roots are never automatically deleted. Different boards
never share a writable root. All deployments targeting a given board must use the same
Jenkins lock name/controller; external callers are not covered by that Jenkins lock.

Each version root has its own freshly generated machine identity on first boot.
SSH host keys are preserved from the Yocto rootfs: provision per-board keys explicitly
and maintain known_hosts when switching versions. This example never disables host
key checking and does not automatically trust fetched host keys.

## Reference documentation

- https://www.jenkins.io/doc/pipeline/steps/pipeline-build-step/
- https://www.jenkins.io/doc/pipeline/steps/pipeline-utility-steps/
- https://www.jenkins.io/doc/pipeline/steps/lockable-resources/
- https://www.jenkins.io/doc/book/pipeline/syntax/
