---
title: Kubernetes
date: 2022-06-05 12:30:00 
categories: [Operating Systems]
tags: [servers]
---
# Installation
This is my setup for creating a 4 node kubernetes cluster.

## Preparation
Install a 64 bit Rasbian and boot the Raspberry Pi and as root do the following
```bash
sudo iptables -F 
sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
sudo update-alternatives --set iptables /usr/sbin/iptables-legacy 
reboot
```
## Master
On the master node install kubernetes and extract the node token
```bash
curl -sfL https://get.k3s.io | K3S_KUBECONFIG_MODE="644" sh -s -
cat /var/lib/rancher/k3s/server/node-token
```
## Worker
On each worker nodes install kubernetes replace master-ip with the IP address of the master node, node-name with the name of the worker node and master token with the token in the previous step 
```bash
curl -sfL https://get.k3s.io | K3S_TOKEN="master-token" K3S_URL="https://master-ip:6443" K3S_NODE_NAME="node-name" sh -
```


```
      Rancher
        Install Rancher see docker stuff or ...
          mkdir -p /etc/rancher/rke2
          cd /etc/rncher/rke2
          vi config.yaml
            #token: mylittlepony
            #tls-san:
            #  - 192.168.1.34
          curl -sfL https://get.rancher.io | sh -
          rancherd --help
          systemctl enable rancherd-server.service
          systemctl start rancherd-server.service
          journalctl -eu rancherd-server -f
          rancherd reset-admin  # It might take some time for this to work
        Use the rancher console
          add cluster - type other - give it a name and continue
          Copy the line at the bottom of the screen and paste it into the master node - try until if works
          Back in rancher select the three dots and view api and select edit
          rancher/rancher-agent:v2.5.8-linux-arm64
    Kubernetes Stuff
      minikube & kubectl https://docs.gitlab.com/charts/development/minikube/
        install https://kubernetes.io/docs/home/
          sudo apt-get update
          sudo apt-get install apt-transport-https
          sudo apt-get upgrade
          wget https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
          chmod +x minikube-linux-amd64
          sudo mv minikube-linux-amd64 /usr/local/bin/minikube
          minikube version
          curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
          chmod +x ./kubectl
          sudo mv ./kubectl /usr/local/bin/kubectl
          kubectl version -o json
        Test
          kubectl create deployment hello-k8s --image=geerlingguy/kube101:intro
          kubectl expose deployment hello-k8s --type=NodePort --port=280
          kubectl get services hello-k8s
          minikube ip
          # Use ip from minikube + port from get services or ...
          minikube service hello-k8s  # Will launch broswer
          minikube delete
        Create Secrets to use github
          In github create an access token - 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
          Either: This method needs a secret per registry
            kubectl create secret docker-registry regcred --docker-server=docker.pkg.github.com --docker-username=crgrimwade --docker-password=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
          Or: This method can use the same secret for mulitple regisry - login to each registry before copying the json file 
            minikube ssh # We need to login to github from withing minikube and creae the .docker/config.json file
            echo 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 | docker login docker.pkg.github.com -u crgrimwade --password-stdin
            exit
            scp -i $(minikube ssh-key) docker@$(minikube ip):.docker/config.json config.json # get the config.json files
            kubectl create secret generic regcred --from-file=.dockerconfigjson=config.json --type=kubernetes.io/dockerconfigjson     
          kubectl get secret
          kubectl get secret -o yaml
        Useful commands
          minikube start
          kubectl cluster-info
          kubectl config view
          kubectl get nodes
          minikube ssh
          minikube stop
          minikube delete
          minikube addons list
          minikube dashboard --url
          minikube delete
          eval $(minikube docker-env)  # Will target docker commands to use minikube instead of the host docker    
      go example
        sudo snap install go
        example go app
          create dir - mkdir -p cmd/hello
          initialise go module:  go mod init cmd/hello
          vi cmd/hello/hello.go
            #package main
            #
            #import (
            #  "fmt"
            #  "log"
            #  "net/http"
            #)
            #
            #func HelloServer(w http.ResponseWriter, r *http.Request) {
            #  fmt.Fprintf(w, "Hello, you requested %s", r.URL.Path)
            #  log.Printf("Received request for path: %s", r.URL.Path)
            #}
            #
            #func main() {
            #  var addr string = ":8180"
            #  handler := http.HandlerFunc(HelloServer)
            #  log.Printf("Starting webserver on %s", addr)
            #  if err := http.ListenAndServe(addr, handler); err != nil {
            #    log.Fatalf("Could not listen on port %s %v", addr, err)
            #  }
            #}
          vi cmd/hello/hello_test.go
            #package main
            #
            #import (
            #  "net/http"
            #  "net/http/httptest"
            #  "testing"
            #)
            #
            #func TestGetHello(t *testing.T) {
            #  t.Run("Returns current path", func(t *testing.T) {
            #    request, _ := http.NewRequest(http.MethodGet, "/testing", nil)
            #    response := httptest.NewRecorder()
            #
            #    HelloServer(response, request)
            #
            #    got := response.Body.String()
            #    want := "Hello, you requested: /testing"
            #
            #    if got != want {
            #      t.Errorf("got %q, want %q", got, want)
            #    }
            #  })
            #}
          cd cmd/hello ; go run . or go run cmd/hello/hello.go
          cd cmd/hello ; go test
          vi dockerfile
            #FROM golang:1-alpine as build
            #WORKDIR /app
            #COPY cmd cmd
            #RUN go build cmd/hello/hello.go
            #
            #FROM alpine:latest
            #WORKDIR /app
            #COPY --from=build /app/hello /app/hello
            #EXPOSE 8180
            #ENTRYPOINT ["./hello"]
          docker build -t crgrimwade/hello .
          docker images
      using github
        authenticate
          create github access token - e.g. ghp_zFmWmp1pDYCGKCMeq4er3DzkM9Vaya2E2gmq
          export CR_PAT=ghp_zFmWmp1pDYCGKCMeq4er3DzkM9Vaya2E2gmq
          login to create .docker/config.json # Login in to all git providers
          echo $CR_PAT | docker login docker.pkg.github.com -u crgrimwade --password-stdin
          echo $CR_PAT | docker login ghcr.io -u crgrimwade --password-stdin
        push
          # As a generic image
          docker tag hello ghcr.io/crgrimwade/hello/hello:v1
          docker push ghcr.io/crgrimwade/hello/hello:v1
          docker rmi ghcr.io/crgrimwade/hello/hello:v1
          # As a docker image
          docker tag hello docker.pkg.github.com/crgrimwade/hello/hello:v
          docker push docker.pkg.github.com/crgrimwade/hello/hello:v1
          docker rmi docker.pkg.github.com/crgrimwade/hello/hello:v1
        pull
          Either - use command lines ....
            kubectl create deployment hello-go --image=docker.pkg.github.com/crgrimwade/hello/hello:v1
            # This will fail for a private registry due to no authorisation use below to examine cause
            watch kubectl get deployment hello-go
            kubectl get pod -l app=hello-go
            kubectl describe pod -l app=hello-go
            # To fix
            kubectl edit deployment hello-go # under template: spec: add the following
            #spec: #After this tag
            #  imagePullSecrets:   # Add these tags
            #  -name: regcred
            #  containers: # Before this tag
            Save and exit - the pod will the load
          Or -using a deployment yaml
            vi hello-go-dep.yaml
            #apiVersion: apps/v1
            #kind: Deployment
            #metadata:
            #  name: hello-go-deployment
            #spec:
            #  selector:
            #    matchLabels:
            #      app: hello-go
            #  replicas: 1
            #  template:
            #    metadata:
            #      labels:
            #        app: hello-go
            #    spec: 
            #      imagePullSecrets:
            #      -name: regcred
            #      containers:
            #      -name: hello-go
            #       image: docker.pkg.github.com/crgrimwade/hello/hello:v1
            #  
            kubectl apply -f hello-go-dep.yaml      
        upgrade/downgrade versions
          kubectl set image deployment/hello-go hello-go=docker.pkg...:v2
          kubectl rollout history deployment hello-go
          kubectl rollout undo deployment hello-go
        x
          kubectl logs -f app=hello-go
 ```