# vr-sandbox

Containerised toolchain for offensive-security / vulnerability research.
General-purpose, batteries-included — binary RE, web pentesting, crypto
analysis, cloud-side recon, forensic carving, exploit dev. CTF challenges
were the original consumer, but nothing in the image is CTF-specific.

## Use it

Pre-built multi-arch image (amd64 + arm64), ~8 GB compressed:

```bash
docker pull nnamon/vr-sandbox:latest
docker tag nnamon/vr-sandbox:latest vr-sandbox
docker run --rm -it -v "$(pwd):/work" -w /work vr-sandbox bash
```

Consumer applications typically bind their work tree at `/work` or
`/challenge` and `chdir` there — the image makes no assumption about
which mountpoint you use.

## Companion sidecar: vr-vpn

`Dockerfile.vpn` builds a ~12 MB alpine + openvpn sidecar (`nnamon/vr-vpn`).
Pair it with `vr-sandbox` when the target sits behind a VPN tunnel
(HTB Machines, corporate lab access, etc.):

```bash
docker run -d --name vpn --cap-add NET_ADMIN --device /dev/net/tun \
  -v ./client.ovpn:/vpn.ovpn:ro nnamon/vr-vpn openvpn --config /vpn.ovpn
docker run --rm -it --network container:vpn -v "$(pwd):/work" -w /work vr-sandbox bash
```

## Build locally

Only needed if you've modified the Dockerfile — the pre-built image is
the same content. Local build is ~30 min on Apple Silicon, longer on
intel under emulation.

```bash
./build.sh                 # full build + smoke
./build.sh --no-build      # smoke an existing image
./build.sh --no-test       # build only
./build.sh my-tag          # custom local tag
```

`smoke.sh` runs a per-category presence check inside a throwaway
container; bind-mounted in, never lands in the runtime image.

## What's inside

See `sandbox-tools.txt` for the full inventory by category. High-level:

- **Binary**: gdb (pwndbg/pef), radare2, ghidra-headless, ilspycmd, CFR, Fernflower, Oxidizer, Capstone, LIEF, frida, qemu-user (per-arch), wine
- **Web**: sqlmap, ffuf, wpscan, nuclei, gobuster, hashcat, john, mitmproxy, Playwright (Chromium + Firefox)
- **Crypto**: SageMath 10 (conda-forge), z3, fpylll, pycryptodome, RsaCtfTool
- **Cloud**: gcloud, az, kubectl, pacu
- **Forensics**: volatility3, binwalk, foremost, exiftool, sleuthkit, photorec
- **Wireless / Radio**: aircrack-ng, kismet, multimon-ng
- **Multi-glibc**: Ubuntu 16.04/18.04/22.04 sysroots so pwn distfiles
  with bundled `libc.so.6` from any of those release lines run
  natively under the matching loader

The image is opinionated about what's pre-installed but doesn't get in
the way of `apt install` / `pip install` for one-off additions
mid-session.

## CI

`.github/workflows/build.yml` builds both `Dockerfile` and `Dockerfile.vpn`
on every push to main, on every PR (validate-only), and on
`workflow_dispatch`. Per-arch matrix on native runners (`ubuntu-latest`
for amd64, `ubuntu-24.04-arm` for arm64) — no QEMU emulation, ~30 min
per lane in parallel. Smoke-tests each arch separately, then merges
into a multi-arch manifest tagged `:latest` + `:<git-sha>`.

## Provenance

Forked out of [nnamon/ctf-agent](https://github.com/nnamon/ctf-agent)
where the image was originally developed. Now a standalone repo so the
sandbox is reusable by other consumers without dragging in the agent
runtime, and so the public-repo Actions allowance covers the amd64
build lane (the agent repo is private; the sandbox repo is public).
