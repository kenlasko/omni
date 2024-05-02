# Introduction
This repository is used to install a Talos Kubernetes cluster using on-prem Omni. Most of these steps should work without modification. Obviously paths and domain names should change as required.

# Omni On-Prem Installation
I installed Omni on a Raspberry Pi I'm using for other Docker-related stuff.
1. Follow the [Omni on-prem install instructions](https://omni.siderolabs.com/docs/how-to-guides/how-to-deploy-omni-on-prem/).
2. Configure [docker-compose.yaml](docker-compose.yaml) file
3. Make sure omni.ucdialplans.com is added to whatever is being used to serve DNS

# Omnictl/Talosctl installation
1. Download omnictl and talosctl from omni.ucdialplans.com and put in proper locations
```
sudo mv omnictl-linux-amd64 /usr/local/bin/omnictl
sudo mv talosctl-linux-amd64 /usr/local/bin/talosctl
sudo chmod u+x /usr/local/bin/omnictl /usr/local/bin/talosctl
```
2. Download omniconfig.yaml and talosconfig.yaml from omni.ucdialplans.com and put in proper locations
```
mv omniconfig.yaml /home/ken/.config/omni/config
mv talosconfig.yaml /home/ken/.talos/config
```
# Omni/Kubectl installation
Assumes Kubectl is already installed.
1. Install Krew
```
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)
```

2. Add Krew to ~/.bashrc
```
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
```

3. Restart shell
```
source ~/.bashrc
```

4. Install OIDC-Login in Kubectl
```
kubectl krew install oidc-login
```

5. Install wslu (for WSL browser redirection)
```
sudo apt install wslu -y
```

# Omni cluster creation/update
```
omnictl cluster template sync -f ~/omni/cluster-(home|lab|laptop).yaml
```