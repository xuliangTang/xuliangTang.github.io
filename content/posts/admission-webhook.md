---
title: "Admission Webhook"
date: 2023-02-08
draft: false
Categories: [kubernetes]
---

Kubernetes 准入控制器是集群管理必要功能。这些控制器主要在后台工作，并且许多可以作为编译插件使用，它可以极大地提高部署的安全性

## Admission Controller

[准入控制器](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/admission-controllers/) 在 API 请求传递到 APIServer 之前拦截它们，并且可以禁止或修改它们。这适用于大多数类型的 Kubernetes 请求。准入控制器在经过适当的身份验证和授权后处理请求

默认情况下启用了几个准入控制器，因为大多数正常的 Kubernetes 操作都依赖于它们。这些控制器中的大多数都包含一些 Kubernetes 源代码树，并被编译为插件。但是，也可以编写和部署第三方准入控制器

在 Kubernetes apiserver 中包含两个特殊的准入控制器：`MutatingAdmissionWebhook` 和 `ValidatingAdmissionWebhook`。这两个控制器将发送准入请求到外部的 HTTP 回调服务并接收一个准入响应。如果启用了这两个准入控制器，Kubernetes 管理员可以在集群中创建和配置一个 admission webhook



## Admission Webhook

准入 Webhook 是一种用于接收准入请求并对其进行处理的 HTTP 回调机制。 可以定义两种类型的准入 webhook，即 [验证性质的准入 Webhook](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/admission-controllers/#validatingadmissionwebhook) 和 [修改性质的准入 Webhook](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/admission-controllers/#mutatingadmissionwebhook)， 修改性质的准入 Webhook 会先被调用

![k8s api request lifecycle](https://raw.githubusercontent.com/xuliangTang/picbeds/main/picgo/202302081950884.png)

总的来说，这样做的步骤如下：

- 检查集群中是否启用了 admission webhook 控制器，并根据需要进行配置
- 编写处理准入请求的 HTTP 回调，回调可以是一个部署在集群中的简单 HTTP 服务，甚至也可以是一个 serverless 函数，例如：[denyenv-validating-admission-webhook](https://github.com/kelseyhightower/denyenv-validating-admission-webhook)
- 通过 `MutatingWebhookConfiguration` 和 `ValidatingWebhookConfiguration` 资源配置 admission webhook

这两种类型的 admission webhook 之间的区别是非常明显的：validating webhooks 可以拒绝请求，但是它们却不能修改在准入请求中获取的对象，而 mutating webhooks 可以在返回准入响应之前通过创建补丁来修改对象，如果 webhook 拒绝了一个请求，则会向最终用户返回错误

```bash
# 查看默认的控制器插件
kube-apiserver --help |grep enable-admission-plugins 
```



## 编写 Webhook

webhook 是一个 http server，是一个限制了请求与响应格式（AdmissionReview/AdmissionResponse）的 http server。下面实现了一个 webhook 示例，通过监听 /mutate 来进行 mutating webhook 验证。这个 webhook 是一个简单的带 TLS 认证的 HTTP 服务，用 Deployment 方式部署在我们的集群中

main.go 文件包含创建 HTTP 服务的代码

```go
func main() {
	http.HandleFunc("/mutate", func(w http.ResponseWriter, r *http.Request) {
		var body []byte
		if r.Body != nil {
			if data, err := io.ReadAll(r.Body); err == nil {
				body = data
			}
		}
		if len(body) == 0 {
			klog.Error("empty body")
			http.Error(w, "empty body", http.StatusBadRequest)
			return
		}

		reqAdmissionReview := admissionV1.AdmissionReview{} // 请求
		rspAdmissionReview := admissionV1.AdmissionReview{  // 响应
			TypeMeta: metaV1.TypeMeta{
				Kind:       "AdmissionReview",
				APIVersion: "admission.k8s.io/v1",
			},
		}

		// 把body decode成对象
		deserializer := lib.Codecs.UniversalDeserializer()
		if _, _, err := deserializer.Decode(body, nil, &reqAdmissionReview); err != nil {
			klog.Error(err)
			rspAdmissionReview.Response = lib.ToV1AdmissionResponse(err)
		} else {
			rspAdmissionReview.Response = lib.AdmitPods(reqAdmissionReview) // 具体逻辑
		}
		
		rspAdmissionReview.Response.UID = reqAdmissionReview.Request.UID
		respBytes, err := json.Marshal(rspAdmissionReview)
		if err != nil {
			klog.Errorf("Can't encode response: %v", err)
			http.Error(w, fmt.Sprintf("could not encode response: %v", err), http.StatusInternalServerError)
		}

		if _, err := w.Write(respBytes); err != nil {
			klog.Errorf("Can't write response: %v", err)
			http.Error(w, fmt.Sprintf("could not write response: %v", err), http.StatusInternalServerError)
		}
	})

	tlsConfig := lib.Config{	// 挂载的证书文件
		CertFile: "/etc/webhook/certs/tls.crt",
		KeyFile:  "/etc/webhook/certs/tls.key",
	}
	server := &http.Server{
		Addr:      ":443",
		TLSConfig: lib.ConfigTLS(tlsConfig),
	}

	go func() {
		if err := server.ListenAndServeTLS("", ""); err != nil {
			klog.Errorf("Failed to listen and serve webhook server: %v", err)
		}
	}()

	klog.Info("Server started")

	// listening OS shutdown signal
	signalChan := make(chan os.Signal, 1)
	signal.Notify(signalChan, syscall.SIGINT, syscall.SIGTERM)
	<-signalChan

	klog.Infof("Got OS shutdown signal, shutting down webhook server gracefully...")
	server.Shutdown(context.Background())
}
```

lib/pods.go 文件的 AdmitPods 方法验证了 pod 名不能是 abc，并且创建了2个补丁，第1个补丁修改第一个容器的镜像为 nginx:1.19-alpine，第2个补丁添加了一个 init 容器到资源中

```go
func patchImage() []byte {
	str := `[
	{
		"op": "replace",
		"path": "/spec/containers/0/image",
		"value": "nginx:1.19-alpine"
	},
	{
		"op": "add",
		"path": "/spec/initContainers",
		"value": [{
					"name": "init-test",
					"image": "busybox:1.28",
					"command": ["sh", "-c", "echo init container is running!"]
				}]
	}
]`
	return []byte(str)
}

func AdmitPods(ar v1.AdmissionReview) *v1.AdmissionResponse {
	podResource := metav1.GroupVersionResource{Group: "", Version: "v1", Resource: "pods"}

	if ar.Request.Resource != podResource {
		err := fmt.Errorf("expect resource to be %s", podResource)
		klog.Error(err)
		return ToV1AdmissionResponse(err)
	}

	raw := ar.Request.Object.Raw
	pod := corev1.Pod{}
	deserializer := Codecs.UniversalDeserializer()
	if _, _, err := deserializer.Decode(raw, nil, &pod); err != nil {
		klog.Error(err)
		return ToV1AdmissionResponse(err)
	}

	reviewResponse := v1.AdmissionResponse{}
	if pod.Name == "abc" {
		klog.Error("pod name cannot be abc")
		return ToV1AdmissionResponse(fmt.Errorf("pod name cannot be abc"))
	}
	
    reviewResponse.Allowed = true
    reviewResponse.Patch = patchImage()	// 通过创建补丁来修改对象
    jsonPatch := v1.PatchTypeJSONPatch
    reviewResponse.PatchType = &jsonPatch
	
	return &reviewResponse
}

// 统一返回error 响应
func ToV1AdmissionResponse(err error) *v1.AdmissionResponse {
	return &v1.AdmissionResponse{
		Result: &metav1.Status{
			Message: err.Error(),
		},
	}
}
```

lib/scheme.go

```go
var scheme = runtime.NewScheme()
var Codecs = serializer.NewCodecFactory(scheme)

func init() {
	addToScheme(scheme)
}

func addToScheme(scheme *runtime.Scheme) {
	utilruntime.Must(corev1.AddToScheme(scheme))
	utilruntime.Must(admissionv1beta1.AddToScheme(scheme))
	utilruntime.Must(admissionregistrationv1beta1.AddToScheme(scheme))
	utilruntime.Must(admissionv1.AddToScheme(scheme))
	utilruntime.Must(admissionregistrationv1.AddToScheme(scheme))
}
```

lib/config.go

```go
// Config contains the server (the webhook) cert and key.
type Config struct {
	CertFile string
	KeyFile  string
}

func ConfigTLS(config Config) *tls.Config {
	sCert, err := tls.LoadX509KeyPair(config.CertFile, config.KeyFile)
	if err != nil {
		klog.Fatal(err)
	}
	return &tls.Config{
		Certificates: []tls.Certificate{sCert},
		// TODO: uses mutual tls after we agree on what cert the apiserver should use.
		// ClientAuth:   tls.RequireAndVerifyClientCert,
	}
}
```



## 部署服务

为了部署 webhook server，我们需要在我们的 Kubernetes 集群中创建一个 service 和 deployment 资源对象，还要将 TLS 证书映射到目录

### 生成 CA 证书

ca-config.json

```json
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "server": {
        "usages": ["signing"],
        "expiry": "8760h"
      }
    }
  }
}
```

ca-csr.json

```json
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "zh",
      "L": "bj",
      "O": "bj",
      "OU": "CA"
   }
  ]
}
```

生成 CA 证书

```bash
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

### 生成服务端证书

server-csr.json

```json
{
  "CN": "admission",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "zh",
      "L": "bj",
      "O": "bj",
      "OU": "bj"
    }
  ]
}
```

签发证书

```bash
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=myhook.kube-system.svc \
  -profile=server \
  server-csr.json | cfssljson -bare server
```

### 创建 Secret

```bash
kubectl create secret tls myhook --cert=server.pem --key=server-key.pem -n kube-system
```

### 部署工作负载

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myhook
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myhook
  template:
    metadata:
      labels:
        app: myhook
    spec:
      nodeName: lain1
      containers:
        - name: myhook
          image: alpine:3.12
          imagePullPolicy: IfNotPresent
          command: ["/app/myhook"]
          volumeMounts:
            - name: hooktls
              mountPath: /etc/webhook/certs
              readOnly: true
            - name: app
              mountPath: /app
          ports:
            - containerPort: 443
      volumes:
        - name: app
          hostPath:
            path: /home/txl/hook/build
        - name: hooktls
          secret:
            secretName: myhook
---
apiVersion: v1
kind: Service
metadata:
  name: myhook
  namespace: kube-system
  labels:
    app: myhook
spec:
  type: ClusterIP
  ports:
    - port: 443
      targetPort: 443
  selector:
    app: myhook
```

### 部署 MutatingWebhook

接着，创建一个 MutatingWebhookConfiguration 将我们创建的 webhook 信息注册到 Kubernetes API server。CA 证书应提供给 admission webhook 配置，这样 apiserver 才可以信任 webhook server 提供的 TLS 证书

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: myhook
webhooks:
  - clientConfig:
  	  # 填充内容为 cat ca.pem | base64
      caBundle: |
        ...
      service:
        name: myhook
        namespace: kube-system
        path: /mutate
    failurePolicy: Fail
    sideEffects: NoneOnDryRun
    name: myhook.virtuallain.com
    admissionReviewVersions: ["v1", "v1beta1"]
    namespaceSelector:
      matchExpressions:
        - key: pod-injection
          operator: In
          values: [ "enable", "1" ]
    rules:
      - apiGroups:   [""]
        apiVersions: ["v1"]
        operations:  ["CREATE"]
        resources:   ["pods"]
```

如上述的清单信息所示，我们要求 Kubernetes 把（部署了 MutatingWebhookConfiguration ）命名空间中所有的 Pod 创建请求，只要匹配上 “pod-injection=true” 标签的，就将其转发到 myhook 的 “/mutate” 路径下，交给其处理
