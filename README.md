# OS-Sanitizer reproduction and extensions

Course project for **Computer System Security** (M.Tech, Semester 3) by Pradeep Kumar Mahato and Vivek Kumar.

We reproduce and extend **OS-Sanitizer**, a system-wide sanitizer that uses eBPF to find latent defects in
Linux applications:

> A. Crump et al., "OS-Sanitizer: System-wide Latent Defect Inference in Linux Applications",
> USENIX Security 2026. Copy of the paper: [os-sanitizer/Document/OS-Sanitizer.pdf](os-sanitizer/Document/OS-Sanitizer.pdf)

The repository holds two Rust projects:

| Directory | What it is | Upstream docs |
|---|---|---|
| [os-sanitizer/](os-sanitizer/) | The sanitizer: eBPF programs, the user-space loader and reporter, a symbolizer, and our added JSON output, clustering, scoring and benchmarks | [os-sanitizer/README.md](os-sanitizer/README.md) |
| [bpf-linker/](bpf-linker/) | The BPF static linker (upstream [aya-rs/bpf-linker](https://github.com/aya-rs/bpf-linker) v0.11.1, unmodified). It links os-sanitizer's eBPF crate into a BPF object file. | [bpf-linker/README.md](bpf-linker/README.md), [bpf-linker/BUILDING.md](bpf-linker/BUILDING.md) |

## How the two fit together

```
os-sanitizer-ebpf  (Rust, target bpfel-unknown-none)
        │  rustc emits LLVM bitcode
        ▼
bpf-linker  ── links the bitcode, runs the BPF optimisations, emits BTF (-C link-arg=--btf)
        │
        ▼
BPF object file ── embedded into the user-space binary by aya-build (os-sanitizer/build.rs)
        │
        ▼
os-sanitizer  (user space, aya) ── loads the uprobes, fentry, tracepoint and LSM programs
                                   and turns kernel events into reports
```

You don't call bpf-linker directly. Rust's `bpfel-unknown-none` target uses it as its linker, so it only
needs to be on your `PATH` when you run `cargo build` for os-sanitizer.

## Requirements

- Linux with BTF (`/sys/kernel/btf/vmlinux`) and the **BPF LSM** enabled (`bpf` must appear in
  `/sys/kernel/security/lsm`; add `lsm=...,bpf` to the kernel command line if it doesn't). Two of the
  eBPF programs are LSM hooks.
- Root to load the programs.
- Rust **nightly** (via [rustup](https://rustup.rs/)) with the `rust-src` component.
- clang / libclang, `bpftool`, and `aya-tool` (`cargo install bindgen-cli && cargo install --git https://github.com/aya-rs/aya aya-tool`).

Our testbed was a Fedora 39 VM (kernel 6.11.9-100.fc39, 10 vCPUs, 22 GiB RAM) with Rust nightly 1.102,
clang 17.0.6 and bpftool 7.5.

## Build

### 1. Install bpf-linker

The easiest way is a prebuilt binary (`cargo binstall bpf-linker`), as described in the
[bpf-linker README](bpf-linker/README.md#installation).

To build the copy in this repository instead, its LLVM major version must match the one in your nightly
compiler:

```bash
rustc +nightly -vV | grep LLVM            # e.g. "LLVM version: 22.1.x" → use llvm-22
cargo install --path bpf-linker --no-default-features --features llvm-22
```

This needs a matching libLLVM on the system. [bpf-linker/BUILDING.md](bpf-linker/BUILDING.md) explains how
to get one, including prebuilt LLVM archives and a Bazel build that compiles LLVM itself.

### 2. Generate kernel bindings

`os-sanitizer-ebpf/src/binding.rs` is **not in the repository** (it is git-ignored), so this step is
required before the first build. Kernel struct layouts differ between kernel versions, so generate the
bindings on the machine you will run on:

```bash
cd os-sanitizer
aya-tool generate task_struct dentry > os-sanitizer-ebpf/src/binding.rs
```

### 3. Build os-sanitizer

```bash
cargo build --release
```

This produces `target/release/os-sanitizer` and `target/release/os-sanitizer-symbolizer`. The eBPF crate
is built automatically by `os-sanitizer/build.rs`.

## Run

```bash
sudo env RUST_LOG=info ./target/release/os-sanitizer --reference-policy --json reports.jsonl
```

| Option | Effect |
|---|---|
| `--access`, `--gets`, `--rwx-mem`, `--memcpy`, `--strcpy`, `--strncpy`, `--sprintf`, `--snprintf`, `--printf-mutability`, `--system-mutability`, `--system-absolute`, `--filep-unlocked`, `--fixed-mmap`, `--interceptable-path`, `--security-file-open` | Turn on one pass. Each pass is described in [os-sanitizer/examples/readme.md](os-sanitizer/examples/readme.md). |
| `--toctou-2005`, `--leaky-vessel` | Extra passes, not included in `--all` |
| `--reference-policy` | The paper's recommended set: every pass except `memcpy`, `strcpy`, `strncpy` and `leaky-vessel` |
| `--all` | Every pass except `leaky-vessel` and `toctou-2005` |
| `-v, --visibility-depth N` | Stack frames per text report (default 7, `0` = full stack) |
| `--json FILE` | **Added by us.** Also append every report to `FILE` as one JSON object per line (see below) |

Reports go to stderr through `env_logger`, so set `RUST_LOG` (`info` shows INFO, WARN and ERROR).
To get source locations from a finite log, run `os-sanitizer-symbolizer access.log`, or pipe a log into it.
The symbolizer uses debuginfod; see [os-sanitizer/README.md](os-sanitizer/README.md#symbolisation).

### Running system-wide as a service

[os-sanitizer/run.sh](os-sanitizer/run.sh) installs the latest build as `/usr/local/sbin/os-sanitizer-collect`
and runs it under systemd with `--reference-policy --json /var/log/os-sanitizer/reports.jsonl`. Before
starting, it archives any earlier `reports.jsonl`. The unit file must exist first, and the upstream unit
can be used as a template:

```bash
sudo cp os-sanitizer/os-sanitizer.service /etc/systemd/system/os-sanitizer-collect.service
sudo ./os-sanitizer/run.sh
journalctl -u os-sanitizer-collect -f      # text reports
```

> **Caveat:** on kernels older than 6.14, removing probes can crash processes that are running at the time
> (journald crashed this way on our VM). Avoid stopping and restarting os-sanitizer repeatedly on such kernels.

## What we added

The paper leaves several gaps between the raw reports and something a person can act on. These are the
additions we made to close them:

1. **Kernel context and JSON output.** Each report now carries kernel-captured credentials (`euid`, `fsuid`, …),
   cgroup, executable identity, file ownership and mode of the opened object and its parent, the writers on the
   path, and the full resolved stack. This is written as JSON lines by
   [json_out.rs](os-sanitizer/os-sanitizer/src/json_out.rs). The changes span
   `os-sanitizer-common/src/lib.rs`, `os-sanitizer-ebpf/src/main.rs` and `os-sanitizer/src/main.rs`.
2. **Clustering** ([calibration/cluster.py](os-sanitizer/calibration/cluster.py)). Merges duplicate reports
   (same pass, binary and stack offsets) into clusters and extracts the features used for scoring.
3. **Scoring** ([calibration/score.py](os-sanitizer/calibration/score.py)). Gives each cluster a confidence
   score from 0 to 1, with a readable rationale, based on whether an untrusted user can influence a
   privileged process.
4. **Labelled calibration scenarios** ([calibration/scenarios.sh](os-sanitizer/calibration/scenarios.sh)).
   Runs the same binary in a benign context and in an exploitable one (uid 1001 as the attacker), so the
   ground truth comes from how each scenario is built.
5. **Benchmarks on our own VM** ([evaluation/quickbench/](os-sanitizer/evaluation/quickbench/)). These
   reproduce the paper's Tables 2–4. SPEC CPU 2017 is licensed, so five open workloads stand in for it:
   xz, gcc on `sqlite3.c`, perl, sqlite and stockfish.

### Headline results

| | Result |
|---|---|
| Live system (13.4 h, reference policy) | 207,854 reports → 1,910 clusters (109× fewer) → **99 clusters scoring ≥ 0.8** to review first |
| Calibration (threshold 0.5, 44 labelled clusters) | TP 8, FP 0, TN 34, FN 2: precision 1.00, recall 0.80 |
| Microbenchmarks (Table 2) | Kernel-side passes 1.0–9.5× slower, snprintf / printf_mut ≈ 30×, memory passes 63–159×. Same ranking as the paper. |
| Real workloads (Table 3) | Single kernel-side passes ≤ 2% overhead. All non-memory passes: gcc 1.45× (paper 1.36×). Memory passes: sqlite ≈ 4.3×. |

The full numbers, figures, slide deck and speaker notes are in
[os-sanitizer/Document/OS-Sanitizer_Presentation/](os-sanitizer/Document/OS-Sanitizer_Presentation/README.md).

## Reproducing the analysis

The scripts are plain Python 3. Run them from `os-sanitizer/`.

```bash
# Cluster reports (JSON lines or text logs), then score the clusters
python3 calibration/cluster.py /var/log/os-sanitizer/reports.jsonl -o clusters.csv
python3 calibration/score.py clusters.csv -o scored.csv

# Paper Table 4: reports per pass and severity (--exclude-test drops our calibration runs)
python3 calibration/table4.py /var/log/os-sanitizer/reports.jsonl --exclude-test -o table4

# Calibration: build the examples, run the labelled scenarios while os-sanitizer runs with --json,
# then score against the ground truth and sweep the threshold
make -C examples all
sudo calibration/scenarios.sh examples /tmp/calib
python3 calibration/cluster.py reports.jsonl -o clusters.csv
python3 calibration/score.py clusters.csv --truth /tmp/calib/ground_truth.tsv --sweep
```

Benchmarks (run as root; `OS_SAN` points at the os-sanitizer binary):

```bash
cd evaluation/quickbench
./setup.sh                                                   # install deps, generate data.tar and corpus.txt
./run_table2.sh mytag 10                                     # Table 2: microbenchmarks, all passes
python3 table2.py results/mytag/stdout.txt
./run.sh mytag 5 none reference-policy access all            # Table 3: real workloads
python3 table3.py results/mytag
```

The original evaluation scripts from the paper's artifact (SPEC, Speedometer, microbenchmarks) are in
[os-sanitizer/evaluation/](os-sanitizer/evaluation/README.md).

### Data included and left out

The processed results are committed:

| Path | Contents |
|---|---|
| `os-sanitizer/calibration/data/ground_truth*.tsv`, `scenarios*` | Calibration scenarios: labels, raw JSON reports, clusters and scores (v1 and v2) |
| `os-sanitizer/calibration/data/vm_20261112/` | Live-VM run: clusters, scores (v1 and v2), cluster summary, Table 4 (Markdown and LaTeX) |
| `os-sanitizer/Document/OS-Sanitizer_Presentation/data/` | The tables and CSVs behind every number in the presentation |

The following files are too large for GitHub, are generated locally, or are scratch files. They are
listed in [.gitignore](.gitignore):

| Left out | How to get it back |
|---|---|
| `os-sanitizer/os-sanitizer-ebpf/src/binding.rs` | `aya-tool generate task_struct dentry > os-sanitizer-ebpf/src/binding.rs` (see [Build](#2-generate-kernel-bindings)) |
| `os-sanitizer/evaluation/quickbench/inputs/data.tar` (64 MiB), `corpus.txt` (48 MiB) | `evaluation/quickbench/setup.sh`. `data.tar` is built from files on the local system, so it will not be byte-identical to ours. |
| `os-sanitizer/evaluation/quickbench/results/` | Raw benchmark output, written by `run.sh` / `run_table2.sh`. The final tables are in the presentation `data/` folder. |
| `os-sanitizer/calibration/data/enriched_*/` (≈ 260 MB) | The raw 13.4 h live-VM report snapshot (`reports_snapshot.jsonl`). It cannot be regenerated exactly, so collect a new one with `run.sh`. The clusters and scores derived from it are in `vm_20261112/`. |
| `target/`, `__pycache__/`, `*.core`, `vgcore.*`, `*.log`, LaTeX build files | Build output, crash dumps and scratch logs |

## Repository map

```
.
├── bpf-linker/                 upstream BPF linker (unmodified)
└── os-sanitizer/
    ├── os-sanitizer/           user-space loader and reporter (+ json_out.rs)
    ├── os-sanitizer-ebpf/      eBPF programs, one file per pass (binding.rs is generated)
    ├── os-sanitizer-common/    types shared by kernel and user space
    ├── os-sanitizer-symbolizer/
    ├── examples/               one small C program per pass (true positives / microbenchmarks)
    ├── calibration/            clustering, scoring, Table 4, labelled scenarios      [ours]
    │   └── data/               calibration and live-VM results (raw snapshot not committed)
    ├── evaluation/             paper's artifact scripts + quickbench/                [quickbench ours]
    ├── Document/               paper, project proposal, final deck, data and figures [ours]
    ├── presentation/           HTML source of the slide deck                         [ours]
    └── run.sh                  system-wide collector as a systemd service            [ours]
```

## License

- os-sanitizer: EUPL-1.2-or-later, see [os-sanitizer/LICENSE](os-sanitizer/LICENSE). Translations are in
  `LICENSE-translations/`.
- bpf-linker: MIT or Apache-2.0, at your option. See [LICENSE-MIT](bpf-linker/LICENSE-MIT) and
  [LICENSE-APACHE](bpf-linker/LICENSE-APACHE).
