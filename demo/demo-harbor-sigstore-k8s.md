# CNCF Bulgaria meetup - Demo - Harbor, Sigstore, K8s

## 0. Pre-requisites

### 0.1 Create Virtual Machine

Lets clone the repo with all the necessary files:

```bash
~$ git clone https://github.com/vhristev/cncf-bulgaria-meetup-2024-feb.git
Cloning into 'cncf-bulgaria-meetup-2024-feb'...
```

### 0.2 Create Virtual Machine

For this lecture we will use Linux Ubuntu 22.04 Virtual Machines. Keep in mind
this is not the focus of the lecture and we will not go into details.

I will use `vagrat` to bring it up:

```bash
~$ vagrant up
Bringing machine 'default' up with 'vmware_desktop' provider...
==> default: Cloning VMware VM: 'hajowieland/ubuntu-jammy-arm'. This can take some time...
==> default: Checking if box 'hajowieland/ubuntu-jammy-arm' version '1.0.0' is up to date...
==> default: Verifying vmnet devices are healthy...
==> default: Preparing network adapters...
==> default: Starting the VMware VM...
==> default: Waiting for the VM to receive an address...
==> default: Forwarding ports...
    default: -- 22 => 2222
==> default: Waiting for machine to boot. This may take a few minutes...
    default: SSH address: 192.168.59.142:22
    default: SSH username: vagrant
SNIP ....
SNIP ....
==> default: Enabling and configuring shared folders...
    default: -- /Users/vhristev/Documents/gitlab_home/vagrant/rxm-labvm: /vagrant

```

### Install Kubernetes

We will use a script to install Kubernetes because its not the focus of our lecture. The script will also install
`Docker` which we will use to build our images.

```
~$ wget -qO - https://raw.githubusercontent.com/RX-M/classfiles/master/k8s.sh | sh
```

Lets add our user to the docker group so we don't use `sudo` all the time.

```bash
~$ sudo usermod -aG docker $(whoami)
~$ newgrp docker
```

If you like auto-completion, don't forget this which will do two things.

- Add the completion script to your `.bashrc` file
- Add an alias for `kubectl` to `k` ( optional ). If you don't want to use 'k'
    just use the full command `kubectl`.

```
echo "source <(kubectl completion bash)" >> ~/.bashrc
echo "alias k=kubectl" >> ~/.bashrc
echo "complete -F __start_kubectl k" >> ~/.bashrc
source ~/.bashrc
```

Lets verify the installation:

```bash
~$ k get nodes
NAME     STATUS   ROLES           AGE     VERSION
ubuntu   Ready    control-plane   3m49s   v1.29.2

```

Everything looks good lets continue

### Install Cosign

We will use it to sign our images.

```bash
# dkpg
ARCH=$(dpkg --print-architecture)
LATEST_VERSION=$(curl https://api.github.com/repos/sigstore/cosign/releases/latest | grep tag_name | cut -d : -f2 | tr -d "v\", ")
curl -O -L "https://github.com/sigstore/cosign/releases/latest/download/cosign_${LATEST_VERSION}_${ARCH}.deb"
sudo dpkg -i cosign_${LATEST_VERSION}_${ARCH}.deb
```


### Demo Harbor account

We will use the official https://demo.goharbor.io/ from [official doc](https://goharbor.io/docs/1.10/install-config/demo-server/) you can find that __The demo server is cleaned and reset every two days.__

Lets create an account

https://demo.goharbor.io/

Click on Sign Up and fill the form there is no need of verification. Just use the user and the password you have entered.

Create a new project and name it whatever you want. We will name ours `yoda`

#### Login to the Harbor

```bash
~$ docker login demo.goharbor.io
```

#### Create an image

Build an image from provided Dockerfile and tag it.

```
~$ docker build -t demo.goharbor.io/yoda/moonlight:1.0.0 .
```

#### Push it to the Harbor

```
~$ docker push demo.goharbor.io/yoda/moonlight:1.0.0

The push refers to repository [demo.goharbor.io/yoda/moonlight]
b09314aec293: Pushed
3.0.0: digest: sha256:eb6b7c17ba2ece109f5548e95a90a0b104bc44aeb1f8b30b9eafa2e5de1c420c size: 527
```
#### Sign the image

To sign our image using `cosign` we will need to generate a key pair first. If we choose to use `openID` as a method of
authentication later when we configure kyverno we will need additional service handling that. For that reason we will
keep it simple and use long lived ( not recommended ) key pair.

Lets generate a key pair:

```bash
~$ cosign generate-key-pair

Enter password for private key:
Enter password for private key again:
Private key written to cosign.key
Public key written to cosign.pub
```

Get the image digest

```bash
~$ DIGESTS=`docker images --digests | grep 1.0.0 | awk '{print $3}'`
~$ echo $DIGESTS
sha256:eb6b7c17ba2ece109f5548e95a90a0b104bc44aeb1f8b30b9eafa2e5de1c420c
```
Sign the image

```bash
~$ cosign sign --key cosign.key demo.goharbor.io/yoda/moonlight@${DIGESTS}

WARNING: "demo.goharbor.io/yoda/moonlight" appears to be a private repository, please confirm uploading to the transparency log at "https://rekor.sigstore.dev"
Are you sure you would like to continue? [y/N] y
...

Are you sure you would like to continue? [y/N] y
tlog entry created with index: 72407934
Pushing signature to: demo.goharbor.io/yoda/moonlight

```

If you wan to check our your signing we can check `Rekor` at [Search Rekor](http://search.sigstore.dev). Now you can use
your email or you can get the `tlog` index from the output of the `cosign sign` command.


Lets build an image with vulnerable software. We will use Dockerfile-vuln

```bash
~$ docker build -f Dockerfile-vuln -t demo.goharbor.io/yoda/moonlight-vuln:2.0.0 .
```


Push the images to your project in Harbor.

```
~$ docker push demo.goharbor.io/yoda/moonlight:1.0.0
~$ docker push demo.goharbor.io/yoda/moonlight-vuln:2.0.0



```

### Install helm

Helm is a package manager for Kubernetes, so it can install and manage software on your cluster.

```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```



### Use Harbor registry

```bash
kubectl create secret docker-registry harbor-credentials \
--docker-server=demo.goharbor.io \
--docker-username=cncfdemo \
--docker-password='HARBOR_ACCOUNT' \
--docker-email=your_email@example.com
```


### Kyverno

A dynamic admission controller for Kubernetes. We will use it to enforce policies.

```bash
~$ helm repo add kyverno https://kyverno.github.io/kyverno/
~$ helm install kyverno kyverno/kyverno -n kyverno --create-namespace
```

Lets check the installation:

```bash
~$ k get pods -n kyverno
NAME                                             READY   STATUS    RESTARTS   AGE
kyverno-admission-controller-69f8479f84-j9rnr    1/1     Running   0          48s
kyverno-background-controller-7b6c6c8574-d26tr   1/1     Running   0          48s
kyverno-cleanup-controller-754bc5976f-qrzxs      1/1     Running   0          48s
kyverno-reports-controller-66d88fc55b-k86h7      1/1     Running   0          48s
```

Good, everything looks good.

### Cosign pub key

Lets create a ConfigMap in Kubernetes that contains our Cosign public key. This ConfigMap will be referenced by the Kyverno policy to verify the signatures of the images. This is the key pair we create earlier.

```bash
~$ kubectl create configmap cosign-public-key --from-file=cosign.pub=./cosign.pub -n kyverno


```

Verify the ConfigMap

```bash
~$ k describe cm -n kyverno cosign-public-key
Name:         cosign-public-key
Namespace:    kyverno
Labels:       <none>
Annotations:  <none>

Data
====
cosign.pub:
----
-----BEGIN PUBLIC KEY-----
MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE5VHpHrsyWa9Rq+IY+L+IwyrVyhZK
8+iZFrJOZ4j0XlJbsu4PlANsiqM8wiIjtS7ZX5+055o9Y9lHhWIFh33BFA==
-----END PUBLIC KEY-----


BinaryData
====

Events:  <none>
```



### Kyberno policies

We need to create a that will check if images are scanned

Lets create our working directory

```bash
~$ mkdir ~/kuverno && cd $_
```



```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signature
spec:
  validationFailureAction: Enforce
  background: false
  rules:
    - name: check-image-signature
      match:
        any:
        - resources:
            kinds:
              - Pod
      verifyImages:
      - image: "demo.goharbor.io/yoda/*" # yoda is our project name
        key: |-
            name: cosign-public-key # This is the configmap we created earlier
            namespace: kyverno
            data: cosign.pub # This is the cosign pub key
```

Create the policy

```bash
~$ k apply -f kp.yaml
clusterpolicy.kyverno.io/verify-image-signature created
```

Verify the policy

```bash
~$ k get clusterpolicy -n kyverno
NAME                     ADMISSION   BACKGROUND   VALIDATE ACTION   READY   AGE   MESSAGE
verify-image-signature   true        false        Enforce           True    34s   Ready
```

You can use `kubectl describe` for full details.

```
~$ k describe clusterpolicy -n kyverno
Name:         verify-image-signature
Namespace:
Labels:       <none>
Annotations:  <none>
API Version:  kyverno.io/v1
Kind:         ClusterPolicy
Metadata:
  Creation Timestamp:  2024-02-19T13:43:47Z
.
.
.

```

Lets create a pod that will violate our policy

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: pod-not-signed
  name: pod-not-signed
spec:
  containers:
  - image: demo.goharbor.io/yoda/moonlight-not-signed:2.0.0
    name: pod-not-signed
