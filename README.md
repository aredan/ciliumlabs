### Cilium LABs with Kind
This is based on https://github.com/chornberger-c2c/isovalent-cilium-lab/blob/main/lab.md with minor changes and all the steps together for easy follow-up.
More stuff will be added soon.

Why?

I know that Cilium offers hosted LABs for free, but here I'm just trying to play with the labs in a different way and without time constrain for lab completion.

### pre-requesities.
For all the LABs in this repo you will ned Docker, Kind, kubectl and kind-cli.

### Docker installation.
Add the repo to your system.

```shell
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

Install the package.

```shell
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Add your user to the Docker group.

```shell
sudo adduser <user> docker
```

Test your Docker installation.

```shell
sudo docker run hello-world
```

### Kind Installation.


```shell
# see https://kind.sigs.k8s.io/docs/user/quick-start#installation
# For AMD64 / x86_64
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
# For ARM64
[ $(uname -m) = aarch64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-arm64
chmod +x kind
mv kind ~/bin  
```

> make sure that you have ~/bin/ in your $PATH

#### Inotify Settings.

```shell
# see https://kind.sigs.k8s.io/docs/user/known-issues/#pod-errors-due-to-too-many-open-files
sudo echo "fs.inotify.max_user_watches=524288 " >> /etc/sysctl.conf 
sudo echo "fs.inotify.max_user_instances=512 "  >> /etc/sysctl.conf  
sudo sysctl -p 
```

### kubectl
You need the kubectl binary to interact with K8s.

```shell
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
mv ./kubectl ~/bin/kubectl
```

### Cilium CLI Tool.
You need to install the cilium cli tool, in this case is easier to use the cilium cli instead of Helm.

```shell
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

```shell
cilium version --client
```
> Make sure you install cilium-cli v0.15.0 or later.


### Next steps.
Now you can continue to one of the fallowing configuration for testing different componentes of Cilium.

#### beginner lab
For a basic lab configuration go to[begginer/](beginner/README.md)