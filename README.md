# k3d ingress nginx-demo cluster

Prepared this as pat of blog post series on production ready kubernetes.

## Quickstart -> run all

```sh
task all
```

## Prerequisites

- docker
- k3d
- helm
- go-task

## Setup Includes

By running `task create-cluster` will

- create a k3d based kubernetes cluster
- install ingress-nginx
- deploy ssl certs for `whoami.k8s.localhost`
  - `whoami.k8s.localhost` + other base suffixes added to your hostsfile
- install reflector and reloader which help manage stuff like ssl-replication and configuration reload

```sh
task all
```

would yield:

```sh
task all
task: [k3d:hosts-file] test -f ./.config/.etchosts || touch ./.config/.etchosts
task: [k3d:hosts-file] cat<<EOF>./.config/.etchosts
127.0.0.1 cluster.localhost
127.0.0.1 k8s.localhost
127.0.0.1 demo.k8s.localhost
127.0.0.1 app.k8s.localhost
127.0.0.1 whoami.k8s.localhost
127.0.0.1 example.k8s.localhost
127.0.0.1 chartmuseum.k8s.localhost
EOF

task: [k3d:hosts-file] grep -qF "127.0.0.1 ingress-demo.k8s.localhost" ./.config/.etchosts || echo "127.0.0.1 ingress-demo.k8s.localhost" | sudo tee -a ./.config/.etchosts
Password:
127.0.0.1 ingress-demo.k8s.localhost
task: [k3d:cluster-template] cat<<EOF>.config/k3d-ingress-demo.yaml
#
# Twinker with the template at your own risk :)
#
apiVersion: k3d.io/v1alpha5
kind: Simple
metadata:
  name: $CLUSTER_NAME
servers: 1
agents: 2
kubeAPI:
  host: "k8s.localhost"
  hostIP: "127.0.0.1"
  hostPort: "6445"
image: rancher/k3s:v1.29.4-k3s1
network: $CLUSTER_NAME-net
ports:
- port: 80:80
  nodeFilters:
  - loadbalancer
- port: 443:443
  nodeFilters:
  - loadbalancer
registries:
  create:
    name: registry.localhost
    host: "0.0.0.0"
    hostPort: "5002"
  config: |
    mirrors:
      "registry.localhost":
        endpoint:
          - http://registry.localhost:5002

volumes:
- volume: "${PWD}/storage:/var/lib/rancher/k3s/storage"
  nodeFilters:
  - server:0
  - agent:*

hostAliases:
- ip: 127.0.0.1
  hostnames:
  - k8s.localhost
  - app.k8s.localhost
  - demo.k8s.localhost
  - app.k8s.localhost
  - example.k8s.localhost
  - whoami.k8s.localhost
  - $CLUSTER_NAME.k8s.localhost
options:
  k3d:
    wait: true
    timeout: "360s"
    loadbalancer:
      configOverrides:
      - settings.workerConnections=2048
  k3s:
    extraArgs:
    - arg: --disable=traefik
      nodeFilters:
      - server:*
    - arg: --tls-san=localhost,127.0.0.1,registry.localhost,k8s.localhost,cluster.localhost,k8s.localhost,demo.k8s.localhost,app.k8s.localhost,example.k8s.localhost,whoami.k8s.localhost,$CLUSTER_NAME.k8s.localhost
      nodeFilters:
      - server:*
    - arg: --kube-proxy-arg=metrics-bind-address=0.0.0.0
      nodeFilters:
      - server:*
      - agent:*
    - arg: --kube-scheduler-arg=bind-address=0.0.0.0
      nodeFilters:
      - server:*
    - arg: --kubelet-arg=node-status-update-frequency=4s
      nodeFilters:
      - server:*
  kubeconfig:
    updateDefaultKubeconfig: true
    switchCurrentContext: true
EOF

task: [k3d:cluster-create] k3d cluster list | grep 'ingress-demo' || k3d cluster create --config .config/k3d-$CLUSTER_NAME.yaml
INFO[0000] Using config file .config/k3d-ingress-demo.yaml (k3d.io/v1alpha5#simple) 
INFO[0000] portmapping '80:80' targets the loadbalancer: defaulting to [servers:*:proxy agents:*:proxy] 
INFO[0000] portmapping '443:443' targets the loadbalancer: defaulting to [servers:*:proxy agents:*:proxy] 
INFO[0000] Prep: Network                                
INFO[0000] Created network 'ingress-demo-net'           
INFO[0000] Created image volume k3d-ingress-demo-images 
INFO[0000] Creating node 'registry.localhost'           
INFO[0000] Successfully created registry 'registry.localhost' 
INFO[0000] Starting new tools node...                   
INFO[0000] Starting node 'k3d-ingress-demo-tools'       
INFO[0001] Creating node 'k3d-ingress-demo-server-0'    
INFO[0001] Creating node 'k3d-ingress-demo-agent-0'     
INFO[0001] Creating node 'k3d-ingress-demo-agent-1'     
INFO[0001] Creating LoadBalancer 'k3d-ingress-demo-serverlb' 
INFO[0001] Using the k3d-tools node to gather environment information 
INFO[0001] Starting new tools node...                   
INFO[0001] Starting node 'k3d-ingress-demo-tools'       
INFO[0002] Starting cluster 'ingress-demo'              
INFO[0002] Starting servers...                          
INFO[0002] Starting node 'k3d-ingress-demo-server-0'    
INFO[0005] Starting agents...                           
INFO[0005] Starting node 'k3d-ingress-demo-agent-1'     
INFO[0005] Starting node 'k3d-ingress-demo-agent-0'     
INFO[0008] Starting helpers...                          
INFO[0008] Starting node 'registry.localhost'           
INFO[0008] Starting node 'k3d-ingress-demo-serverlb'    
INFO[0014] Injecting records for hostAliases (incl. host.k3d.internal) and for 6 network members into CoreDNS configmap... 
INFO[0018] Cluster 'ingress-demo' created successfully! 
INFO[0018] You can now use it like this:                
kubectl cluster-info
task: [k3d:cluster-create] echo -e "\nYour cluster has been created. Type 'k3d cluster list' to confirm."

Your cluster has been created. Type 'k3d cluster list' to confirm.
task: [k3d:dns] sleep 0.01 && sudo hostctl add k8s -q < .config/.etchosts
task: [k3d:dns] echo -e "Added 'k8s.localhost' and related domains to your hosts file!"
Added 'k8s.localhost' and related domains to your hosts file!
task: [k3d:certs] echo -e "Creating local certificates\n"
Creating local certificates

task: [k3d:certs] rm cert.pem key.pem base/tls-secret.yaml ca.pem 2> /dev/null
task: [k3d:certs] cp -r /Users/hagzag/Projects/post-4-ingress/config/tls/* /Users/hagzag/Projects/post-4-ingress/.config/tls
cp: /Users/hagzag/Projects/post-4-ingress/config/tls/*: No such file or directory
task: [k3d:certs] mkcert -install
The local CA is already installed in the system trust store! 👍
ERROR: no Firefox security databases found

task: [k3d:certs] mkcert -cert-file cert.pem -key-file key.pem -p12-file p12.pem "*.k8s.localhost" k8s.localhost "*.localhost" ::1 127.0.0.1 localhost 127.0.0.1 "*.internal.localhost" "*.local" 2> /dev/null
task: [k3d:certs] echo -e "Creating certificate secrets on Kubernetes for local TLS enabled by default\n"
Creating certificate secrets on Kubernetes for local TLS enabled by default

task: [k3d:certs] kubectl config set-context --current --namespace=kube-system --cluster=k3d-ingress-demo
Context "k3d-ingress-demo" modified.
task: [k3d:certs] kubectl create secret tls tls-secret --cert=cert.pem --key=key.pem --dry-run=client -o yaml >base/tls-secret.yaml
task: [k3d:certs] kubectl apply -k ./
secret/tls-secret created
task: [k3d:certs] echo -e "\nCertificate resources have been created.\n"

Certificate resources have been created.

task: [config:helm:install] helm upgrade --install reloader -f /Users/hagzag/Projects/post-4-ingress/config/apps/reloader/values.yaml -n reloader --create-namespace --wait /Users/hagzag/Projects/post-4-ingress/config/apps/reloader
Release "reloader" does not exist. Installing it now.
NAME: reloader
LAST DEPLOYED: Sun May 12 15:59:58 2024
NAMESPACE: reloader
STATUS: deployed
REVISION: 1
TEST SUITE: None
task: [config:helm:install] helm upgrade --install reflector -f /Users/hagzag/Projects/post-4-ingress/config/apps/reflector/values.yaml -n reflector --create-namespace --wait /Users/hagzag/Projects/post-4-ingress/config/apps/reflector
Release "reflector" does not exist. Installing it now.
NAME: reflector
LAST DEPLOYED: Sun May 12 16:00:42 2024
NAMESPACE: reflector
STATUS: deployed
REVISION: 1
TEST SUITE: None
task: [ingress-nginx:helm:install] helm upgrade --install ingress-nginx -f /Users/hagzag/Projects/post-4-ingress/config/apps/ingress-nginx/values.yaml -n ingress-nginx --create-namespace --wait /Users/hagzag/Projects/post-4-ingress/config/apps/ingress-nginx
Release "ingress-nginx" does not exist. Installing it now.
NAME: ingress-nginx
LAST DEPLOYED: Sun May 12 16:01:04 2024
NAMESPACE: ingress-nginx
STATUS: deployed
REVISION: 1
TEST SUITE: None
task: [config:helm:install] helm upgrade --install whoami -f /Users/hagzag/Projects/post-4-ingress/config/apps/whoami/values.yaml -n whoami --create-namespace --wait /Users/hagzag/Projects/post-4-ingress/config/apps/whoami
Release "whoami" does not exist. Installing it now.
NAME: whoami
LAST DEPLOYED: Sun May 12 16:02:21 2024
NAMESPACE: whoami
STATUS: deployed
REVISION: 1
REVISION: 2
task: [all] curl https://whoami.k8s.localhost
Hostname: whoami-7bb5955dc-z9r7v
IP: 127.0.0.1
IP: ::1
IP: 10.42.1.6
IP: fe80::d888:dbff:fe17:e92e
RemoteAddr: 10.42.2.5:38342
GET / HTTP/1.1
Host: whoami.k8s.localhost
User-Agent: curl/8.4.0
Accept: */*
X-Forwarded-For: 10.42.2.1
X-Forwarded-Host: whoami.k8s.localhost
X-Forwarded-Port: 443
X-Forwarded-Proto: https
X-Forwarded-Scheme: https
X-Real-Ip: 10.42.2.1
X-Request-Id: 9724c7d80a61e636c06e1eb18b7bbaff
X-Scheme: https
```

> 1st run ?
> If you get this:

```html
task: [all] curl https://whoami.k8s.localhost

<html>
<head><title>503 Service Temporarily Unavailable</title></head>
<body>
<center><h1>503 Service Temporarily Unavailable</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

Running `curl https://whoami.k8s.localhost` a few seconds later passes 

```sh
curl https://whoami.k8s.localhost
# hostname may vary based on your replicaset id ...

Hostname: whoami-7bb5955dc-z9r7v
IP: 127.0.0.1
IP: ::1
IP: 10.42.1.6
IP: fe80::d888:dbff:fe17:e92e
RemoteAddr: 10.42.2.5:48252
GET / HTTP/1.1
Host: whoami.k8s.localhost
User-Agent: curl/8.4.0
Accept: */*
X-Forwarded-For: 10.42.2.1
X-Forwarded-Host: whoami.k8s.localhost
X-Forwarded-Port: 443
X-Forwarded-Proto: https
X-Forwarded-Scheme: https
X-Real-Ip: 10.42.2.1
X-Request-Id: 372f998045f1284bb2608bf4af7f9989
X-Scheme: https

```
