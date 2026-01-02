#### On Docker

1. Create server to host the Rancher app
2. Run command to install docker, docker-compose

```cmd
apt update
apt install docker.io
apt install docker-compose
```

3. Install rancher
	1. Check for rancher version is compatible with kube version ![[Screenshot 2025-12-08 at 21.40.52.png]]
	2. ![[Screenshot 2025-12-08 at 21.41.51.png]]

4. Create docker-compose.yml

```
version: '3'
services:
	rancher-server:
		image: rancher/rancher:v2.9.2
		container_name: rancher-server
		restart: unless-stopped
		ports:
		- "80:80"
		- "443:443"
		volumes:
		- ////
		privileged: true
```

5. Exec

```cmd
docker-compose up -d
```

6. Connect Cluster with Rancher
