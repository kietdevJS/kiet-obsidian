# Docker Volume

## What is it?
- Persistent storage for Docker containers.
- Data lives outside the container (survives restarts/deletion).
- Managed by Docker (`/var/lib/docker/volumes/`).

## Why use it?
-  Persist data  
-  Share between containers  
-  Better performance than container FS  
-  Easy backup & migration  

## Commands

```
# Create
docker volume create myvolume

# List
docker volume ls

# Inspect
docker volume inspect myvolume

# Remove
docker volume rm myvolume
```

### Run with a Named Volume

`docker run -d -v myvolume:/app/data myimage`

- `myvolume` = Docker-managed volume (created automatically if not exist).
    
- `/app/data` = path inside the container.
    
- `myimage` = your Docker image.

Data written in `/app/data` persists even if the container is removed.

---

###  Run with a Host Directory (Bind Mount)

`docker run -d -v /host/path:/app/data myimage`

- `/host/path` = directory on your machine.
    
- `/app/data` = directory inside the container.
    

👉 Useful if you want to directly edit files on your host and see changes inside the container.

---

### 🐳 Example with Nginx

`docker run -d -p 8080:80 -v myvolume:/usr/share/nginx/html nginx`

- Stores Nginx website files in `myvolume`.