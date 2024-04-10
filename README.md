# Omni/Kubectl installation
Assumes Kubectl is already installed.
## Install Krew
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

## Add Krew to ~/.bashrc
```
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
```

## Restart shell
```
source ~/.bashrc
```

## Install OIDC-Login in Kubectl
```
kubectl krew install oidc-login
```

## Install wslu (for browser redirection)
```
sudo apt install wslu -y
```

# Omni cluster creation
omnictl cluster template sync -f cluster.yaml