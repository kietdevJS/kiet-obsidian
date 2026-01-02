
---

# Essential Dockerfile Commands

### 1. FROM

Defines the base image your container will build on.

```dockerfile
FROM ubuntu:22.04
```

Usually the first line in most Dockerfiles.

---

### 2. WORKDIR

Sets the working directory inside the container (like `cd`).

```dockerfile
WORKDIR /app
```

All following commands (`RUN`, `COPY`, `CMD`) will be executed inside `/app`.

---

### 3. COPY

Copies files/folders from host → container.

```dockerfile
COPY . /app
```

Copies everything from your local folder into `/app` inside the container.

---

### 4. RUN

Executes commands while building the image (install software, etc.).

```dockerfile
RUN apt-get update && apt-get install -y python3
```

Each `RUN` creates a new image layer.

---

### 5. ENV

Sets environment variables inside the container.

```dockerfile
ENV APP_ENV=production
```

Useful for configuration values the app needs.

---

### 6. EXPOSE

Documents the port the container will listen on.

```dockerfile
EXPOSE 8080
```

This does not publish the port — you still need `docker run -p`.

---

### 7. CMD

Specifies the default command when the container runs.

```dockerfile
CMD ["python3", "app.py"]
```

Can be overridden by `docker run <image> <command>`.

---

### 8. ENTRYPOINT

Defines a fixed main command for the container.

```dockerfile
ENTRYPOINT ["python3", "app.py"]
```

Arguments given at `docker run` are passed to this.

---

# Example Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt ./
RUN pip install -r requirements.txt

COPY . .

ENV PORT=5000
EXPOSE 5000

ENTRYPOINT ["python"]
CMD ["app.py"]
```

Would you like me to also make a **side-by-side comparison of CMD vs ENTRYPOINT** (when to use one over the other)?