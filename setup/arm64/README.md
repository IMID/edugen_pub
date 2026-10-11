# omics-genomika — arm64 Docker image (kernel fix)

This directory contains a native **arm64** rebuild of the `omics-genomika`
Jupyter image used for the OMICS genomics course, built from the upstream
Dockerfile at
[`IMID/edugen_pub/setup/Dockerfile`](https://github.com/IMID/edugen_pub/blob/main/setup/Dockerfile).

It exists because the published image, `matdawidziuk/omics-genomika:1.1.0`,
does not work on Apple Silicon Macs — the Jupyter kernel never starts.

## The problem

Running the course's documented command:

```bash
docker run --rm -it --platform linux/amd64 \
  -p 127.0.0.1:8888:8888 \
  -v "$HOME/omics-genomika:/home/jovyan/work" \
  matdawidziuk/omics-genomika:1.1.0 \
  jupyter-lab --ip='*' --NotebookApp.token='' --NotebookApp.password='' \
  --notebook-dir=/home/jovyan/work
```

the JupyterLab server starts fine and lists the `omics-genomika` kernel, but
opening any notebook hangs forever trying to start that kernel. The
underlying error (visible with the Jupyter client directly) is:

```
RuntimeError: Kernel didn't respond in 120 seconds
```

## Root cause

1. **Architecture mismatch.** `matdawidziuk/omics-genomika:1.1.0` is built
   `linux/amd64` only. On an Apple Silicon Mac, Docker Desktop runs it
   through Rosetta emulation inside the Linux VM.
2. **That alone isn't fatal** — the stock `python3` kernel from the same
   image starts fine under the same emulation. The failure is specific to
   the `omics-genomika` kernel's own conda venv
   (`/home/jovyan/venv/omics-genomika`, Python 3.9).
3. **The real issue:** that venv's `pip install` (in the original
   Dockerfile) pulls in `igv_jupyterlab==1.0.1` and `gsutil==5.30`, both of
   which drag in a full second copy of the Jupyter *server* stack
   (`jupyterlab 3.6.8`, `notebook 6.5.7`, `jupyter_client 7.4.9`) alongside
   `ipykernel 6.29.5` inside the kernel's own venv. The kernel process binds
   all its ZMQ sockets and prints its banner, but never answers the
   `kernel_info` handshake — it just sits there. `pip check` reports no
   broken requirements, so this isn't a declared conflict; it's two
   coinstalled Jupyter stacks interfering with the kernel's event loop.
   Upgrading `jupyter_client` alone did not fix it, confirming the problem
   isn't one single pinned version.
4. As a secondary issue, `igv_jupyterlab` in the published 1.1.0 image isn't
   even importable (`ModuleNotFoundError`), so the published image had
   already drifted from what the Dockerfile describes.

## The fix

[`Dockerfile`](./Dockerfile) in this directory is the upstream Dockerfile
rebuilt natively for `linux/arm64`, with these changes:

| Change | Why |
|---|---|
| `bwa` and `vt` installed from Ubuntu's arm64 apt repo instead of compiled from source | Both use x86 SSE2 intrinsics in their Makefiles and do not compile on ARM |
| `google-cloud-sdk` → `google-cloud-cli` | Google renamed the apt package; the old name has no installation candidate anymore |
| `apt-key` → `gpg --dearmor` + explicit `[arch=...]` repo line | `apt-key` is deprecated; explicit arch avoids pulling the wrong package list |
| `HOME=/home/jovyan` (was `/tmp/jovyan` in the original) | The original cloned the course repo into `/tmp`, which doesn't survive container restarts and is shadowed by the `-v $HOME/omics-genomika:/home/jovyan/work` bind mount anyway. Course repo is also copied to `/opt/edugen_pub` as a backup. |
| Kernel venv installs **only** `ipykernel` + the scientific libraries (`numpy`, `pandas`, `pysam`, `matplotlib`, `biothings_client`, `ipywidgets`, `igv-notebook`) | Keeps the kernel's own venv free of any second Jupyter server stack |
| `igv_jupyterlab==1.0.1` installed with `--no-deps` | Gets the widget without re-pulling jupyterlab/notebook as dependencies |
| `gsutil` provided only via the `google-cloud-cli` apt package (CLI), not pip-installed into the kernel venv | The notebooks call `gsutil` as a shell command via `subprocess`, not as a Python import — no need for it inside the kernel venv |
| Dockerfile has a hard verification step (`python -c "import numpy,pandas,pysam,..."`) at the end of the kernel-venv build | Build now fails loudly if the kernel environment is ever broken again, instead of silently shipping a broken image |

## Building

```bash
cd ~/omics-genomika/docker
docker build --platform linux/arm64 -t omics-genomika:1.1.0-arm64 -f Dockerfile .
```

Build time is roughly 5–8 minutes (htslib/samtools/bcftools/bedtools are
compiled from source; everything else is apt/pip/prebuilt).

## Running

Same as before, minus `--platform linux/amd64` and pointing at the local tag:

```bash
docker run --rm -it \
  -p 127.0.0.1:8888:8888 \
  -v "$HOME/omics-genomika:/home/jovyan/work" \
  omics-genomika:1.1.0-arm64 \
  jupyter-lab --ip='*' --NotebookApp.token='' --NotebookApp.password='' \
  --notebook-dir=/home/jovyan/work
```

Then open http://127.0.0.1:8888/lab — the `omics-genomika` kernel now starts
and runs normally.

## What was verified

- JupyterLab server starts, both kernels (`omics-genomika`, `python3`) are
  listed.
- The `omics-genomika` kernel starts and executes code
  (`import numpy, pandas, pysam, matplotlib, igv_notebook, igv_jupyterlab`
  all succeed; `2+2` evaluates and returns correctly over the wire).
- The kernel venv (`pip list`) contains no leftover `jupyterlab`/`notebook`
  server packages — only `jupyter_client` (a direct dependency of
  `ipykernel` itself, not a server install).
- All CLI tools used by the course notebooks run correctly inside the
  container: `bwa`, `samtools 1.21`, `bcftools 1.21`, `bedtools 2.31.1`,
  `fastqc`, `gatk 4.6.0.0`, `gsutil 5.37`, `tabix`, `vt`, `multiqc`.
- `$HOME/omics-genomika` (notebooks and the `git/edugen_pub` checkout) is
  visible and writable at `/home/jovyan/work` inside the container.

## Known differences from the published 1.1.0 image

- This image is `linux/arm64` only. It won't run (without emulation) on an
  Intel/amd64 machine — rebuild with `--platform linux/amd64` on such a
  machine if needed; the underlying venv fix applies to either architecture.
- `igv_jupyterlab` now actually imports (it didn't in the published image).
- `gsutil` is a CLI tool (via `google-cloud-cli`) but not a Python package
  inside the kernel venv — fine for the notebooks, which only invoke it as a
  shell command.

## If you want to upstream this

The fixes here (native-arm64 package choices for `bwa`/`vt`, the renamed
`google-cloud-cli` package, and — most importantly — keeping the kernel venv
free of a second Jupyter server stack) are all applicable to the upstream
Dockerfile too, independent of architecture. Worth opening a PR against
[`IMID/edugen_pub`](https://github.com/IMID/edugen_pub) so the next person
doesn't hit the same multi-hour kernel hang.
