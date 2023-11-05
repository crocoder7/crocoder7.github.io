---
title: "Minikube nginx ingress local setup"

categories: 
  - k8s
  - minikube
  - DevOps
last_modified_at: now
---
# Minikube nginx ingress on local mac
I was using minikube on mac trying to local test.<br>
I faced problem accessing local ingress after I set up. My configuration of local ingress was like below
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-service
                port:
                  number: 8080
```
I kept trying to access through minikube ip, which was 192.168.49.2 in my case, but couldn't get any response. <br>
The solution for me was open up a new terminal and run `minikube tunnel` command. <br>
After I ran the command, I was able to get response from localhost on the browser. <br>
<br>
What the command does is that it makes network connection between localhost and minikube. It is just like the service object needs port forwarding to access due to local environment. <br>
<br>
If you want to set up domain names in your ingress configuration for test, descriptions are below
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  rules:
    - host: local.minikube.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-service
                port:
                  number: 8080
```
After using configuration like above, the domain should be added to the host file. <br>
In my case, the host file was located at `/private/etc/hosts`. <br>
Insert a new row of `127.0.0.1 local.minikube.com` and save it.

* References
  * [Stackoverflow](https://stackoverflow.com/questions/58561682/minikube-with-ingress-example-not-working)
  * [Minikube Doc](https://minikube.sigs.k8s.io/docs/handbook/accessing/#:~:text=minikube%20tunnel%20runs%20as%20a,on%20the%20host%20operating%20system.)
