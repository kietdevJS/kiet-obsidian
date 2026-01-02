![[Screenshot 2025-11-01 at 19.41.09.png]]
![[Pasted image 20251101194347.png]]

1. Control plane: usually run on a dedicated server
	- api-server: where kubectl command goes
	- etcd: cluster state and config database
	- controller manager: manage if everything runs well
	- scheduler: decide the which node to put the pod
2. Worker nodes: where pods run
	- Kubelet: manage pod
	- Kube-proxy: networking
	- Container runtime: vm to run app (docker, containerd, CRI-O)