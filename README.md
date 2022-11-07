Heavily sourced from:
https://github.com/onedr0p/home-ops
https://github.com/onedr0p/flux-cluster-template



## 👋 Introduction

The following components will be installed in your [k3s](https://k3s.io/) cluster by default. Most are only included to get a minimum viable cluster up and running.

- [flux](https://toolkit.fluxcd.io/) - GitOps operator for managing Kubernetes clusters from a Git repository
- [kube-vip](https://kube-vip.io/) - Load balancer for the Kubernetes control plane nodes
- [metallb](https://metallb.universe.tf/) - Load balancer for Kubernetes services
- [cert-manager](https://cert-manager.io/) - Operator to request SSL certificates and store them as Kubernetes resources
- [calico](https://www.tigera.io/project-calico/) - Container networking interface for inter pod and service networking
- [external-dns](https://github.com/kubernetes-sigs/external-dns) - Operator to publish DNS records to Cloudflare (and other providers) based on Kubernetes ingresses
- [k8s_gateway](https://github.com/ori-edge/k8s_gateway) - DNS resolver that provides local DNS to your Kubernetes ingresses
    - [ingress-nginx](https://kubernetes.github.io/ingress-nginx/) - swapping out ingress-nginx for traefik
- [traefik](https://doc.traefik.io/traefik/providers/kubernetes-crd/) - Kubernetes ingress controller used for a HTTP reverse proxy of Kubernetes ingresses
- [local-path-provisioner](https://github.com/rancher/local-path-provisioner) - provision persistent local storage with Kubernetes

_Additional applications include [hajimari](https://github.com/toboshii/hajimari), [error-pages](https://github.com/tarampampam/error-pages), [echo-server](https://github.com/Ealenn/Echo-Server), [system-upgrade-controller](https://github.com/rancher/system-upgrade-controller), [reloader](https://github.com/stakater/Reloader), and [kured](https://github.com/weaveworks/kured)_

For provisioning the following tools will be used:

- [Fedora 36 Server](https://getfedora.org/en/server/download/) - Universal operating system that supports running all kinds of home related workloads in Kubernetes and has a faster release cycle
- [Ansible](https://www.ansible.com) - Provision Fedora Server and install k3s
- [Terraform](https://www.terraform.io) - Provision an already existing Cloudflare domain and certain DNS records to be used with your k3s cluster

### ⚠️ pre-commit - check into the sops-pre-commit settings to make sure that is correct

It is advisable to install [pre-commit](https://pre-commit.com/) and the pre-commit hooks that come with this repository.
[sops-pre-commit](https://github.com/k8s-at-home/sops-pre-commit) will check to make sure you are not committing non-encrypted Kubernetes secrets to your repository.

1. Enable Pre-Commit

    ```sh
    task precommit:init
    ```

2. Update Pre-Commit, though it will occasionally make mistakes, so verify its results.

    ```sh
    task precommit:update

### 🔐 Setting up Age

📍 Here we will create a Age Private and Public key. Using [SOPS](https://github.com/mozilla/sops) with [Age](https://github.com/FiloSottile/age) allows us to encrypt secrets and use them in Ansible, Terraform and Flux.

1. Create a Age Private / Public Key

    ```sh
    age-keygen -o age.agekey
    ```

2. Set up the directory for the Age key and move the Age file to it

    ```sh
    mkdir -p ~/.config/sops/age
    mv age.agekey ~/.config/sops/age/keys.txt
    ```

3. Export the `SOPS_AGE_KEY_FILE` variable in your `bashrc`, `zshrc` or `config.fish` and source it, e.g.

    ```sh
    export SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt
    source ~/.bashrc
    ```

4. Fill out the Age public key in the `.sops.yaml` under each age key (if using different ones), **note** the public key should start with `age`...

### 📄 Configuration

📍 The `.config.env` file contains necessary configuration that is needed by Ansible, Terraform and Flux.

1. Copy the `.config.sample.env` to `.config.env` and start filling out all the environment variables.

    **All are required** unless otherwise noted in the comments.

    ```sh
    cp .config.sample.env .config.env
    ```

2. Once that is done, verify the configuration is correct by running:

    ```sh
    task verify
    ```

3. If you do not encounter any errors run start having the script wire up the templated files and place them where they need to be.

    ```sh
    task configure
    ```

### ⚡ Preparing Fedora Server with Ansible

📍 Here we will be running a Ansible Playbook to prepare Fedora Server for running a Kubernetes cluster.

📍 Nodes are not security hardened by default, you can do this with [dev-sec/ansible-collection-hardening](https://github.com/dev-sec/ansible-collection-hardening) or similar if it supports Fedora Server.

1. Ensure you are able to SSH into your nodes from your workstation using a private SSH key **without a passphrase**. This is how Ansible is able to connect to your remote nodes.

   [How to configure SSH key-based authentication](https://www.digitalocean.com/community/tutorials/how-to-configure-ssh-key-based-authentication-on-a-linux-server)

2. Install the Ansible deps

    ```sh
    task ansible:init
    ```

3. Verify Ansible can view your config

    ```sh
    task ansible:list
    ```

4. Verify Ansible can ping your nodes

    ```sh
    task ansible:ping
    ```

5. Run the Fedora Server Ansible prepare playbook

    ```sh
    task ansible:prepare
    ```

6. Reboot the nodes

    ```sh
    task ansible:reboot
    ```
### ⛵ Installing k3s with Ansible

📍 Here we will be running a Ansible Playbook to install [k3s](https://k3s.io/) with [this](https://galaxy.ansible.com/xanmanning/k3s) wonderful k3s Ansible galaxy role. After completion, Ansible will drop a `kubeconfig` in `./provision/kubeconfig` for use with interacting with your cluster with `kubectl`.

☢️ If you run into problems, you can run `task ansible:nuke` to destroy the k3s cluster and start over.

1. Verify Ansible can view your config

    ```sh
    task ansible:list
    ```

2. Verify Ansible can ping your nodes

    ```sh
    task ansible:ping
    ```

3. Install k3s with Ansible

    ```sh
    task ansible:install
    ```

4. Verify the nodes are online

    ```sh
    task cluster:nodes
    # NAME           STATUS   ROLES                       AGE     VERSION
    # k8s-0          Ready    control-plane,master      4d20h   v1.21.5+k3s1
    # k8s-1          Ready    worker                    4d20h   v1.21.5+k3s1
    ```

### ☁️ Configuring Cloudflare DNS with Terraform










### 🔹 GitOps with Flux

📍 Here we will be installing [flux](https://toolkit.fluxcd.io/) after some quick bootstrap steps.

1. Verify Flux can be installed

    ```sh
    task cluster:verify
    # ► checking prerequisites
    # ✔ kubectl 1.21.5 >=1.18.0-0
    # ✔ Kubernetes 1.21.5+k3s1 >=1.16.0-0
    # ✔ prerequisites checks passed
    ```

2. Push you changes to git

    📍 **Verify** all the `*.sops.yaml` and `*.sops.yml` files under the `./cluster` and `./provision` folders are **encrypted** with SOPS

    ```sh
    git add -A
    git commit -m "Initial commit :rocket:"
    git push
    ```

3. Install Flux and sync the cluster to the Git repository

    ```sh
    task cluster:install
    # namespace/flux-system configured
    # customresourcedefinition.apiextensions.k8s.io/alerts.notification.toolkit.fluxcd.io created
    ```

4. Verify Flux components are running in the cluster

    ```sh
    task cluster:pods -- -n flux-system
    # NAME                                       READY   STATUS    RESTARTS   AGE
    # helm-controller-5bbd94c75-89sb4            1/1     Running   0          1h
    # kustomize-controller-7b67b6b77d-nqc67      1/1     Running   0          1h
    # notification-controller-7c46575844-k4bvr   1/1     Running   0          1h
    # source-controller-7d6875bcb4-zqw9f         1/1     Running   0          1h
    ```

### 🎤 Verification Steps

_Mic check, 1, 2_ - In a few moments applications should be lighting up like a Christmas tree 🎄

You are able to run all the commands below with one task

```sh
task cluster:resources
```

1. View the Flux Git Repositories

    ```sh
    task cluster:gitrepositories
    ```

2. View the Flux kustomizations

    ```sh
    task cluster:kustomizations
    ```

3. View all the Flux Helm Releases

    ```sh
    task cluster:helmreleases
    ```

4. View all the Flux Helm Repositories

    ```sh
    task cluster:helmrepositories
    ```

5. View all the Pods

    ```sh
    task cluster:pods
    ```

6. View all the certificates and certificate requests

    ```sh
    task cluster:certificates
    ```

7. View all the ingresses

    ```sh
    task cluster:ingresses
    ```

🏆 **Congratulations** if all goes smooth you'll have a Kubernetes cluster managed by Flux, your Git repository is driving the state of your cluster.

☢️ If you run into problems, you can run `task ansible:nuke` to destroy the k3s cluster and start over.

🧠 Now it's time to pause and go get some coffee ☕ because next is describing how DNS is handled.

## 📣 Post installation

### 🔏 Authenticate Flux over SSH - Put this after running flux config

Authenticating Flux to your git repository has a couple benefits like using a private git repository and/or using the Flux [Image Automation Controllers](https://fluxcd.io/docs/components/image/).

By default this template only works on a public GitHub repository, it is advised to keep your repository public.

The benefits of a public repository include:

* Debugging or asking for help, you can provide a link to a resource you are having issues with.
* Adding a topic to your repository of `k8s-at-home` to be included in the [k8s-at-home-search](https://whazor.github.io/k8s-at-home-search/). This search helps people discover different configurations of Helm charts across others Flux based repositories.

<details>
  <summary>Expand to read guide on adding Flux SSH authentication</summary>

  1. Generate new SSH key:
      ```sh
      ssh-keygen -t ecdsa -b 521 -C "github-deploy-key" -f ./cluster/github-deploy-key -q -P ""
      ```
  2. Paste public key in the deploy keys section of your repository settings
  3. Create sops secret in `cluster/flux/flux-system/github-deploy-key.sops.yaml` with the contents of:
      ```yaml
      # yamllint disable
      apiVersion: v1
      kind: Secret
      metadata:
          name: github-deploy-key
          namespace: flux-system
      stringData:
          # 3a. Contents of github-deploy-key
          identity: |
              -----BEGIN OPENSSH PRIVATE KEY-----
                  ...
              -----END OPENSSH PRIVATE KEY-----
          # 3b. Output of curl --silent https://api.github.com/meta | jq --raw-output '"github.com "+.ssh_keys[]'
          known_hosts: |
              github.com ssh-ed25519 ...
              github.com ecdsa-sha2-nistp256 ...
              github.com ssh-rsa ...
      ```
  4. Encrypt secret:
      ```sh
      sops --encrypt --in-place ./cluster/flux/flux-system/github-deploy-key.sops.yaml
      ```
  5. Apply secret to cluster:
      ```sh
      sops --decrypt cluster/flux/flux-system/github-deploy-key.sops.yaml | kubectl apply -f -
      ```
  6.  Update `cluster/flux/flux-system/flux-cluster.yaml`:
      ```yaml
      ---
      apiVersion: source.toolkit.fluxcd.io/v1beta2
      kind: GitRepository
      metadata:
        name: flux-cluster
        namespace: flux-system
      spec:
        interval: 10m
        # 6a: Change this to your user and repo names
        url: ssh://git@github.com/$user/$repo
        ref:
          branch: main
        secretRef:
          name: github-deploy-key
      ```
  7. Commit and push changes
  8. Force flux to reconcile your changes
     ```sh
     task cluster:reconcile
     ```
  9. Verify git repository is now using SSH:
      ```sh
      task cluster:gitrepositories
      ```
  10. Optionally set your repository to Private in your repository settings.
</details>



### 🔏 Setting up OPNSense Router to have some Fun

BGP Setup:
I'm using opnSense as my primary router. Below are some notes on setting that up on there
    Walkthrough on basics: https://forum.opnsense.org/index.php?topic=15756.0
    Install ffr-plugin on OpnSense
    Enable both Routing and BGPv4 from OpnSense admin gui
    Configure BGPv4 as follows (edit AS number as needed):
        Enable: Check
        BGP AS Number: 64512

Build 2 New VLANS - 
    NET_NODE_CIDR Network (/24)
    NET_EIP_CIDR Network (/24)
    Setup Firewall Rules
        I'm allowing all between NET_NODE_CIDR and my primary LAN
        For now, I'm also allowing all between NET_EIP_CIDR Network and my LAN, but this will get locked down later

Install OS-FRR

<details>
  <summary>Expand to read guide on IPV6 Setup with ATT to get more than a /64 out the network:</summary>

Got this from: https://forums.att.com/conversations/att-fiber-account/ipv6-prefix-delegation/5e6309f2c17a0663de2c0532

I'm up and running now with both native and tunnel routing for IPv6 using OpenWrt and have finally two subnets with correct v6 addressing.

LAN has

option ip6hint '000a'

GUEST has

option ip6hint '000b'

I skipped using 192.168.1.0/24 because that routes to the BGW320 for its web interface at 192.168.1.254

I still demand my /56

Here's the /etc/config/network settings

config interface 'wan6tun'
	option proto '6rd'
	option iface6rd 'wan'
	option peeraddr '12.83.49.81'
	option ip6prefixlen '60'
	option mtu '1480'
	option ip6prefix '2602:30x:xxxx:xxx0::'
	option ip4prefixlen '32'

config interface 'wan6native'
	option ifname 'eth1.2'
	option proto 'dhcpv6'
	option reqaddress 'try'
	option peerdns '0'
	list dns '2001:4860:4860::8888'
	list dns '2001:4860:4860::8844'
	option reqprefix '56'     # maybe one day we'll finally get a /56

The tunnel should auto-config from a DHCP option, but AT&T doesn't advertise it so you'll need to calculate your addy

@OpenWrt:~# V4L=69.209.xx.xx
@OpenWrt:~# echo $V4L | awk -F. '{ t=sprintf("%02x%02x%02x%02x", $1, $2, $3, $4); print "2602:30"substr(t,1,1)":"substr(t,2,4)":"substr(t,6)"0::" }'
2602:30x:xxxx:xxx0::
@OpenWrt:~#
</details>