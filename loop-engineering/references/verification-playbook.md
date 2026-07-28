# Verification Playbook

How to self-test common project types without asking the user to do anything.

Use this in PLAN §1.3, when the project type isn't obviously testable. Find your row, probe for
what it needs, install if missing, and write the verification into PLAN before building.

The rule throughout: **a weaker real check beats a stronger imagined one.**

## Platform note — macOS and Linux

Detect first, don't assume:

```bash
uname -s        # Darwin | Linux
command -v brew apt-get dnf pacman 2>/dev/null
```

| Need | macOS | Debian/Ubuntu | RHEL/Fedora |
|---|---|---|---|
| package manager | `brew install X` | `sudo apt install -y X` | `sudo dnf install -y X` |
| Docker | Docker Desktop / `colima start` | `docker.io` + user in `docker` group | `podman` often preinstalled — `alias docker=podman` mostly works |
| headless browser deps | bundled | `npx playwright install-deps` | `npx playwright install-deps` |
| virtual display | not needed | `xvfb-run -a <cmd>` | `xvfb-run -a <cmd>` |
| service manager | `launchctl` | `systemctl` | `systemctl` |
| open ports | `lsof -i :8000` | `ss -tlnp` | `ss -tlnp` |

All `sudo` installs are system-level — name them in PLAN for approval. Language-level installs
(`npm`, `pip`, `cargo`, `go install`) are project-local and run free on both platforms.

`sed -i` differs: GNU is `sed -i 's/a/b/'`, BSD/macOS needs `sed -i '' 's/a/b/'`. Use `perl -pi -e`
if you want one command that works on both.

---

## Browser extension (Chrome / Edge / Firefox)

The one people assume can't be automated. It can.

**Needs:** `npm i -D playwright` (or `puppeteer`), then `npx playwright install chromium`.

```bash
# Playwright launches a real Chromium with the unpacked extension loaded
npx playwright test
```

```js
// playwright.config.js — persistent context is required; extensions don't load otherwise
const ctx = await chromium.launchPersistentContext('', {
  headless: false,                    // MV3 service workers need headed; use xvfb-run in CI
  args: [
    `--disable-extensions-except=${EXT_PATH}`,
    `--load-extension=${EXT_PATH}`,
  ],
});
// service worker (MV3) — proves the background script actually booted
const [sw] = ctx.serviceWorkers();
const extId = sw.url().split('/')[2];
// popup / options page
await page.goto(`chrome-extension://${extId}/popup.html`);
// content script — navigate to a real page and assert it injected
await page.goto('https://example.com');
await expect(page.locator('#my-injected-banner')).toBeVisible();
```

Headless note: MV3 service workers don't run in old headless. Use `headless: false` under
`xvfb-run -a`, or Playwright's `chromium --headless=new`.

**Ladder if Playwright can't run:**
1. Validate `manifest.json` against the MV3 schema; assert required keys, permissions, paths resolve.
2. Unit-test content-script and background logic as plain modules (extract pure functions, mock `chrome.*`).
3. Build check: bundler exits 0, output files exist, no unresolved imports.
4. State plainly: "builds, manifest valid, logic tested — not verified rendering in a live browser."

---

## Web UI / SPA

**Needs:** Playwright or Puppeteer, or just `curl` + a DOM parser.

```bash
npm run dev &                                    # background it
npx wait-on http://localhost:3000                # don't sleep, wait properly
npx playwright test
kill %1                                          # always clean up
```

Assert on real state: element visible, text content, network response, no console errors
(`page.on('console')`), screenshot diff if you have a baseline.

**Ladder:** curl the SSR'd HTML and grep/parse it → component tests (Vitest + Testing Library,
no browser needed) → build succeeds and bundle emits.

## HTTP service / API

**Needs:** nothing beyond `curl` and `jq`.

```bash
uvicorn app:main --port 8000 &
npx wait-on tcp:8000 || until curl -sf localhost:8000/health; do sleep 0.3; done
curl -s -o /dev/null -w '%{http_code}' localhost:8000/api/me   # expect 200
curl -s localhost:8000/api/items | jq -e '.items | length > 0'
kill %1
```

Assert status codes, response shape via `jq -e` (non-zero exit on false), auth rejection paths,
and error cases — not just the happy path.

## CLI tool

```bash
./mytool --help                        # exit 0
./mytool build ./fixture               # exit 0
echo $?
diff -u expected.txt <(./mytool render ./fixture)
./mytool --bad-flag; test $? -ne 0     # failure paths matter too
```

## Library / package

Test suite is the verification. Also verify it's actually consumable:

```bash
pytest -q            # or: npm test -- --run
npm pack && npm i -g ./pkg-1.0.0.tgz && node -e "require('pkg')"   # import works after install
python -c "import mypkg; print(mypkg.__version__)"
```

## Container / image

```bash
docker build -t app:test .                      # exit 0
docker run -d --name t -p 8080:8080 app:test
until curl -sf localhost:8080/health; do sleep 0.5; done
docker logs t 2>&1 | grep -qv 'ERROR'
docker rm -f t
```

Check the image runs as non-root and the healthcheck passes, not just that it built.

## Kubernetes manifests / Helm

```bash
kubectl apply --dry-run=server -f .             # server-side, catches admission errors
helm template . | kubeconform -strict -          # or kubeval
helm lint .
# real cluster available? kind gives you one in ~30s
kind create cluster --name t && kubectl apply -f . && kubectl wait --for=condition=Ready pod -l app=x --timeout=120s
kind delete cluster --name t
```

## Terraform / IaC

```bash
terraform init -backend=false
terraform validate
terraform fmt -check
terraform plan -out=tf.plan          # plan is read-only — safe to run unattended
terraform show -json tf.plan | jq '.resource_changes[].change.actions'
```

`plan` is the verification. `apply` is irreversible — pre-approve it in PLAN or don't run it.

## Ansible

```bash
ansible-lint playbook.yml
ansible-playbook --syntax-check playbook.yml
ansible-playbook -i inv --check --diff playbook.yml     # dry run
# real target? molecule + docker driver
molecule test
```

## Database migration

```bash
# never test against real data — spin up a throwaway
docker run -d --name pgt -e POSTGRES_PASSWORD=x -p 5433:5432 postgres:16
alembic upgrade head          # forward
alembic downgrade -1          # and back — a migration you can't reverse isn't done
psql -h localhost -p 5433 -c '\d mytable'
docker rm -f pgt
```

## Scheduled job / cron / worker

Don't wait for the schedule. Invoke the handler directly with a synthetic payload, assert on
its output and side effects. Then separately validate the schedule expression parses and
computes the next-fire time you expect.

## Java / JVM

```bash
mvn -q verify                      # compile + test + package, exit 0
./gradlew build --console=plain
```

Spring Boot — start it, wait on the actuator, test, kill:

```bash
java -jar target/app.jar --server.port=8080 &
until curl -sf localhost:8080/actuator/health | jq -e '.status=="UP"'; do sleep 1; done
curl -s localhost:8080/api/items | jq -e '.[0].id'
kill %1
```

Verify the JAR is actually runnable, not just that it built: `java -jar` and check it boots.
For libraries, `mvn install` then compile a one-class consumer against it.
JUnit output is the evidence — quote the real `Tests run: 42, Failures: 0`.

## Go

```bash
go build ./... && go vet ./...
go test ./... -count=1        # -count=1 defeats the test cache
go test -race ./...           # concurrency bugs
./myapp & until curl -sf localhost:8080/health; do sleep 0.3; done; kill %1
```

## Rust

```bash
cargo build --release
cargo test
cargo clippy -- -D warnings
cargo fmt --check
```

## .NET

```bash
dotnet build -warnaserror
dotnet test --logger "console;verbosity=normal"
dotnet run --project src/Api &
until curl -sf localhost:5000/health; do sleep 0.5; done; kill %1
```

## Static site / SSG (Hugo, Jekyll, Astro, Next export)

```bash
npm run build                                  # or: hugo --minify
npx serve -p 4173 ./dist &
npx wait-on http://localhost:4173
npx linkinator http://localhost:4173 --recurse # dead links
npx lighthouse http://localhost:4173 --output=json --quiet --chrome-flags="--headless" \
  | jq '.categories.performance.score'
kill %1
```

Assert the build emitted the pages you expect (`test -f dist/about/index.html`), not just that
it exited 0.

## Linux service / systemd / packaging

```bash
systemd-analyze verify ./myapp.service        # unit file valid before installing
sudo systemctl daemon-reload && sudo systemctl start myapp
systemctl is-active myapp                     # expect: active
journalctl -u myapp -n 50 --no-pager | grep -qv 'error'
```

Packaging:

```bash
rpmlint ./myapp-1.0.rpm                       # or: lintian ./myapp_1.0.deb
rpm -qpl ./myapp-1.0.rpm                      # payload is what you expect
# install into a throwaway container, never the host
docker run --rm -v "$PWD:/pkg" rockylinux:9 bash -c \
  'dnf install -y /pkg/myapp-1.0.rpm && myapp --version'
```

SELinux: `ls -Z` for contexts, `ausearch -m avc -ts recent` for denials, and check
`getenforce` before blaming your code.

## Desktop / mobile app

The genuinely hard case. Be honest about it early.

- **Electron** — Playwright supports it via `_electron.launch()`. Fully testable.
- **iOS / Android** — needs a simulator and usually a paid runner. Realistically: unit-test the
  logic, run the build, verify it compiles and signs. State that you can't verify runtime UI.
- **Native desktop (Qt/GTK/WinForms)** — test the logic layer; UI generally not verifiable
  unattended.

For these, go to ladder rung 3 in PLAN — narrow the definition of done, in writing, before
building.

---

## Universal patterns

**Backgrounding a service.** Start it, wait for it properly (`npx wait-on`, or poll the health
endpoint), test, then always kill it — a leaked process poisons the next run. Use a trap:

```bash
trap 'kill %1 2>/dev/null' EXIT
```

**Ports.** Assume the obvious one is taken. Pick a high random port and pass it through.

**Network-restricted sandbox.** If installs fail, check for a registry mirror or vendored
`node_modules` before declaring blocked. Then fall down the ladder to checks needing no network
— lint, typecheck, unit tests, schema validation.

**Flaky ≠ failing.** If a check passes sometimes, that's a real finding to report, not
something to re-run until green.

**Always clean up.** Kill processes, remove containers, delete temp clusters — before REPORT.
Leaving a stray `docker run` behind is a side effect nobody asked for.

**Screenshots.** Save to `.loop/artifacts/` with descriptive names. Take them yourself — never
ask the user to. Look at them when a visual bug is suspected; assertions catch "the element
exists", only looking catches "it rendered behind the header."

## Remote targets over SSH

Declared in PLAN. Read-only diagnostics run unattended:

```bash
ssh host 'systemctl status myapp --no-pager; journalctl -u myapp -n 100 --no-pager'
ssh host 'df -h; free -m; ss -tlnp'
```

Writes to a remote host — editing files, restarting services, installing packages — are
pre-approved in PLAN, never improvised. Use `ssh -o BatchMode=yes` so a missing key fails fast
instead of hanging on a password prompt. Prefer `--check`/`--dry-run` modes first.

Against live URLs: `GET`/`HEAD` freely; `POST`/`PUT`/`PATCH`/`DELETE` is irreversible — approve
in PLAN. Never point a load test at production without sign-off.
