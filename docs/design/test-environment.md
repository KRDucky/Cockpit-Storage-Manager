# Test environment: disposable VMs and loopback fixtures

Status: M0 closeout. Defines *how* every integration test in this repository is run.
Implemented incrementally from M1 (harness skeleton + guard) and M2 (first VM scenarios).

## 1. Tiers
| Tier | Where | Privilege | Touches block devices | Runs in CI |
|---|---|---|---|---|
| T0 unit / property | any dev machine, CI container | none | no | every push |
| T1 component UI | browser test runner against mock transport | none | no | every push |
| T2 container contract | Debian 13 / Ubuntu 26.04 / 26.10 containers: package build, install, helper read-only calls against **recorded fixtures** | none | no | every push |
| T3 disposable VM | QEMU/KVM guest built from distro cloud images, one per Tier-1/Tier-2 target | root **inside the guest only** | loop devices and virtual disks inside the guest | nightly / on demand; never on a developer's host |
| T4 fault injection | T3 plus QMP hot-unplug, dm-error/dm-flakey, power-off | root inside the guest | yes | before M8/M9 release gates |

No test tier above T2 may run on a machine that is not a disposable VM.

## 2. Disposable-host interlock (enforced by the harness, not by convention)
A T3/T4 test refuses to start unless **all** hold inside the target:
1. `/etc/cockpit-storage-manager-test-host` exists, contains the VM's machine-id, and was
   written by the image builder (not by the test).
2. `systemd-detect-virt --vm` reports a VM.
3. Hostname matches `^csm-test-[a-z0-9-]+$`.
4. Every device the test will touch is either a loop device whose backing file is under
   `/var/tmp/csm-test/` or a virtio disk whose serial starts with `CSMTEST`.
5. No device it will touch holds `/`, `/boot`, `/boot/efi`, swap, or any mount outside
   `/srv/csm-test/`.
Failure of any check aborts with a non-zero exit before any command runs. The interlock has its
own unit tests (T0) including spoofing attempts (symlinked marker file, wrong machine-id).

## 3. VM image and lifecycle
- Base: official Debian 13 and Ubuntu 26.04/26.10 cloud images; cloud-init installs the target
  versions of cockpit, mergerfs, snapraid, samba, nfs-kernel-server, udisks2, smartmontools.
- Built by a script in `integration/` (M2) into a qcow2 overlay; each test run uses a fresh
  overlay and discards it. Package versions inside the guest are recorded in the test report.
- Extra virtio disks with serials `CSMTEST-<n>` are attached for scenarios needing realistic
  hot-unplug; loop devices (`losetup --find --show` on sparse files) for everything else.
- The host only needs QEMU/KVM; nothing on the host is mounted, formatted or modified.

## 4. Fault-injection techniques
| Fault | Technique |
|---|---|
| Surprise disk removal | QMP `device_del` on a virtio disk |
| Device returns I/O errors | swap a dm table to `error` target (`dmsetup load/resume`) |
| Intermittent errors | `dm-flakey` |
| Delayed mount | systemd drop-in delaying the mount unit; or attach disk after boot |
| Wrong-device remount | mount a different loop filesystem at a branch path |
| Duplicate UUID | `dd` a filesystem image onto a second loop device |
| `/dev/sdX` reorder | detach and re-attach loop devices / virtio disks in a different order |
| Power loss | QMP `system_reset` / `quit` at an injected phase boundary (harness hook) |
| Concurrent admin edit | second process edits a config file between plan and execute |
| Concurrent writers | Samba client (`smbclient`), NFS client (loopback mount), local writer, direct-to-branch writer |
| Huge / hostile output | stub executables on PATH **inside an isolated test root only** (exec allowlist points at them via test configuration) |

## 5. Evidence and reporting
Each T3/T4 run stores: guest package versions, command transcript from `exec.py` (redacted),
job event journals, before/after file manifests (path, size, mtime, sha256; atime excluded),
`lsblk`/`findmnt` snapshots, journald excerpts. Test results quoted in milestone reports must
come from these artifacts.

## 6. What is blocked today
The M0 research workspace has no KVM access and no Cockpit/mergerfs/SnapRAID installed; T3/T4
cannot run there. Whether CI runners provide nested virtualization is an M1 question; if not,
T3/T4 run on an owner-provided disposable VM host and results are attached to the PR.
