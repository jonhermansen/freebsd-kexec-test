# freebsd-kexec-test

A throwaway experiment spun off from `nixos-symbiote`.

## The question

`nixos-symbiote` pivots a running GitHub Actions runner from Ubuntu into **NixOS**
via `systemctl soft-reboot` — which keeps the **running kernel** and swaps only
userland. The agent survives because the kernel (which holds every process, fd,
and socket) never goes away.

**FreeBSD is a different kernel.** Reaching it from Linux requires replacing the
kernel itself (`kexec`/`kboot`) — a full kernel reboot. The open question:

> Can a GitHub Actions **job** survive a kernel reboot (`kexec`) at all?

A `kexec` into FreeBSD and a `kexec` into the *same* Linux kernel are identical in
the way that matters: both tear down every process, the runner's live TLS/TCP
connection to GitHub, the controlling TTY/session, and all kernel-resident state.
So we test the cheap, decisive version first — **`kexec` Linux → same Linux** —
with no FreeBSD and no `kboot`. If the job can't survive *that*, FreeBSD is moot.

## What dies in a kexec (and would need to be reconstructed "ourselves")

- Process table / the `Runner.Worker` executing the job (in-memory job state).
- The live TLS/TCP connection to GitHub (kernel-resident; peer sees a teardown).
- The job's OAuth token / session / lock lease (in-memory — likely not on disk).
- The controlling TTY and process session of the job's step tree.

GitHub has **no** "resume this job on a rebooted runner" path, so any continuity
would have to be reimplemented: re-auth from on-disk runner credentials, re-acquire
and renew the *same* job's server-side lock before GitHub abandons it, and resume
step reporting. The make-or-break unknown is whether the job token/lock is
recoverable from disk at all — this probe is how we find out.

## The probe

`.github/workflows/kexec-probe.yaml` records the boot id, fires `kexec -e`, and a
following step checks whether it ever runs again (and whether `boot_id` changed).
Watch the run from the API side to see what GitHub does with the job when the
runner's kernel reboots mid-step.
