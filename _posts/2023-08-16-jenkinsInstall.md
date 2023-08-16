---
title: "An error when installing jenkins on Amazon Linux"

categories: 
  - AWS
  - DevOps
  - Jenkins
last_modified_at: now
---
# An error when installing jenkins on Amazon Linux based EC2
I faced following error when I tried "sudo yum install jenkins"
```
...
You can remove cached packages by executing ‘yum clean packages’.
Error: GPG check FAILED
```
There were suggested solution using apt-get command.<br>
I couldn't use those solutions since amazon linux uses yum as default.<br>
I could solve the problem by installing through dnf.
```
sudo wget -O /etc/yum.repos.d/jenkins.repo \
    https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key

// following commands are needed
sudo dnf upgrade
sudo yum install java-11-amazon-corretto
sudo dnf install jenkins
sudo systemctl daemon-reload
sudo systemctl restart jenkins.service
```
* References
  * [Jenkins Doc](https://www.jenkins.io/doc/book/installing/linux/#fedora)
