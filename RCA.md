# Root Cause Analysis

Fill this in as you go. Bullet points are fine — be specific, not long.

---

## Defect 1: Proxy image fails to build

### Symptom / error output

Log lines showing where the build failed:

```
Configuration file '/etc/nginx/nginx.conf'
==> File on system created by you or by a script.
==> File also in package provided by package maintainer.
What would you like to do about it ?  Your options are:
Y or I  : install the package maintainer's version
N or O  : keep your currently-installed version
D     : show the differences between the versions
Z     : start a shell to examine the situation
The default action is to keep your current version.
*** nginx.conf (Y/I/N/O/D/Z) [default=N] ? dpkg: error processing package nginx-common (--configure):
end of file on stdin at conffile prompt
```

### How I diagnosed it
1. Ran docker build . from the proxy/ directory. The build failed on the apt-get install step.

2. The last line of the error didn't say what went wrong, so I sent the full build log to an AI assistant to help me understand it.

3. The AI pointed to a question the installer asked about /etc/nginx/nginx.conf. At first I wasn't convinced, because the log said the default answer is to keep the current file, so I thought the installer just picked that and continued.

4. I checked the full log again and found the smoking gun right after the question: "dpkg: error processing package nginx-common (--configure): end of file on stdin at conffile prompt". dpkg (the tool that actually installs the packages) asked the question, got no answer because nobody can type anything during a Docker build, and failed. Since nginx depends on nginx-common, it failed too.

5. Checked the Dockerfile and saw the cause: COPY nginx.conf runs before apt-get install nginx.

6. To confirm the cause, I swapped the order of the steps in the Dockerfile so nginx is installed first and nginx.conf is copied after it. I rebuilt the image and the build passed.

### Root cause
The problem was the order of the steps in the Dockerfile. Our nginx.conf was copied to /etc/nginx/nginx.conf before nginx was installed. When apt-get then installed nginx, the package found a config file it didn’t create and stopped to ask whether to replace it. A docker build has no one to answer, so the installation failed and the build failed with it.

### Fix
Instead of patching the original Dockerfile, I rewrote it following Docker best practices:

1. Use the official nginx:alpine base image: nginx comes pre-installed, so there’s no apt-get install step and no config-file prompt. Our nginx.conf is simply copied over the default one.

2. Smaller image: Switching from ubuntu:24.04 to Alpine and dropping tools the proxy doesn’t need (curl, htop, vim, git) reduces the image size and the attack surface.

3. Run as a non-root user. Added USER nginx, gave that user ownership of /var/cache/nginx, and moved the PID file to /tmp so nginx can start without root permissions.

---

## Defect 2: Proxy cannot reach the backend

### Symptom / error output
Requests to the backend through the proxy (for example http://localhost:8081/health) returned 502 Bad Gateway. The /nginx-health endpoint still returned 200, because nginx answers it by itself without the backend.

Log lines from the proxy:
```
[error] 21#21: *1 connect() failed (111: Connection refused) while connecting to upstream, client: 192.168.65.1, server: localhost, request: "GET /favicon.ico HTTP/1.1", upstream: "http://127.0.0.1:8080/favicon.ico", host: "localhost:8081", referrer: "http://localhost:8081//nginx-health"
```

### How I diagnosed it
1. Brought up the backend and the proxy with docker compose and opened http://localhost:8081/health in the browser. Got 502 Bad Gateway.
2. Using the paths listed in the README, I checked each container on its own in the browser. The backend (localhost:8080/health and
 /api/info) and the proxy (localhost:8081/nginx-health) both worked. But going through the proxy to the backend (localhost:8081/health) returned 502 Bad Gateway, so the problem was the connection between them.
3. Ran docker compose logs proxy and found the smoking gun: "connect() failed (111: Connection refused) while connecting to upstream", with upstream "http://127.0.0.1:8080".
4. Used an AI assistant to help me understand the log. nginx was sending requests to 127.0.0.1, which is the loopback address of the proxy container itself. The backend runs in a different container, so nothing was listening on port 8080 there and the connection was refused.
5. Checked proxy/nginx.conf and saw the upstream was set to server 127.0.0.1:8080.

### Root cause
The nginx upstream pointed to 127.0.0.1:8080. Every container has its own loopback address, so nginx was connecting to port 8080 inside its own container. nginx listens on port 80 and the backend runs in a separate container, so nothing was listening on port 8080 there and the connection was refused. Docker Compose puts all services on a shared network where each service is reachable by its service name, so the correct address is backend:8080.

### Fix
Changed the upstream in proxy/nginx.conf from server 127.0.0.1:8080 to server backend:8080, then rebuilt and restarted the proxy. Requests through the proxy to /health and /api/info now reach the backend.

---

## Defect 3: Tests still fail after fixing defect 2

### Symptom / error output
After fixing the proxy routing, ./scripts/run-tests.sh still failed. The /nginx-health test passed, but /api/info returned 502, and test-runner-1 exited with code 1.

```
test-runner-1  | TEST RUNNER CRASHED: HTTP Error 502: Bad Gateway
proxy-1        | [error] connect() failed (111: Connection refused) while connecting to upstream, client: 172.19.0.4, server: localhost, request: "GET /api/info HTTP/1.1", upstream: "http://172.19.0.2:8080/api/info", host: "proxy:80"
test-runner-1 exited with code 1
...
backend-1      | [Backend] loaded config key: service
```

### How I diagnosed it
1. The upstream in the error was now 172.19.0.2:8080, the backend container's IP, so the routing fix from Defect 2 worked. But the connection was still refused, which made me suspect a timing problem: the tests were running before the backend was ready.
2. The log confirmed it: by the time the test failed, the backend had printed only "loaded config key: service" and never "Backend listening on port 8080...".
3. Read backend/app.py to understand why it starts so slowly. Before starting the server, load_config() loads 4 config keys and sleeps 2 seconds for each one, so the backend needs about 8 seconds before it accepts any connection.
4. Checked docker-compose.yml. The proxy already had depends_on: backend, and the test runner had depends_on: proxy, but depends_on only waits for a container to start, not for the application inside it to be ready. So I needed a way to wait until the backend actually answers, either with a delay or with a healthcheck.

### Root cause
The backend takes about 8 seconds to start listening because it loads its config first. docker compose only made the proxy and test runner wait until the backend container started, not until the app inside it was ready. The tests ran too early and nginx returned 502 because nothing was listening on the backend yet.

### Fix, and why I chose it over the alternatives
Added a healthcheck to the backend that calls /health, and a healthcheck to the proxy that calls /nginx-health. Then changed depends_on to condition: service_healthy, so each service starts only after the one before it is actually ready:
backend healthy → proxy starts → proxy healthy → test runner starts.

Alternatives I considered:
1. Adding a sleep of a few seconds before the tests. I rejected it because it's a guess: too short and it fails on a slower machine or in CI, too long and every run wastes time. It hides the problem instead of solving it.
2. Changing backend/app.py to start faster. Not allowed, since it's treated as a third-party service.
3. Adding retries inside the tests. That changes the tests to tolerate a problem in the environment, which goes against the rule of not weakening the tests.

The healthcheck is reliable: we only move forward once the container is up and its main application answers /health with 200, no matter how long that takes on a given machine.

---

## Defect 4: CI pipeline fails

### Symptom / error output
The GitHub Actions job failed at the "Execute test suite" step, before any test ran:

```
/home/runner/work/_temp/8414b550-e026-485a-8981-8d6a25921b0c.sh: line 1: ./scripts/run-tests.sh: Permission denied
Error: Process completed with exit code 126.
```

### How I diagnosed it
1. Opened the failed run in the Actions tab of my repo and read the log of the "Execute test suite" step.
2. The error was "Permission denied" with exit code 126. I searched online and learned that 126 means the file was found but isn't allowed to run, so it's an execute permission problem and not a problem in the script itself.
3. Checked .github/workflows/ci.yml: it runs ./scripts/run-tests.sh directly, which requires the file to be executable.

### Root cause
scripts/run-tests.sh was committed without execute permission. GitHub Actions makes a fresh clone of the repo, so it got a file that isn't allowed to run.

### Fix
Gave the script execute permission on my machine with chmod +x scripts/run-tests.sh, then committed and pushed the change. Now every fresh clone, including CI, gets an executable file, so it stays fixed for whoever clones the repo next.

---

## Bonus questions

Optional. Answer only the ones you got to — number them, skip the rest.

1. **Smaller proxy image**
   - Changed the base image from ubuntu:24.04 to the official nginx:1.30.5-alpine, which already includes nginx, so there is no apt-get step at all.
   - Removed tools the proxy doesn't need at runtime: curl, htop, vim and git. Fewer packages also means fewer tools for an attacker to use if the container is compromised.
   - Image size: 485 MB before (ubuntu:24.04 with nginx and the extra tools) → 92.1 MB after (nginx:1.30.5-alpine).

2. **Running the proxy as a non-root user**
   - Added USER nginx to the Dockerfile. The nginx user already exists in the official image, so no need to create it.
   - nginx writes temporary files to /var/cache/nginx, which belongs to root, so I gave the nginx user ownership of it with chown -R nginx:nginx /var/cache/nginx.
   - nginx saves its PID file to /var/run by default, which is also owned by root, so I moved it to /tmp with -g "pid /tmp/nginx.pid; daemon off;" in the CMD.
   - The proxy still listens on port 80. Docker allows non-root processes to bind low ports inside containers, so no change was needed in nginx.conf or docker-compose.yml.

3. **Additional improvements**
   - Pinned the base image to the exact version nginx:1.30.5-alpine instead of latest, so every build is identical. The trade-off is that security patches need a manual version bump.
   - Added healthchecks to the backend and the proxy, so services start only when the ones they depend on are actually ready.

---

## Anything you would do differently with more time

- Stronger CI checks. Add a Dockerfile linter (hadolint) and an image vulnerability scan (Trivy) to the pipeline, so problems like the ones in this challenge are caught before they reach main.

- Learn more about the backend. The backend loads its config from what's meant to be a remote config service, which is why it needs about 8 seconds to start. My healthcheck timings are based on what I saw on my own machine. With more time, I'd find out how slow startup can get in a real environment (for example, if the config service is under load) and set the healthcheck limits based on that.

