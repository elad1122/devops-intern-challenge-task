# Root Cause Analysis

Fill this in as you go. Bullet points are fine — be specific, not long.

---

## Defect 1: Proxy image fails to build

### Symptom / error output
ERROR: failed to build: failed to solve: process "/bin/sh -c apt-get update &&     apt-get install -y nginx curl htop vim git" did not complete successfully: exit code: 100
dpkg: error processing package nginx-common (--configure): , What would you like to do about it ?  
nginx depends on nginx-common (= 1.24.0-2ubuntu7.18); however:
Package nginx-common is not configured yet.
end of file on stdin at conffile prompt


### How I diagnosed it
Tried to run: docker image build -t proxy-test . to inspect the build logs through it. 

### Root cause
COPY nginx.conf /etc/nginx/nginx.conf was placed before the RUN apt-get install command. When nginx-common was installed, dpkg detected that /etc/nginx/nginx.conf already existed and differed from the package's default version. This triggered an interactive conffile prompt. Because Docker builds run non-interactively without an attached stdin, dpkg encountered EOF and failed with exit code 100.
### Fix:
added ARG of DEBIAN_FRONTEND to be noniteractive to surpress some of the terminal questions
moved COPY nginx.conf below RUN apt-get, because we do want the customed ngnix.conf override the default of nginx


---

## Defect 2: Proxy cannot reach the backend

### Symptom / error output
curl: (6) Could not resolve host: app_backend
  Trying 127.0.0.1:8080...
connect to 127.0.0.1 port 8080 from 127.0.0.1 port 60872 failed: Connection refused
Failed to connect to 127.0.0.1 port 8080 after 0 ms: Couldn't connect to server


### How I diagnosed it
docker compose up -d
docker exec -it devops-intern-challenge-task-proxy-1 curl -v http://app_backend:8080/api/info
docker exec -it devops-intern-challenge-task-proxy-1 curl -v http://127.0.0.1:8080/health

### Root cause
In Docker Compose, containers reach each other across a shared network using their service names as hostnames. The proxy configuration failed because it tried routing traffic to 127.0.0.1 and app_backend.
127.0.0.1 points only to the proxy container itself, so it never leaves the container to find the backend.
app_backend does not exist on the Compose network, the service is defined simply as backend.

### Fix
Updated proxy/nginx.conf to set proxy_pass http://backend:8080/
Rebuilt the proxy container with docker compose up -d --build proxy 
---

## Defect 3: Tests still fail after fixing defect 2

### Symptom / error output
Testing /nginx-health...
PASSED: /nginx-health
Testing proxied backend /api/info...
TEST RUNNER CRASHED: HTTP Error 502: Bad Gateway

### How I diagnosed it
Ran docker compose logs test-runner -f and noticed the 502 Bad Gateway only occurred on the very first test execution right after docker compose up.

### Root cause
A startup race condition. Standard depends_on only waits for the upstream container to start running, not for the application inside it to finish booting and be ready for requests. The test runner sent requests to the proxy before the Python backend had finished starting up on port 8080, causing Nginx to return a 502 Bad Gateway.

### Fix, and why I chose it over the alternatives
Added a health check to the backend service in compose.yaml using Python's standard library urllib.request
Updated proxy's depends_on block to require condition: service_healthy

I believe adding healthchecks + condition is a better fix then adding retry logic in the code of the tests due to the following reasons:

Accurate failure signals: If the backend crashes during boot, Docker marks it unhealthy immediately. Polling loops in test code hide crashes behind timeouts.
Separation of concerns: Readiness belongs in infrastructure orchestration (compose.yaml), not duplicated across test scripts.
Clean logs: Prevents Nginx error logs from being flooded with false-positive 502s on cold starts.
---

## Defect 4: CI pipeline fails

### Symptom / error output
Run ./scripts/run-tests.sh
/home/runner/work/_temp/0775c08b-9472-4408-a070-499c319b8125.sh: line 1: ./scripts/run-tests.sh: Permission denied

### How I diagnosed it
Pushed changes to GitHub and inspected the failed job logs under the "Execute test suite" step 

### Root cause
Because operating systems default to stripping execute permissions via umask, the test script was initially committed to Git as a standard non-executable file (644). When CI runners clone the repository, Git restores that exact mode on disk, causing the Linux kernel to reject direct execution with Permission denied

### Fix
Ran git update-index --chmod=+x scripts/run-tests.sh to flip the mode bit from 644 to 755 directly in Git's index.
Committed and pushed the updated file mode so future clones retain the executable bit automatically. 
---

## Bonus questions

Optional. Answer only the ones you got to — number them, skip the rest.
1. changed the base image to be nginx:alpine instead of ubuntu:24.04 , This comes pre-configured with Nginx on top of Alpine Linux rather than a full Ubuntu distribution.
Removed apt-get update and the installation of debugging and editing packages (htop, vim, git, curl) , once the container works and connectivity to backend works too i wont need these packages anymore.
(see the full Dockerfile in the Dockerfile of proxy under comments)
got the images size by running:
docker image build -t proxy:after .  , docker image build -t proxy:before .
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}" | grep -E "REPOSITORY|proxy"
before: 452MB after: 93.6MB

2. modified the proxy container Dockerfile by Configure directories and PID file permissions for the pre-existing 'nginx' user.
RUN touch /var/run/nginx.pid && \
    chown -R nginx:nginx /var/run/nginx.pid /var/cache/nginx /etc/nginx
USER nginx
(see the full Dockerfile in the Dockerfile of proxy under comments)
in nginx.conf -> changed the listen port to 8080
in docker-compose.yml -> in the proxy service ports , changed to 8081:8080 host-port:container-port and in the test service , changed the environment of PROXY_URL port to 8080 because its no longer 80

>

---

## Anything you would do differently with more time

Not Related to the task itself but If I had more time, i would try to improve the ci pipeline of this project by eliminating duplicate PR trigger runs, tightening job timeouts from 20 to 5 minutes, and caching Docker layers to speed up integration test feedback loops.
