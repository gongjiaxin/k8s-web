# K8S Web 部署说明

纯静态站点(`index.html` + 6 个文档页)。★ 线上采用 hostPath 版(`k8s-web.yaml`):nginx 根目录指向节点 `/data/git/web`。

## 零、★ 本地改完 → 发布到线上(hostPath 版必看)

本地 `D:\git\github\k8s-web` 只是源文件,**线上站点读的是服务器节点 `/data/git/web` 里的副本** —— 本地改完不同步,线上永远看不到!在开发机执行:

```bash
# 同步全部页面(如本次新增 k8s-mysql.html + 改了 index.html)
scp D:/git/github/k8s-web/*.html root@10.18.170.34:/data/git/web/

# 或只同步本次涉及的两个文件
scp D:/git/github/k8s-web/index.html D:/git/github/k8s-web/k8s-mysql.html root@10.18.170.34:/data/git/web/

# 服务器上核对(按时间排,新文件应在最上面)
ssh root@10.18.170.34 "ls -lt /data/git/web | head"
```

- hostPath 特性:文件落盘**立即生效**,无需重启 Pod;浏览器可能缓存主页,用 **Ctrl+F5** 强刷。
- ⚠️ `k8s-web.yaml` 为 2 副本且未固定节点:Pod 分布在多个节点时,每个节点都要放一份页面;只想维护一份就按清单注释打标签 + 启用 `nodeSelector`。
- 不发布的本地预览:浏览器直接打开 `D:\git\github\k8s-web\index.html`。

## 一、部署步骤

```bash
# 0. 进入项目根目录(即 index.html 所在目录)
cd D:/git/github/k8s-web

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
| `k8s/deploy.yaml` | Namespace + Deployment + Service + Ingress(ConfigMap 版备选) |
| `k8s-cert.html` | K8S 证书管理:域名 SSL(443) + 集群内部证书(pki) |
| `k8s-cmd.html` | K8S 常用命令速查 |
| `k8s-issues.html` | K8S 使用问题总汇(排障一本通) |
| `k8s-mysql.html` | K8S 安装 MySQL(Helm · 1主1备) |
| `ingress.yaml` | ★ Ingress 规则(ergesanbaishou ns),经 controller NodePort 31588 入口访问 |
