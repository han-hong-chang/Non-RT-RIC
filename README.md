# Non-RT-RIC

![image](https://hackmd.io/_uploads/HJNP9Xr5T.png =150x) ＠NTUST, Taiwan

# 【H Release】Installation Guide of Non-RT RIC
###### `tags: 【RIC-platform note】`
:::info
- Installation info
  1. Hardware requests：
      - CPU： 6 core
      - RAM： 12 GB 
      - Disk： 32 GB
  2. Installation Environment：
      - BMWlab intel server
      - Host：192.168.8.117
      - User：non-rt-ric
  3. Version
      - Ubuntu: 20.04.6 LTS
      - Kernal:GNU/Linux 5.15.0-92-generic x86_64
      - kubelet, kubeadm, kubectl = 1.21.1
      - containerd 1.7.2
  4. Reference
      - [Non-RT RIC](https://emphasized-eater-a0b.notion.site/Non-RT-RIC-ed9764a725754775a1a9faa4bd23d813)
      - [【從題目中學習k8s】-【Day3】建立K8s Cluster環境-以kubeadm為例](https://ithelp.ithome.com.tw/articles/10235069)
      - [[Day5] Kubernetes & CRI (Container Runtime Interface)(I)](https://ithelp.ithome.com.tw/articles/10218127)
      - [[Day3] 淺談 Container 實現原理, 探討 OCI 實作](https://ithelp.ithome.com.tw/articles/10216880)
      - [ORAN wiki Near-RT RIC Release H - Run in Kubernetes](https://wiki.o-ran-sc.org/display/RICNR/Release+H+-+Run+in+Kubernetes)
:::

## 1. Install the Containerd, Kubernetes and Helm 3
### 1.1 Entry root and Update Dependent Tools
```shell=
sudo su
sudo apt-get update
sudo apt-get install -y apt-transport-https curl
```
![image](https://hackmd.io/_uploads/S1ZwYmr9a.png )
### 1.2 Forward IPv4 and letting iptables see bridged traffic
```shell=
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```
![image](https://hackmd.io/_uploads/ByeB3Qr5a.png)

### 1.3 Swap off
```shell=
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
![image](https://hackmd.io/_uploads/Bksi2mB9a.png)

### 1.4 Install a Container Runtime Interface-Containerd
- Containerd is a container runtime that manages the lifecycle of a container on a physical or virtual machine (a host). It is a daemon process that creates, starts, stops, and destroys containers.
```shell=
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
```
![image](https://hackmd.io/_uploads/rJz1Rf856.png)

![image](https://hackmd.io/_uploads/BJ9yAfLqa.png)
![image](https://hackmd.io/_uploads/rJKxCMU5p.png)
### 1.5 Install & Run k8s
#### 1.5.1 Install k8s
```shell=
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

sudo apt-get update
sudo apt-get install -y kubelet=1.21.1-00 kubeadm=1.21.1-00 kubectl=1.21.1-00
sudo apt-mark hold kubelet kubeadm kubectl
```
![image](https://hackmd.io/_uploads/SJuiRzUqT.png)

![image](https://hackmd.io/_uploads/B1XWkQL96.png)

![image](https://hackmd.io/_uploads/ry-X17L56.png)
#### 1.5.2 Run k8s
```shell=
#Initialize the cluster
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Get the Token for the new adding Node of k8s which missing Token
kubeadm token list
```
![image](https://hackmd.io/_uploads/r1tg-YLqp.png)

![image](https://hackmd.io/_uploads/rk3GZFLqa.png)

![image](https://hackmd.io/_uploads/HkWEWYL96.png)
### 1.6 Deploy CNI
```shell=
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubectl taint nodes --all node-role.kubernetes.io/master-
```
![image](https://hackmd.io/_uploads/SkIzGFL5T.png)

### 1.7 Install helm3
```shell=
wget https://get.helm.sh/helm-v3.8.2-linux-amd64.tar.gz
tar zxvf helm-v3.8.2-linux-amd64.tar.gz
sudo mv ./linux-amd64/helm /usr/local/bin/helm
rm -rf linux-amd64/
```
![image](https://hackmd.io/_uploads/H1mtGFLqa.png)
### 1.8 Install Chartsmuseum
```shell=
apt-get install -y git
git clone "https://gerrit.o-ran-sc.org/r/it/dep"
./dep/smo-install/scripts/layer-0/0-setup-charts-museum.sh
./dep/smo-install/scripts/layer-0/0-setup-helm3.sh
```
![image](https://hackmd.io/_uploads/rJGe7KIcT.png)

![image](https://hackmd.io/_uploads/ByOL7FUc6.png)

![image](https://hackmd.io/_uploads/r1bF7KU9T.png)

![image](https://hackmd.io/_uploads/HyB9mtI9a.png)

### 1.9 buildkit and nerdctl
```shell=
wget https://github.com/moby/buildkit/releases/download/v0.11.3/buildkit-v0.11.3.linux-amd64.tar.gz
tar xf buildkit-v0.11.3.linux-amd64.tar.gz
cp bin/buildkitd bin/buildctl /usr/local/bin/

cat >> /lib/systemd/system/buildkitd.socket << EOF
[Unit]
Description=BuildKit
Documentation=https://github.com/moby/buildkit

[Socket]
ListenStream=%t/buildkit/buildkitd.sock

[Install]
WantedBy=sockets.target
EOF

cat >> /lib/systemd/system/buildkitd.service << EOF
[Unit]
Description=BuildKit
Requires=buildkitd.socket
After=buildkit.socket
Documentation=https://github.com/moby/buildkit

[Service]
ExecStart=/usr/local/bin/buildkitd --oci-worker=false --containerd-worker=true

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now buildkitd
systemctl status buildkitd
```
==Press `Control-C`==
```shell=
wget https://github.com/containerd/nerdctl/releases/download/v1.2.0/nerdctl-1.2.0-linux-amd64.tar.gz
tar xf nerdctl-1.2.0-linux-amd64.tar.gz
mv nerdctl /usr/local/bin/
```
![image](https://hackmd.io/_uploads/rk7BEYUqp.png)

![image](https://hackmd.io/_uploads/B19UVYL96.png)

![image](https://hackmd.io/_uploads/r1d9NF89a.png)

![image](https://hackmd.io/_uploads/r1BaVFIqp.png)

![image](https://hackmd.io/_uploads/BJh0VYIcT.png)

## 2. Install Non-RT RIC
### 2.1 Import CAPIF Image
- move the file `capif_1_2_5.tar` to `/root`
```shell=
nerdctl load -i capif_1_2_5.tar --namespace k8s.io
nerdctl -n k8s.io image ls
```
![image](https://hackmd.io/_uploads/r1myKaDqT.png)

![image](https://hackmd.io/_uploads/HJi1Kawqp.png)
### 2.2 Revise the Components Configuration
```shell=
apt-get install -y vim
vim dep/RECIPE_EXAMPLE/NONRTRIC/example_recipe.yaml
```
![image](https://hackmd.io/_uploads/S1lIW6P5p.png)
#### 2.2.1 Configuration of components to install
==Press `i` to edit==
```shell=
nonrtric:
  installPms: true
  installA1controller: false
  installA1simulator: false
  installControlpanel: true
  installInformationservice: true
  installRappcatalogueservice: true
  installRappcatalogueenhancedservice: true
  installNonrtricgateway: false
  installKong: false
  installDmaapadapterservice: true
  installDmaapmediatorservice: true
  installHelmmanager: true
  installOrufhrecovery: true
  installRansliceassurance: true
  installCapifcore: true
  installRanpm: false
  installrAppmanager: true
  installDmeParticipant: true
```
![image](https://hackmd.io/_uploads/B1ip-av9p.png)
#### 2.2.2 capifcore Setting
```shell=
capifcore:
  capifcore:
    imagePullPolicy: IfNotPresent
    image:
      registry: "o-ran-sc"
      name: nonrtric-plt-capifcore
      tag: 1.2.5
    env:
      chart_museum_url: "http://chartmuseum:8080"
      repo_name: "capifcore"
```
![image](https://hackmd.io/_uploads/ryYs9aD5a.png)

==Press `ESC` & Enter `wq` to store the modifying==
### 2.3 Revise CAPIF Configuration
#### 2.3.1 values file
```shell=
vim dep/nonrtric/helm/capifcore/values.yaml
```
```shell=
capifcore:
  imagePullPolicy: IfNotPresent
  image:
    registry: 'o-ran-sc'
    name: nonrtric-plt-capifcore
    tag: 1.2.5
  service:
    httpName: http
    allowHttp: true
    internalPort: 8090
    targetPort: 8090
    enternalPort: 30094
  env:
    chart_museum_url: "http://chartmuseum:8080"
    repo_name: "capifcore"

```
![image](https://hackmd.io/_uploads/ryDS9aw9a.png)
#### 2.3.2 service file
```shell=
vim dep/nonrtric/helm/capifcore/templates/service.yaml
```
```shell=
kind: Service
apiVersion: v1
metadata:
  name: {{ include "common.name.capifcore" . }}
  namespace: {{ include "common.namespace.nonrtric" . }}
  labels:
    app: {{ include "common.namespace.nonrtric" . }}-{{ include "common.name.capifcore" . }}
    chart: {{ .Chart.Name }}-{{ .Chart.Version | replace "+" "_" }}
    release: {{ .Release.Name }}
    heritage: {{ .Release.Service }}
spec:
  ports:
    {{if eq .Values.capifcore.service.allowHttp true -}}
    - name: {{ index .Values.capifcore.service.httpName }}
      port: {{ .Values.capifcore.service.internalPort  }}
      targetPort: {{ .Values.capifcore.service.targetPort }}
      nodePort: {{ .Values.capifcore.service.enternalPort }}
      protocol: TCP
    {{- end }}
  selector:
    app: {{ include "common.namespace.nonrtric" . }}-{{ include "common.name.capifcore" . }}
    release: {{ .Release.Name }}
  type: NodePort
```
![image](https://hackmd.io/_uploads/Bkd-3Tvq6.png)

### 2.4 Revise Controlpanel Configuration
==Press `i` to edit==
```shell=
vim dep/nonrtric/helm/controlpanel/resources-ngw/nginx.conf
```
```shell=
events{}

http {
    include /etc/nginx/mime.types;

    upstream backend {
        server  capifcore:8090;
    }

    server {
        listen 8080;
        server_name localhost;
        root /usr/share/nginx/html;
        index index.html;
        location /a1-policy/ {
            proxy_pass  http://backend;
        }
         location /data-producer/ {
            proxy_pass  http://backend;
        }
        location /data-consumer/ {
            proxy_pass  http://backend;
        }
        location / {
            try_files $uri $uri/ /index.html;
        }
    }
}
```
![image](https://hackmd.io/_uploads/SyaKrawcp.png)

==Press `ESC` & Enter `wq` to store the modifying==

### 2.5 Installation the pod of Non-RT RIC
```shell=
sudo dep/bin/deploy-nonrtric -f dep/RECIPE_EXAMPLE/NONRTRIC/example_recipe.yaml
kubectl get pods -A
```
![image](https://hackmd.io/_uploads/SyuBhTP56.png)

![image](https://hackmd.io/_uploads/ByifGAD5T.png)
- Results similar to the output shown below indicate a complete and successful deployment, all are either **“Completed”** or **“Running”**, and that none are **“Error”**, **“Pending”**, **“Evicted”**,or **“ImagePullBackOff”**.
- The status of pods “**PodInitializing”** & **“Init”** & **“ContainerCreating”** mean that the pods are creating now, you need to wait for deploying.

## 3. Test
### 3.1
