---
title: "Permission denied when using docker command in Docker Desktop for mac"

categories: 
  - Docker
  - DevOps
last_modified_at: now
---
# Permission denied using docker command in mac
I was using Docker Desktop for mac with intel chip.<br>
When I tried to build docker image, I faced error message below.<br>
I was able to build image with sudo, but I couldn't bear it.
```
ERROR: open /Users/user/.docker/buildx/activity/desktop-linux: permission denied
```
The error was due to the ownership of the docker folder. I guess somehow Docker Desktop install as root user.<br>
I had to change the ownership from root to me to make it writable by me.<br>
The following command have solved the problem.
```
sudo chown -R $(id -u):$(id -g) $HOME/.docker
```
* References
  * [Docker forum](https://www.jenkins.io/doc/book/installing/linux/#fedora](https://forums.docker.com/t/error-failed-to-solve-error-getting-credentials-err-exit-status-1-out/136124/3)https://forums.docker.com/t/error-failed-to-solve-error-getting-credentials-err-exit-status-1-out/136124/3)
