![[Screenshot 2025-11-04 at 23.38.51.png]]

#### What is a namespace ?

- Is a way to organize resources in a k8s cluster
- Divide resources into logical workspaces
- **Best Practice**: Use Namespaces to isolate different projects (e.g., a dedicated `car-service` namespace) or environments (dev, staging, prod) to keep the cluster organized and prevent naming collisions.

#### Create namespace

Using command:

1. Get pods:

	```cmd
	kubectl get pod
	or
	kubectl get pod --namespace <name>
	```

2. Get / Create / Delete namespace:

```cmd
kubectl get ns
kubectl create ns <name>
```

Using yaml

1. Create directory
2. Create a yml file to store the setting

```yml
apiVersion: 1
kind: Namespace
metadata:
	name: project-1
```

3. Apply the config

```cmd
kubectl appy -f file.yaml
or
kubectl delete -f file.yaml
```

#### Access control

1. Create config file resourcequota.yml

```yaml
apiversion: v1
kind: ResourceQuota
metadata:
	name: mem-cpu-quota
	namespace: project-1
spec:
	hard:
		requests.cpu: "2"
		requests.memory: 4Gi
```

2. Apply the config (same as above)