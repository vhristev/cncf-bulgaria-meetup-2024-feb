# CNCF Bulgaria meetup - Demo - Harbor, Sigstore, K8s

## 0. Pre-requisites

### 0.1 Create Virtual Machine

Lets clone the repo with all the necessary files:

```bash
~$ git clone https://github.com/vhristev/cncf-bulgaria-meetup-2024-feb.git
Cloning into 'cncf-bulgaria-meetup-2024-feb'...
```

Lets move the demo directory

```bash
~$ mv cncf-bulgaria-meetup-2024-feb/demo/ .
~$ cd demo/
~/demo$
```

Small VIM optimization

```bash
$ echo "set number" >> ~/.vimrc
$ echo "set paste" >> ~/.vimrc
```
```

### 0.2 Create Virtual Machine

For this lecture we will use Linux Ubuntu 22.04 Virtual Machines. Keep in mind
this is not the focus of the lecture and we will not go into details.

We will use Amazon EC2 instance but here is a pure vanilla `Vagrantfile` file that you can use.

```bash
~/ demo$ vagrant up
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
    default: -- /Users/vhristev/Documents/gitlab_home/vagrant/rxm-labvm: /vagran

```

### Install Cosign

We will install `cosign` which is a tool for signing and verifying container images. We will use it to sign our images.

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

Create a new project and name it whatever you want. We will name ours `meetup`

#### Login to the Harbor

We will use our account name `meetup` and password `Meetup12`

```bash
~/demo$ docker login demo.goharbor.io
Username: meetup
Password:
WARNING! Your password will be stored unencrypted in /home/ubuntu/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded

```

#### Create an image

Build an image from repo provided `Dockerfile` with a simple `alpine` container image and tag it. This image is latest
alpine and there is no critical vulnerabilities.

```bash
~/demo$ cat Dockerfile
FROM alpine:3.19.1
CMD sleep "1000"
```
Lets build the image

```bash
~/demo$ docker build -t demo.goharbor.io/meetup/moonlight:1.0.0 .
```

#### Push it to the Harbor

Let's push the image to the Harbor

```bash
~$ docker push demo.goharbor.io/meetup/moonlight:1.0.0

The push refers to repository [demo.goharbor.io/meetup/moonlight]
b09314aec293: Pushed
3.0.0: digest: sha256:eb6b7c17ba2ece109f5548e95a90a0b104bc44aeb1f8b30b9eafa2e5de1c420c size: 527
```
#### Sign the image

To sign our image using `cosign` we will need to generate a key pair first. If we choose to use `openID` as a method of
authentication later when we configure kyverno we will need additional service handling that. For that reason we will
keep it simple and use long lived ( not recommended ) key pair.

Lets generate a key pair:

```bash
~/demo$ cosign generate-key-pair

Enter password for private key:
Enter password for private key again:
Private key written to cosign.key
Public key written to cosign.pub
```

We will get the image digest and store it in a variable for later use.

```bash
~/demo$ DIGESTS=`docker images --digests | grep 1.0.0 | awk '{print $3}'`
~/demo$ echo $DIGESTS
sha256:eb6b7c17ba2ece109f5548e95a90a0b104bc44aeb1f8b30b9eafa2e5de1c420c
```
Sign the image

```bash
~/demo$ cosign sign --key cosign.key demo.goharbor.io/meetup/moonlight@${DIGESTS}

WARNING: "demo.goharbor.io/meetup/moonlight" appears to be a private repository, please confirm uploading to the transparency log at "https://rekor.sigstore.dev"
Are you sure you would like to continue? [y/N] y
...

Are you sure you would like to continue? [y/N] y
tlog entry created with index: 72407934
Pushing signature to: demo.goharbor.io/meetup/moonlight

```

If you wan to check our your signing we can check `Rekor` at [Search Rekor](http://search.sigstore.dev). Now you can use
your email or you can get the `tlog` index from the output of the `cosign sign` command.


Lets build an image with vulnerable software. We will use `Dockerfile-vuln` which use older `alpine` image which is
vulnerable

```bash
~/demo$ docker build -f Dockerfile-vuln -t demo.goharbor.io/meetup/moonlight-vuln:1.0.0 .
```

Push the images to your project in Harbor.

```
~$ docker push demo.goharbor.io/meetup/moonlight-vuln:1.0.0
Pushing image to demo.goharbor.io/meetup/moonlight-vuln:1.0.0
```



### Install Kubernetes

We will use a script to install Kubernetes because its not the focus of our lecture. The script will also install
`Docker` which we will use to build our images.

```
~/demo$ wget -qO - https://raw.githubusercontent.com/RX-M/classfiles/master/k8s.sh | sh
```

Lets add our user to the docker group so we don't use `sudo` all the time.

```bash
~/demo$ sudo usermod -aG docker $(whoami)
~/demo$ newgrp docker
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
~/demo$ k get nodes
NAME     STATUS   ROLES           AGE     VERSION
ubuntu   Ready    control-plane   3m49s   v1.29.2

```

Everything looks good lets continue

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
~/demo$ k create secret docker-registry harbor-credentials \
--docker-server=demo.goharbor.io \
--docker-username=meetup \
--docker-password='Meetup12' \
--docker-email=your_email@example.com
```


### Kyverno

A dynamic admission controller for Kubernetes. We will use it to enforce policies.

```bash
~/demo$ helm repo add kyverno https://kyverno.github.io/kyverno/
~/demo$ helm install kyverno kyverno/kyverno -n kyverno --create-namespace
```

Lets check the installation:

```bash
~/demo$ k get pods -n kyverno
NAME                                             READY   STATUS    RESTARTS   AGE
kyverno-admission-controller-69f8479f84-j9rnr    1/1     Running   0          48s
kyverno-background-controller-7b6c6c8574-d26tr   1/1     Running   0          48s
kyverno-cleanup-controller-754bc5976f-qrzxs      1/1     Running   0          48s
kyverno-reports-controller-66d88fc55b-k86h7      1/1     Running   0          48s
```

Good, everything looks good.

### Kyverno policies

Lets create our first policy. We will use `kyverno` to enforce that all images are signed.
`Kyverno` will use `cosign` public key to verify the signature of the images.

We need to get the `cosign.pub` key which we will add in our policy.

```bash
~/demo$ cat cosign.pub
-----BEGIN PUBLIC KEY-----
MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEsYce/B39gE5focMAnDWf5saXsvzh
lzgDNilMRKqg94/dGc8cYAmSQNa6i2AoQWueXUWSG6+SkdL2nT0NkgH1hw==
-----END PUBLIC KEY-----
```

Create our working directory

```bash
~/demo$ mkdir ~/kyverno && cd $_

```

Here is our policy

```bash
~demo$ vim kp.yaml && cat $_
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
      - image: "demo.goharbor.io/meetup/*" # meetup is our project name
        key: |- # This is our cosign.pub key
          -----BEGIN PUBLIC KEY-----
          MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEsYce/B39gE5focMAnDWf5saXsvzh
          lzgDNilMRKqg94/dGc8cYAmSQNa6i2AoQWueXUWSG6+SkdL2nT0NkgH1hw==
          -----END PUBLIC KEY-----
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

```bash
~/kyverno$ vim pod.yaml && cat $_
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: pod-not-signed
  name: pod-not-signed
spec:
  containers:
  - image: demo.goharbor.io/meetup/moonlight:1.0.0
    name: pod-not-signed
```

Now we can try to create our pod and see what actually happens.

```bash
~/kyverno$ k apply -f pod.yaml
Error from server: error when creating "pod.yaml": admission webhook "mutate.kyverno.svc-fail" denied the request:

resource Pod/default/pod-not-signed was blocked due to the following policies

verify-image-signature:
  check-image-signature: 'failed to verify image demo.goharbor.io/meetup/moonlight-vuln:1.0.0:
    .attestors[0].entries[0].keys: no matching signatures'
```

As you can see our image was not signed and the pod was not created. Lets try with a signed image.

```bash
~/kyverno$ vim pod-signed.yaml && cat $_
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: pod-signed
  name: pod-not-signed
spec:
  containers:
  - image: demo.goharbor.io/meetup/moonlight:2.0.0
    name: pod-signed
```

`moonlight` image is the signed one ,so lets see what is going to happen.

```bash
~/kyverno$ k apply -f pod-signed.yaml
pod/pod-not-signed created
ubuntu@ip-172-31-17-242:~/kyverno$ k get pods
NAME             READY   STATUS    RESTARTS   AGE
pod-signed   1/1     Running   0          5s
```

As you can see the pod was created successfully. That is how easy it is to enforce policies with `kyverno` and `cosign`.
Now we can be sure that our running pods are using only signed images.

## Scanner with trivy


### Install trivy

```bash
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
```

### Kyverno policy for scan check

This is our policy for `kyverno` to check if the image was scanned and the scan is not older than 12 hours.

```bash
~/kyverno$ vim kp-trivy.yaml && cat $_
```
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: check-vulnerabilities-scan
spec:
  validationFailureAction: Enforce
  background: false
  webhookTimeoutSeconds: 30
  failurePolicy: Fail
  rules:
    - name: checking-vulnerability-scan
      match:
        any:
        - resources:
            kinds:
              - Pod
      verifyImages:
      - imageReferences:
        - "*"
        attestations:
        - type: https://cosign.sigstore.dev/attestation/vuln/v1
          conditions:
          - all:
            - key: "{{ time_since('','{{ metadata.scanFinishedOn }}', '') }}"
              operator: LessThanOrEquals
              value: "12h"
          attestors:
          - count: 1
            entries:
            - keys:
                publicKeys: |- # This is our cosign.pub key
                  -----BEGIN PUBLIC KEY-----
                  MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEsYce/B39gE5focMAnDWf5saXsvzh
                  lzgDNilMRKqg94/dGc8cYAmSQNa6i2AoQWueXUWSG6+SkdL2nT0NkgH1hw==
                  -----END PUBLIC KEY-----
```

Test with pod

```bash
~/kyverno$ vim pod-scan.yaml && cat $_
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: pod-scanned
  name: pod-scanned
spec:
  containers:
  - image: demo.goharbor.io/meetup/moonlight:1.0.0
    name: pod-scanned
```

```bash
~/kyverno$ k apply -f pod-scan.yaml
```

and it fails

### Trivy scan image


With `Trivy` we will scan our image:

```bash
~/kyverno$  trivy image --ignore-unfixed --format cosign-vuln --output scan.json demo.goharbor.io/meetup/moonlight:1.0.0
```

- --ignore-unfixed: Ensures only the vulnerabilities, which have a already a fix available, are displayed
- --output scan.json: The scan output is saved to a scan.json file instead of being displayed in the terminal.

### Attack attestation to the image

```bash
~/kyverno$ cosign attest --key cosign.key --predicate scan.json --type vuln demo.goharbor.io/meetup/moonlight:1.0.0
```


Test with a pod

```bash
~/kyverno$ vim pod-scan.yaml && cat $_
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: pod-scanned
  name: pod-scanned
spec:
  containers:
  - image: demo.goharbor.io/meetup/moonlight:1.0.0
    name: pod-scanned
```

As you can see the pod can run now because the image was scanned and the scan is not older than 12 hours.

```bash
:~/kuverno$ k apply -f pod.yaml
pod/pod-not-signed created
ubuntu@ip-172-31-17-242:~/kuverno$ k get pods
NAME             READY   STATUS    RESTARTS        AGE
pod-scanned   1/1     Running   3 (9m58s ago)   60m
```

Thank you.

@Valentin Hristev
@Orlin Vasilev
