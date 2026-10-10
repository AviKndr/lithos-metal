# CI

Two lanes. Hosted runners check every push and pull request; bare-metal Macs run what needs a real Apple GPU, on
request for pull requests because a self-hosted machine executes the code under test.

| Lane | Workflow | Runs on | When | Required check |
|---|---|---|---|---|
| 1. Hosted | `ci.yml` | GitHub-hosted Linux and macOS | every push and pull request | `ci-finish` |
| 2. GPU tiers | `mac-gpu.yml` | self-hosted Macs | `/mac-test` on a pull request; every push to `main` | `mac-gpu (<runner>)` |

## Lane 1: hosted

The contract tier on Linux (Python 3.10 and 3.13), the hygiene and extension checks (CONTRIBUTING.md), and a
`macos-26` job that, for Python 3.10, 3.13 and 3.14, builds the Metal module, imports it, collects the GPU tiers and
runs the contract tier on Darwin. Hosted macOS is virtualized: its paravirtual GPU cannot run the kernel tests and its numbers mean nothing.
`ci-finish` passes when every other job passed or was skipped; it is the one check branch protection requires.

## Lane 2: the GPU tiers on bare-metal Macs

`tools/ci/mac_gpu.py` builds the checkout into a new virtual environment under `$LITHOS_CI_ROOT/venv` every run (the
`dev,oracle,serve` extras, mlx-lm, httpx), downloads the checkpoints the oracle tiers read (`tools/ci/models.json`:
public Hub repositories pinned to a commit, every file checked against its SHA-256 on every run, by
`tools/ci/provision.py`) and checks out the DeepSpec reference the drafter oracle imports afresh (a pinned commit): the
tests run as the runner's user and could have rewritten what an earlier run left. It then runs `tests/contract`,
`kernels`, `runtime`, `spec`, `layers` and `models`, one pytest run per tier. A test may skip only for a reason
`ALLOWED_SKIPS` lists: a module that did not build, a missing checkpoint, package or chip profile skips silently on a
developer's machine and fails here. `--shader-validation` (`/mac-test --shader-validation`) runs the Metal tiers under
`MTL_SHADER_VALIDATION=1`; use it for kernel and emitter changes (CLAUDE.md).

A maintainer starts the lane on a pull request by commenting, after reading the change:

```
/mac-test                          every runner in MAC_CI_RUNNERS
/mac-test m5_max_40c m4_pro        named runners
/mac-test --shader-validation
/mac-test <full SHA>               the head you read: required when the author cannot write here
```

`mac-slash.yml` runs from the default branch, checks that the commenter has write access and that the pull request is
open and ready, pins its head commit (refusing a fork's pull request whose head is not the commit named: the author
could push between the comment and the job), marks a pending `mac-gpu (<runner>)` status and dispatches `mac-gpu.yml` on
the default branch with that commit as input. The Mac job checks out the commit without credentials and holds only a
read token; a hosted job sets the final statuses from the uploaded `result.json`, so no token that can write reaches the
Mac. A new `/mac-test` on the same pull request cancels the older run. A dispatch that runs only some tiers reports
`mac-gpu partial (<runner>)`, a status no ruleset requires. No workflow may name a Hugging Face token
(`tools/ci/hygiene.py`): CI reads public checkpoints only.

## A runner Mac

Any bare-metal Apple silicon Mac on macOS 26 with the Command Line Tools: an office machine or an EC2 Mac
(`mac-m4pro.metal`, `mac-m4max.metal`, `mac-m3ultra.metal`, … on a dedicated host, allocated for at least 24
hours; grow the APFS container to the EBS volume after the first boot). The GPU tiers fit a 24 GB machine.

1. `brew install python@3.12`; keep the machine awake (`sudo pmset -a sleep 0 disablesleep 1`); 60 GB free for
   the checkpoints and environments.
2. Settings → Actions → Runners → New self-hosted runner (macOS, ARM64), then in the runner directory
   `./config.sh --url https://github.com/lithos-ai/lithos-metal --token <token> --labels <backend> --unattended`
   and `./svc.sh install && ./svc.sh start`. The label is the chip's backend name under
   `monolith/backends/metal/` (`m5_max_40c`, `m4_pro`, …) or the name a new backend will take.
3. In the runner directory's `.env`:

   ```
   LITHOS_CI_ROOT=/Users/<user>/lithos-ci
   LITHOS_CI_PYTHON=/opt/homebrew/bin/python3.12
   PATH=/opt/homebrew/bin:/usr/bin:/bin:/usr/sbin:/sbin
   # optional: a command prefix that serializes GPU work with other users of the machine
   LITHOS_CI_GPU_WRAPPER=/path/to/lock --
   ```

   `LITHOS_CI_ROOT` must be a real directory, not a symlink: the lanes delete and rebuild what is under it.

4. Add the label to the repository variable `MAC_CI_RUNNERS` (a JSON list, e.g. `["m5_max_40c", "m4_pro"]`).

## Repository settings

* Branch ruleset on `main`: pull requests required; required checks `ci-finish`, and `mac-gpu (<runner>)` once the
  runners are stable.
* Actions → General → "Require approval for all external contributors": a fork's workflows run only after a
  maintainer approves them.
* Variable `MAC_CI_RUNNERS`.
