![[Pasted image 20251104232442.png]]

Ingress là một tài nguyên ở mức Namespace trên K8S. Và giống như các tài nguyên khác như Pod, Deployment hay Service, ta có thể định nghĩa nó bằng cách sử dụng file manifest dạng yaml.

**Ví dụ nội dung một file định nghĩa Ingress như sau:**

```none
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx-example
  rules:
  - http:
      paths:
      - path: /testpath
        pathType: Prefix
        backend:
          service:
            name: test
            port:
              number: 80
```

Ý nghĩa của khai báo trên là mọi request tới mà có Path chứa Prefix là **/testpath** thì sẽ được forward tới servcie **test** ở port **80**.