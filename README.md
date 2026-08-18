# gobox

Code execution environment for [Code Compete](https://github.com/shashwat-dixit/code-compete).

Judge0-shaped (async submit, familiar statuses, per-language compile/run, CPU / wall / memory limits) but **built here**, not hosted Judge0 and not Judge0’s source. Isolation is **Docker**, not Isolate-in-a-privileged-container.

This repo is built **by hand**. No product code lives here yet — use the checklist below as the build plan.

## What this is

Untrusted user code in, a verdict out:

```text
source + language + stdin tests + limits
        │
        ▼
   gobox (this repo)
        │  docker run --network=none ...
        ▼
   compile (if needed) → run each test → compare stdout
        │
        ▼
status, passed/total, time, memory, compile_output
```

Code Compete never compiles or runs player code in the API process. Matchmaking, auth, WebSockets, ELO, and problem storage stay in **code-compete**. Sandbox images, compile/run commands, limits, stdout compare, and isolation proofs live **here**.

## What this is not

- Hosted or self-hosted [Judge0](https://github.com/judge0/judge0) (GPL-3 — do not copy their source; copy the **product contract**)
- Public Judge0 CE / RapidAPI
- LeetCode GraphQL as a judge
- Match loop, OAuth, Redis Streams, or Postgres schema for matches
- `eval` / in-process execution

Judge0 CE is roughly `HTTP JSON → Postgres row + Resque → Isolate compile/run`. We want that **feel** (token, statuses, limits) with Docker as the jail and **no privileged host**.

## Ownership vs code-compete

| Lives in **gobox** | Lives in **code-compete** |
| --- | --- |
| Docker sandbox flags and teardown | API, GitHub OAuth, WebSocket fanout |
| Language images + compile/run commands | Problems, hidden tests, Postgres |
| Compile once → run tests → aggregate status | `submission.queued` / `execution.completed` streams |
| CPU / wall / memory / PID limits | Match state machine, first-AC-wins, ELO |
| Stdout compare (newline / trailing-space rules) | `worker-runner` as a **thin client** of gobox |
| Isolation proofs (no network, TLE, no leftover containers) | Resolver + live board (`passed_tests` / `total_tests`) |
| Optional localhost debug HTTP (`POST /internal/run`) | Public HTTP/WS — never a public Judge0 clone |

Contract code-compete expects: [docs/13-judge-runner.md](https://github.com/shashwat-dixit/code-compete/blob/main/docs/13-judge-runner.md) and [docs/06-execution-and-security.md](https://github.com/shashwat-dixit/code-compete/blob/main/docs/06-execution-and-security.md).

Suggested split in practice: gobox exposes an **internal** run API (or a Go library). `apps/worker-runner` in code-compete loads submission + tests from Postgres, calls gobox, publishes `execution.completed`. Gobox does not need Redis or the match DB.

## Statuses (Judge0 meanings, our names)

| id | Judge0 | gobox / Code Compete |
| --- | --- | --- |
| 1 | In Queue | `QUEUED` |
| 2 | Processing | `RUNNING` |
| 3 | Accepted | `ACCEPTED` |
| 4 | Wrong Answer | `WRONG_ANSWER` |
| 5 | Time Limit Exceeded | `TIME_LIMIT_EXCEEDED` |
| 6 | Compilation Error | `COMPILATION_ERROR` |
| 7–12 | Runtime Error | `RUNTIME_ERROR` (+ optional `signal`) |
| — | (Judge0 often folds MLE into RE/TLE) | `MEMORY_LIMIT_EXCEEDED` |
| 13 | Internal Error | `INTERNAL_ERROR` |

One **problem submission** is many runs (one stdin per test):

1. Compile once if needed. Non-zero → `COMPILATION_ERROR`, stop.
2. Run tests in order. First fail sets WA / TLE / MLE / RE. Remaining tests may be skipped.
3. All pass → `ACCEPTED`.
4. Return `passed_tests` / `total_tests` (what the live board shows). Never return hidden stdin/expected.

## Languages

Pinned tags, never `latest` in production. Source filenames are fixed so user paths never hit a shell. No client-supplied compiler flags in V1.

| key | image (starting point) | compile | run |
| --- | --- | --- | --- |
| `python` | `python:3.12-alpine` | none | `python3 main.py` |
| `cpp` | `gcc:14` | `g++ -std=c++17 -O2 -o main main.cpp` | `./main` |
| `go` | `golang:1.22-alpine` | `go build -o main main.go` | `./main` |
| `java` | `eclipse-temurin:17-jdk` | `javac Main.java` | `java -Xmx{mem}m Main` |

Inside the sandbox: user `coder` (uid 1000). Java needs a fatter wall-time and/or ~512MB for JVM warmup.

## Docker flags (minimum)

```text
docker run --rm \
  --network=none \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  --tmpfs /work:rw,noexec,nosuid,size=64m \
  --workdir /work \
  --user 1000:1000 \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  --pids-limit 64 \
  --memory <mb>m --memory-swap <mb>m \
  --cpus 1 \
  --ulimit nproc=64 \
  <image> <run command>
```

- Compile steps: tmpfs on `/work` **without** `noexec` (compilers write binaries). Still `nosuid,nodev`.
- Never mount `/var/run/docker.sock` into the **sandbox**. The **host** gobox process may talk to the Docker engine.
- No `--privileged`, no `--pid=host`, no `--volume /`.
- `docker rm -f` in a `defer`. Leftover containers are an outage.

## Internal run contract

What code-compete will send (source + tests loaded **there**, not on a stream):

```text
Run(ctx, Job) → Result

Job:    language, source, tests[{stdin, expected}], time_limit_ms, memory_limit_mb
Result: status, passed_tests, total_tests, runtime_ms, memory_kb,
        first_fail_index?, compile_output?, signal?
```

Compare stdout with **normalized newlines** and trailing spaces stripped per line. Do not trim interior spaces unless a problem says so.

Cap stdout/stderr (e.g. 1MB). Strip sandbox paths from `compile_output` / stderr before returning. Gobox must not receive `DATABASE_URL`, AWS keys, or API secrets.

Optional: localhost-only `POST /internal/run` for `curl` debugging. Do not expose it.

---

## Build checklist

Build in this order. Do not start later boxes in the same PR as the Python harness.

### 0. Repo skeleton

- [ ] Go module (`github.com/shashwat-dixit/gobox`)
- [ ] `cmd/` binary that can run a job from CLI (stdin/file) without Redis/Postgres
- [ ] Makefile / `go test ./...`
- [ ] CI: fmt, vet, test
- [ ] `.env.example` only (no secrets)
- [ ] Pinned language images documented; **no** `docker pull` on the submit hot path (pre-pull)

### 1. Python harness (first useful slice)

- [ ] Language table (or YAML) with `python` only
- [ ] Write source to a host temp dir (`0700`, not in git)
- [ ] Run `main.py` in a fresh container with the Docker flags above
- [ ] Pipe one stdin, capture stdout/stderr, compare to expected
- [ ] Return `ACCEPTED` / `WRONG_ANSWER`
- [ ] Tear down container + temp dir on success **and** error
- [ ] Proof: two-sum (or similar) judged locally

### 2. Isolation proofs (required before more languages)

- [ ] Network off: `socket` / `curl` in user code fails
- [ ] TLE: `while True: pass` → `TIME_LIMIT_EXCEEDED`, container gone
- [ ] Wall clock > CPU limit so `sleep 100` dies (CPU-only limits miss this)
- [ ] Memory cap → `MEMORY_LIMIT_EXCEEDED` (or documented mapping if the runtime surfaces it as RE)
- [ ] Output cap: huge stdout does not blow the worker
- [ ] No Docker socket / host FS from inside the sandbox

### 3. Compile pipeline + remaining languages

- [ ] Compile step separate from run; CE stops the job
- [ ] Compile tmpfs allows exec on `/work` only
- [ ] `cpp`
- [ ] `go`
- [ ] `java` (higher wall-time / memory budget documented)
- [ ] Fixed filenames only (`main.py`, `main.cpp`, `main.go`, `Main.java`)
- [ ] User strings never interpolated into a shell

### 4. Multi-test aggregation

- [ ] Compile once, run N tests in order
- [ ] First failure sets status; later tests may skip
- [ ] `passed_tests` / `total_tests` / `first_fail_index`
- [ ] Hidden fixtures never appear in logs at info, in HTTP responses, or in anything code-compete might echo to WS

### 5. Limits and resources (every run)

- [ ] `cpu_time_limit` from the job (`time_limit_ms`)
- [ ] `wall_time_limit` ≈ 2–3× CPU (or +2s)
- [ ] `memory_limit` from the job (`memory_limit_mb`) + swap equal to memory
- [ ] `--cpus 1`, `--pids-limit`, `nproc` ulimit
- [ ] Non-root `coder`, dropped caps, `no-new-privileges`, read-only root

### 6. Internal API (so code-compete can call it)

- [ ] Library `Run(ctx, Job) → Result` **or** localhost HTTP that wraps the same thing
- [ ] If HTTP: bind localhost; no public Judge0 clone; no `wait=true` that holds workers hostage
- [ ] Timeouts on the Docker client; context cancel kills the container
- [ ] Idempotency: same job id twice does not leak a second long-running sandbox (document the rule)

### 7. Hook up to Code Compete

- [ ] Document the exact request/response JSON code-compete `worker-runner` will use
- [ ] Local: gobox on the **host** so it can use the host Docker engine; compose stays Postgres + Redis in code-compete
- [ ] code-compete adapter: `submission.queued` → load source/tests → gobox → `execution.completed`
- [ ] Adapter does not re-implement sandbox logic

### 8. Observability

- [ ] Queue wait vs execution wall time (if gobox queues)
- [ ] Status counts (AC / WA / TLE / RE / CE / MLE)
- [ ] Container start failures
- [ ] Structured logs with a job/submission id; **do not** log full source at info (users paste secrets)

### 9. Later (not V1)

- [ ] gVisor (`runsc`) / Firecracker
- [ ] Extra languages beyond the four
- [ ] Base64 source (JSON UTF-8 is enough)
- [ ] Image build pipeline for slimmer custom images

## Review blockers

Fail the PR if any of these land:

- Privileged containers, or Docker socket inside the sandbox
- User-controlled strings interpolated into a shell
- Hidden tests returned to a caller that might reach a player
- Missing wall-time limit
- Containers left running after errors
- Pulling images per submit
- Secrets (`DATABASE_URL`, cloud keys) injected into the sandbox
- Judge0 source copied into this tree

## Suggested PR order

1. CLI Python harness (checklist §0–§1)
2. Network + TLE proofs (§2)
3. C++, then Go, then Java (§3)
4. Multi-test aggregation + internal API (§4–§6)
5. code-compete `worker-runner` adapter (§7)

Do not mix 2–5 into the first PR.
