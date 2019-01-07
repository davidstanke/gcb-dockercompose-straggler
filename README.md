# gcb-dockercompose-straggler
## Repro of a strange logging behavior by Cloud Build when running docker-compose

I've been experimenting with using docker-compose in Cloud Build and there's a strange behavior that is tripping me up.

The basic idea is that we use `docker-compose up` to spin up a multi-service environment, and then run an external command (ie. an E2E test) against that environment. To do so, we use docker-compose with the `-d` argument so that the environment persists in the background—in "detached" mode—ready for incoming connections.

### Normal behavior on local
To see the normal behavior, clone this repo and run:
```
# the compose config is designed to run in Cloud Build, where there is a network 'cloudbuild'
docker network create cloudbuild

docker-compose up
```
...you should get output like:
```
Creating web ... done
```
At this point, the docker-compose environment is running as a background service, ready to accept connections (but only from within the 'cloudbuild' network).

### Strange behavior in Cloud Build
Now, use Cloud Build to spin up the compose environment:
```
gcloud builds submit .
```

The cloudbuild.yaml config will spin up the docker-compose environment, which will respond to connections. Everything works, except for the logs are strange:
```
...
Step #0: Status: Downloaded newer image for python:latest
Step #0: Creating web ...
Step #0: Creating web
Finished Step #0
Starting Step #1
Step #1: Already have image (with digest): gcr.io/cloud-builders/curl
Step #1: HTTP/1.0 200 OK
Step #1: Server: SimpleHTTP/0.6 Python/3.7.2
Step #1: Date: Mon, 07 Jan 2019 16:00:49 GMT
Step #1: Content-type: text/html; charset=utf-8
Step #1: Content-Length: 987
Step #1:
Finished Step #1
Starting Step #2
Step #2: Already have image (with digest): gcr.io/cloud-builders/curl
Step #2: sleeping for 30 sec ...
Step #2: done sleeping
Finished Step #2
PUSH
Creating web ... done
Step #0:
```
The logs indicate that steps 0, 1, and 2 are all finished, and then the entire build is completed (as indicated by the "PUSH" statement). And *then* we see the final output of the initial step: `Creating web ... done`. The fact that the curl request was successful suggests that this event happened way back in Step 0. But it doesn't show up in the log until after the end of the build.

This isn't a big deal. Except it's super annoying when trying to troubleshoot... I've been trying to figure out why my docker-compose environment won't start, but actually it does start. That has made me bark up several wrong trees. :( What's going on here?
