Docker file commands: [[Docker file commands]]

How to write optimized docker file:
-  Use non root user
-  Base image:
	- Alpine image is best for being light weigh
	- Tool for image scan
-  Multiple stage:
	- Normally, when you build a Docker image, every `RUN`, `COPY`, etc. adds layers, and you might end up with a big image (including build tools you don’t need at runtime).  
		**Multi-stage builds** let you use multiple `FROM` statements in one Dockerfile, and only keep what you need.

---

Notes:
#### Port exposing and running

- Declaring a port to expose in dockerfile![[Screenshot 2025-09-01 at 20.39.29.png]]
- When run the container, the port to access from host will go first, then the exposed port by the container go later

```
  docker run --name shoeshop -dp 8888:8080 shoeshop:v1
```

