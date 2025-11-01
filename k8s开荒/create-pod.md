```yaml
# nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx-container
      image: nginx
```
运行后，Kubernetes会创建一个Pod：
- Pod名称: `nginx-pod`
- 容器名称: `nginx-container`
- 镜像: `nginx-latest`
- 容器启动命令: 使用 `nginx`默认ENTRYPOINT（会启动nginx服务监听80端口）
假设有已启动的k8s集群，运行命令为：
```bash
kubectl apply -f nginx.yaml
```
查看Pod状态：
```bash
kubectl get pods
```

会得到类似的结果：
```sql
NAME         READY   STATUS    RESTARTS   AGE
nginx-pod    1/1     Running   0          10s
```

查看Pod日志：
```bash
kubectl logs nginx-pod
```
进入容器：
```bash
kubectl exec -it nginx-pod -- /bin/bash
```
测速服务端口：将nginx默认的80端口映射到本机的4000端口
```bash
kubectl port-forward nginx-pod 4000:80
```

为了让其他Pod或Service知道容器暴露了哪些端口，可以添加`ports`字段：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx-container
      image: nginx
      ports:
        - containerPort: 80
```

如果想让外部访问，可以用一个Serivce暴露它：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```