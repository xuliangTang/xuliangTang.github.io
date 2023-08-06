---
title: "Cert Manager 签发证书"
date: 2023-08-06
draft: false
Categories: [kubernetes]
---

[cert-manageropen](https://cert-manager.io/docs/) 是 Kubernetes 上的全能证书管理工具，支持利用 cert-manager 基于 [ACMEopen](https://tools.ietf.org/html/rfc8555) 协议与 [Let's Encryptopen](https://letsencrypt.org/) 签发免费证书并为证书自动续期，实现永久免费使用证书

```bash
# 安装
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.0/cert-manager.yaml
```



## 工作原理

cert-manager 部署到 Kubernetes 集群后，它会 watch 它所支持的 CRD 资源，我们通过创建 CRD 资源来指示 cert-manager 为我们签发证书并自动续期

![README-2021-10-14-20-29-31](https://raw.githubusercontent.com/xuliangTang/picbeds/main/picgo/202308062236663.svg)

- `Issuer` / `ClusterIssuer`: 用于指示 cert-manager 用什么方式签发证书（ClusterIssuer 与 Issuer 的唯一区别就是 Issuer 只能用来签发自己所在 namespace 下的证书，ClusterIssuer 可以签发任意 namespace 下的证书）
  - 由权威 CA 机构签名的「公网受信任证书：这类证书会被浏览器、小程序等第三方应用/服务商信任
  - 本地签名证书：即由本地 CA 证书签名的数字证书
  - 自签名证书：即使用证书的私钥为证书自己签名
- `Certificate`: 用于告诉 cert-manager 我们想要什么域名的证书以及签发证书所需要的一些配置，包括对 Issuer/ClusterIssuer 的引用



## 示例

### 创建自签证书

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: myissuer
  namespace: default
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: tls-p
  namespace: default
spec:
  secretName: tls-p # 生成的secret名称
  duration: 5h		# 有效期
  renewBefore: 2h	# 自动续期临近日期
  dnsNames:
  - p.virtuallain.com
  issuerRef:
    name: myissuer	# 关联签发者
```

### 双向认证

生成CA证书

```bash
# 生成key文件
openssl genrsa -out ca.key 2048
# 生成ca.crt证书文件
openssl req -x509 -new -nodes -key ca.key -subj "/CN=*.virtuallain.com" -days 3650 -reqexts v3_req -extensions v3_ca -out ca.crt
# 导入k8s
kubectl create secret tls myca --cert=ca.crt --key=ca.key -n default
```

创建 issuer 和 certificate 资源，自动生成服务端证书

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: ca-issuer
  namespace: default
spec:
  ca:
    secretName: myca
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: tls-p
  namespace: default
spec:
  secretName: tls-p
  commonName: p.virtuallain.com
  dnsNames:
  - p.virtuallain.com
  issuerRef:
    name: ca-issuer
    kind: Issuer
```

ingress 开启双向认证

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: p.virtuallain.com
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/auth-tls-secret: default/tls-p	# 证书secret
    nginx.ingress.kubernetes.io/auth-tls-verify-client: "on"	# 开启双向认证
spec:
  ingressClassName: nginx
  rules:
  - host: p.virtuallain.com
    http:
      paths:
      - backend:
          service:
            name: test
            port:
              number: 80
        path: /
        pathType: Prefix
  tls:
  - hosts:
    - p.virtuallain.com
    secretName: tls-p	# 证书secret
```

生成客户端证书

```bash
openssl genrsa -out client.key 2048
openssl req -new -key client.key -out client.csr -subj "/CN=p.virtuallain.com"
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client.crt -days 365
```

客户端请求

```go
func main() {
	pool := x509.NewCertPool()
	caCrt, _ := ioutil.ReadFile("./certs/ca.crt")
	pool.AppendCertsFromPEM(caCrt)
	
	cliCrt, _ := tls.LoadX509KeyPair("./certs/client.crt", "./certs/client.key")

	tr := &http.Transport{
		TLSClientConfig: &tls.Config{
			RootCAs:      pool,
			Certificates: []tls.Certificate{cliCrt},
			// InsecureSkipVerify: true,
		},
	}
	req, err := http.NewRequest("GET", "https://p.virtuallain.com/", nil)
	if err != nil {
		log.Fatal(err)
	}

	client := &http.Client{Transport: tr}
	rsp, err := client.Do(req)
	if err != nil {
		log.Fatal(err)
	}
	defer rsp.Body.Close()
	b, _ := ioutil.ReadAll(rsp.Body)
	fmt.Println(string(b))
}
```

### 签发 Let’s Encrypt 免费证书

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsissuer
spec:
  acme:
    email: ""
    server: https://acme-v02.api.letsencrypt.org/directory
    preferredChain: "ISRG Root X1"
    privateKeySecretRef:
      name: letstlskey
    solvers:
      - http01:
          ingress:
            class: nginx
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: tls-p
  namespace: default
spec:
  secretName: tls-p
  subject:
    organizations:
      - virtuallain.com
  issuerRef:
    name: letsissuer
    kind: ClusterIssuer
    group: cert-manager.io
  dnsNames:
    - p.virtuallain.com
  privateKey:
    algorithm: RSA
    encoding: PKCS1
    size: 2048
```
