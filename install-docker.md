## Install Docker on Fedora

Run the following commands to install Docker:

```bash
sudo dnf -y install dnf-plugins-core
sudo dnf-3 config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## Start Docker automatically 

```bash 
sudo systemctl enable --now docker
```