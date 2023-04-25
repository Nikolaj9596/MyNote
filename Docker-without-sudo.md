## Docker run without sudo

```bash
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
```
