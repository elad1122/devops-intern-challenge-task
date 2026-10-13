# Root Cause Analysis

Fill this in as you go. Bullet points are fine — be specific, not long.

---

## Defect 1: Proxy image fails to build

### Symptom / error output
### got this : ERROR: failed to build: failed to solve: process "/bin/sh -c apt-get update &&     apt-get install -y nginx curl htop vim git" did not complete successfully: exit code: 100
### dpkg: error processing package nginx-common (--configure): , What would you like to do about it ? which arent make sense because usually these are questions with pre-setted answers. 
### 14.42  nginx depends on nginx-common (= 1.24.0-2ubuntu7.18); however:
### 14.42   Package nginx-common is not configured yet.
### 13.83  end of file on stdin at conffile prompt


### How I diagnosed it
### Tried to run: docker image build -t proxy-test . and saw the build logs through it. 

### Root cause
### nginx wasnt installed before running the COPY command, so the copied nginx.conf created a conflict between the default one and it.
### it caused dpkg (package manager) to write prompt of new configuration questionire 
##  Also the exit code 100 , told me that questionarei caused end of file in stdin in the container and failed the build
### Fix:
### added ARG of DEBIAN_FRONTEND to be noniteractive to surpress some of the terminal questions
### moved COPY nginx.conf below RUN, because we do want the customed ngnix.conf override the default of nginx


---

## Defect 2: Proxy cannot reach the backend

### Symptom / error output
### curl: (6) Could not resolve host: app_backend
###   Trying 127.0.0.1:8080...
### connect to 127.0.0.1 port 8080 from 127.0.0.1 port 60872 failed: Connection refused
### Failed to connect to 127.0.0.1 port 8080 after 0 ms: Couldn't connect to server


### How I diagnosed it
### docker compose up -d
### docker exec -it devops-intern-challenge-task-proxy-1 curl -v http://app_backend:8080/api/info
### docker exec -it devops-intern-challenge-task-proxy-1 curl -v http://127.0.0.1:8080/health

### Root cause
### in docker compose , the containers names are defined by their service name in the compose.yml.
### in containers, 127.0.0.1 is internal ip tha never leaves the container's network.
### when i did the curl of 127.0.0.1:8080 it didnt leave the network and did the search in the proxy container (which didnt find becasue it listens to 80)
### when i did the curl -f app_backend:8080 , it wasnt the host of the backend  

### Fix
### changed in nginx.conf -> server backend:8080
### then launched docker compose up -d --build proxy to rebuild image for the proxy container and restart the container in the docker compose.
---

## Defect 3: Tests still fail after fixing defect 2

### Symptom / error output
### Testing /nginx-health...
### PASSED: /nginx-health
### Testing proxied backend /api/info...
### TEST RUNNER CRASHED: HTTP Error 502: Bad Gateway
### Starting integration tests against http://proxy:80...
### Testing /nginx-health...
### PASSED: /nginx-health
### Testing proxied backend /api/info...
### PASSED: /api/info proxied successfully!
### ALL TESTS PASSED!

### How I diagnosed it
### docker logs devops-intern-challenge-task-test-runner-1 -f 

### Root cause
### race condition between depeneds_on of the proxy and the actual srart of the backend service in docker compose.
### depends_on only waits for the service to start but the app itself could have a deplay and might finish boot just a second after
### that's why i saw it only at first (bacause the backend was ready after running it again)

### Fix, and why I chose it over the alternatives
### in the docker-compose.yml , added health checks (using runtime binary of python because curl wasnt installed to the backend app) to make sure the app is ready before any other service reaches it.
### also , in proxy service added condition for the backend to check if healthy because without it , it defaults to service_started condition and it causes the race_condition 
### then, docker compose down && docker compose up -d
### I believe adding healthchecks + condition is a better fix then adding retry logic in the code of the tests due to the following reasons:

### Fast and accurate failure signals: If the backend crashes during boot, Docker flags it as unhealthy right away. With test retries, the runner just keeps polling a broken service, hiding the real crash behind repeated 502s until it finally times out.
### No duplicated boilerplate: Readiness belongs in the infrastructure config (compose.yaml). Putting retries in the test script means every new test file or test runner we add later has to re-implement the same polling and sleep logic.
### Keeps proxy logs clean: Gating the proxy behind the backend's health check stops Nginx from taking traffic before the upstream is ready. With retries, Nginx's error logs get flooded with false-positive 502 Bad Gateway and Connection refused errors on every cold start.
---

## Defect 4: CI pipeline fails

### Symptom / error output

### How I diagnosed it

### Root cause

### Fix

---

## Bonus questions

Optional. Answer only the ones you got to — number them, skip the rest.

>

---

## Anything you would do differently with more time
