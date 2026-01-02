# 🐳 Docker Compose

## What is it?

- Tool to **define & run multi-container apps** with a YAML file.
- Good for **local dev & small deployments**.
- Not a full orchestrator (like Kubernetes).

---
### Top-level keys

- **version** → (optional in v2.20+)  
- **services** → Defines containers to run
- **volumes** → (optional) Named volumes for persistent data.  
- **networks** → (optional) Custom networks.  

### Example file

```
version: "3.9"

services:
  web:
    image: nginx:latest        # Docker image
    ports:
      - "8080:80"              # host:container
    volumes:
      - ./html:/usr/share/nginx/html
    depends_on:
      - db                     # ensures db starts first

  db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: example
    volumes:
      - dbdata:/var/lib/mysql

volumes:
  dbdata:                      # named volume for MySQL
```

## Explanation

- **services** → Each container (e.g., `web`, `db`).
    
    - `image` → Which Docker image to use.
        
    - `ports` → Expose container ports to host.
        
    - `volumes` → Mount host paths or named volumes.
        
    - `environment` → Env vars for the container.
        
    - `depends_on` → Control startup order.
        
- **volumes** → Define named volumes shared across services.
    
- **networks** → Define custom networking (optional).