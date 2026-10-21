# K8S Web 部署说明

纯静态站点(`index.html` + `ingress.html`),通过 ConfigMap 挂载到 `nginx:alpine`,无需构建镜像。

## 一、部署步骤

```bash
# 0. 进入项目根目录(即 index.html 所在目录)
cd D:/git/k8s-web

# 1. 用本地页面生成 ConfigMap(在 k8s-web 命名空间)
kubectl create namespace k8s-web
kubectl create configmap k8s-web-html -n k8s-web \
  --from-file=index.html \
  --from-file=ingress.html \
  --dry-run=client -o yaml | kubectl apply -f -

# 2. 部署 Deployment / Service / Ingress
kubectl apply -f k8s/deploy.yaml

# 3. 先改 k8s/deploy.yaml 里的 host(k8s-web.example.com)和 ingressClassName,
#    没有域名时可用端口转发本地验证:
kubectl port-forward svc/k8s-web 8080:80 -n k8s-web
# 浏览器访问 http://localhost:8080
```

## 二、更新页面后重新发布

```bash
# subPath 挂载不会热更新,需重建 ConfigMap 并重启 Pod
kubectl create configmap k8s-web-html -n k8s-web \
  --from-file=index.html --from-file=ingress.html \
  --dry-run=client -o yaml | kubectl apply -f -
kubectl rollout restart deploy/k8s-web -n k8s-web
```

## 三、验证

```bash
kubectl get pods,svc,ingress -n k8s-web
# Ingress ADDRESS 出现 IP 且 Pod Ready 即部署成功
curl -H "Host: k8s-web.example.com" http://<INGRESS_IP>/
```

## 四、文件说明

| 文件 | 说明 |
|---|---|
| `k8s-web.yaml` | ★ 推荐:hostPath 版一键部署,nginx 根目录指向节点 `/data/git/web`,改文件即时生效 |
| `index.html` | 站点主页,含跳转到 ingress.html 的入口卡片 |
| `ingress.html` | Ingress 完整使用文档 |
| `k8s/deploy.yaml` | Namespace + Deployment + Service + Ingress |
| `ingress.yaml` | ★ Ingress 规则(ergesanbaishou ns),经 controller NodePort 31588 入口访问 |
