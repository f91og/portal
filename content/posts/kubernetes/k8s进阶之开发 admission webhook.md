+++
title = 'k8s 进阶之开发 admission webhook'
date = 2024-12-25T10:56:38Z
draft = false
toc = true
tags = ['k8s']
categories = ['Kubernetes']
+++

admission webhook 是对 api server 的扩展，简单来说就是在 api server 认证客户端请求阶段有 2 个 webhook 可以来定制请求的验证逻辑和对请求进行修改。

<!--more-->
## 简介

官网文档上的解释：[动态准入控制 | Kubernetes](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/extensible-admission-controllers/)

了解这个知识点之前需要知道关于 kube api server 对请求的处理流程，大致如下图所示：
![](/images/20240913225324.png)
图片出处：[Outshift | In-depth introduction to Kubernetes admission webhooks (cisco.com)](https://outshift.cisco.com/blog/k8s-admission-webhooks)

如上图所示，有 2 个 webhook 的地方可以定制一些我们自己想要的逻辑，比如利用 validating webhook 确保 tenant 只有在其 namespace 匹配的情况下才能创建资源，而利用 mutating webhook 则可以在用户创建资源时修改其 api 请求，一个很典型的使用案例是 istio 通过 mutating webhook 来向用户的 pod 里自动注入 envoy 容器。

和 operator 一样，这 2 个是属于概念性质的东西，实际上干活的还是 pods，在 pods 上启动服务器，api server 与这个服务器 https 通信来实现 validate 和 mutate 功能，其基本使用方法就是部署一个 deployment 到集群中。

接下来用 admission webhook 来实践一些简单的功能：
- 限制 svc 的端口，如果 name 不是 https 则不可以使用 443 端口
- 给 svc 自动加一个标签 `enabled`， true 或者 false，表示启用或禁用

开发之前首先要确认集群有没有开启相关的功能：
```shell
# kubectl api-versions
......
admissionregistration.k8s.io/v1 # 有这个就可以了
......
```
没有的话需要在 api server 的启动参数里指定 `enable-admission-plugins` 参数。

## 项目的基本框架

首先来创建新项目：
```shell
mkdir my-webhook && cd my-webhook
go mod init my-webhook
touch main.go
```

然后在 main.go 中添加 api handler ，把大致的 https 请求处理框架给起来：
```go
package main

import (
	"crypto/tls"
	"encoding/json"
	"flag"
	"fmt"
	"io/ioutil"
	"net/http"

	// 项目根目录下创建webhooks, 用于存存放不同resource的valiate，mutate逻辑
	"my-webhook/webhooks" 

	"github.com/golang/glog"
	"k8s.io/api/admission/v1beta1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/runtime/serializer"

	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

type serverParams struct {
	port     int
	certFile string
	keyFile  string
}

type WebhookServer struct {
	server *http.Server
}

var (
	runtimeScheme = runtime.NewScheme()
	codecs        = serializer.NewCodecFactory(runtimeScheme)
	deserializer  = codecs.UniversalDeserializer()
)

func (ws *WebhookServer) validate(w http.ResponseWriter, r *http.Request) {
	var body []byte
	if r.Body != nil {
		if data, err := ioutil.ReadAll(r.Body); err == nil {
			body = data
		}
	}
	if len(body) == 0 {
		http.Error(w, "empty body", http.StatusBadRequest)
		return
	}
	contentType := r.Header.Get("Content-Type")
	if contentType != "application/json" {
		http.Error(w, "invalid Content-Type, expect `application/json`", http.StatusUnsupportedMediaType)
		return
	}
	ar := &v1beta1.AdmissionReview{}
	var admissionResponse *v1beta1.AdmissionResponse
	if _, _, err := deserializer.Decode(body, nil, ar); err != nil {
		glog.Errorf("Can't decode body: %v", err)
		admissionResponse = &v1beta1.AdmissionResponse{
			Result: &metav1.Status{
				Message: err.Error(),
			},
		}
	} else {
		admissionResponse = webhooks.Validate(ar) // validation
	}
	ar.Response = admissionResponse
	resp, _ := json.Marshal(ar)
	w.Write(resp)
}

func (ws *WebhookServer) mutate(w http.ResponseWriter, r *http.Request) {
	var body []byte
	if r.Body != nil {
		if data, err := ioutil.ReadAll(r.Body); err == nil {
			body = data
		}
	}
	if len(body) == 0 {
		http.Error(w, "empty body", http.StatusBadRequest)
		return
	}
	contentType := r.Header.Get("Content-Type")
	if contentType != "application/json" {
		http.Error(w, "invalid Content-Type, expect `application/json`", http.StatusUnsupportedMediaType)
		return
	}
	ar := &v1beta1.AdmissionReview{}
	var admissionResponse *v1beta1.AdmissionResponse
	if _, _, err := deserializer.Decode(body, nil, ar); err != nil {
		glog.Errorf("Can't decode body: %v", err)
		admissionResponse = &v1beta1.AdmissionResponse{
			Result: &metav1.Status{
				Message: err.Error(),
			},
		}
	} else {
		admissionResponse = webhooks.Mutate(ar) // mutate
	}
	ar.Response = admissionResponse
	resp, _ := json.Marshal(ar)
	w.Write(resp)
}

func main() {
	var parameters serverParams
	// 指定webhook server的端口，和使用的证书的位置
	flag.IntVar(&parameters.port, "port", 443, "Webhook server port.")
	flag.StringVar(&parameters.certFile, "tlsCertFile", "/etc/webhook/certs/cert.pem", "File containing the x509 Certificate for HTTPS.")
	flag.StringVar(&parameters.keyFile, "tlsKeyFile", "/etc/webhook/certs/key.pem", "File containing the x509 private key to --tlsCertFile.")
	flag.Parse()

	pair, err := tls.LoadX509KeyPair(parameters.certFile, parameters.keyFile)
	if err != nil {
		glog.Errorf("Failed to load key pair: %v", err)
	}

	whsvr := &WebhookServer{
		server: &http.Server{
			Addr:      fmt.Sprintf(":%v", parameters.port),
			TLSConfig: &tls.Config{Certificates: []tls.Certificate{pair}},
		},
	}

	// define http server and server handler
	mux := http.NewServeMux()
	mux.HandleFunc("/validate", whsvr.validate)
	mux.HandleFunc("/mutate", whsvr.mutate)
	mux.HandleFunc("/healthcheck", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("running"))
	}))
	whsvr.server.Handler = mux

	// start webhook server in new routine
	if err := whsvr.server.ListenAndServeTLS("", ""); err != nil {
		glog.Errorf("Failed to listen and serve webhook server: %v", err)
	}
}
```

把 validate 和 mutate 相关的逻辑都放在 webhooks 目录下，在 webhooks 目录下创建 `handle.go`, 用 Validate 和 Mutate 方法来统一处理相应的逻辑。
```go
// webhooks/handle.go
package webhooks

import (
	"encoding/json"
	"fmt"
	"strings"

	"github.com/golang/glog"

	"k8s.io/api/admission/v1beta1"
	apiv1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

type patchOperation struct {
	Op    string      `json:"op"`
	Path  string      `json:"path"`
	Value interface{} `json:"value,omitempty"`
}

func Validate(ar *v1beta1.AdmissionReview) *v1beta1.AdmissionResponse {
	req := ar.Request
	var resp *v1beta1.AdmissionResponse
	switch req.Kind.Kind {
	case "Service":
		switch req.Kind.Group {
		case "":
			s := &apiv1.Service{}
			if err := json.Unmarshal(req.Object.Raw, s); err != nil {
				glog.Errorf("Could not unmarshal raw object %v", err)
				return UnmarshalError(err)
			}
			sv := NewServiceValidator(s)
			resp = sv.validate()
		}
		// case "Pod": ....
	}
	if resp == nil {
		resp = &v1beta1.AdmissionResponse{
			Allowed: true,
		}
	}
	return resp
}

func Mutate(ar *v1beta1.AdmissionReview) *v1beta1.AdmissionResponse {
	req := ar.Request
	resp := &v1beta1.AdmissionResponse{}
	switch req.Kind.Kind {
	case "Service":
		sv, err := NewServiceManager(req)
		if err != nil {
			return UnmarshalError(err)
		}
		resp = sv.Mutate()
	}
	return resp
}

func MakeAdmissionResponse(allowed bool, reason metav1.StatusReason) *v1beta1.AdmissionResponse {
	resp := new(v1beta1.AdmissionResponse)
	resp.Allowed = allowed
	resp.Result = &metav1.Status{
		Reason: reason,
	}
	return resp
}

func UnmarshalError(err error) *v1beta1.AdmissionResponse {
	errMsg := fmt.Sprintf("Cannot unmarshal raw objects from API server, %v", err)
	if strings.Contains(err.Error(), "AnalysisMessageBase_Level") {
		errMsg = "Cannot unmarshal the object due to Istio API issue. If you are creating a virtual service, please check if you provide the correct gateway in the manifest."
	}
	return MakeAdmissionResponse(false, metav1.StatusReason(errMsg))
}
```
现在基本的框架都搭好了。

## 开发 validate 和 mutate 逻辑

接下来在 webhooks 目录下实现相应的资源的 validate 和 mutate 逻辑就可以了， 这里我们要验证 service, 所以新建 service.go
```go
// webhooks/service.go
package webhooks

import (
	"encoding/json"
	"fmt"
	"strings"

	"github.com/golang/glog"
	"k8s.io/api/admission/v1beta1"
	apiv1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

const (
	SVC_NS_NAME_LENGTH = 56
)

type ServiceValidator struct {
	Service *apiv1.Service
}

type ServiceManager struct {
	service   *apiv1.Service
	user      string
	namespace string
}

func NewServiceManager(req *v1beta1.AdmissionRequest) (*ServiceManager, error) {
	sv := &ServiceManager{
		service:   &apiv1.Service{},
		user:      req.UserInfo.Username,
		namespace: req.Namespace,
	}
	if err := json.Unmarshal(req.Object.Raw, sv.service); err != nil {
		return nil, err
	}
	return sv, nil
}

func NewServiceValidator(s *apiv1.Service) *ServiceValidator {
	return &ServiceValidator{
		Service: s,
	}
}

func (sv *ServiceValidator) hasProtocolPrefix(name string) bool {
	i := strings.IndexByte(name, '-')
	if i >= 0 {
		name = name[:i]
	}
	return true
}

func (sv *ServiceValidator) validatePorts() *v1beta1.AdmissionResponse {
	for _, sp := range sv.Service.Spec.Ports {
		switch sp.Protocol {
		case apiv1.ProtocolUDP: // skip udp validate
			break
		}
		if sp.Port == 443 {
			if !strings.HasPrefix(sp.Name, "https") {
				return MakeAdmissionResponse(false,
					"You cannot configure a non-HTTPs service with 443 port")
			}
		}
	}
	return nil
}

func (sv *ServiceValidator) validate() *v1beta1.AdmissionResponse {
	if r := sv.validateLength(); r != nil {
		return r
	}
	if r := sv.validatePorts(); r != nil {
		return r
	}
	return nil
}

func makePatchOperation(verb, path string) *patchOperation {
	return &patchOperation{
		Op:   verb,
		Path: path,
	}
}

func (sv *ServiceManager) injectRequiredLabels() []*patchOperation {
	patch := []*patchOperation{}

	labels := sv.service.ObjectMeta.Labels
	existLabels := make(map[string]string)
	for k, v := range labels {
		existLabels[k] = v
	}
	existLabels["enabled"] = "yes" // add enabled label and set yes as default value

	labelPath := makePatchOperation("add", "/metadata/labels")
	labelPath.Value = existLabels
	patch = append(patch, labelPath)

	return patch
}

func (sv *ServiceManager) Mutate() *v1beta1.AdmissionResponse {
	resp := &v1beta1.AdmissionResponse{
		Allowed: true,
	}

	operations := []*patchOperation{}
	operations = append(operations, sv.injectRequiredLabels()...)

	finalPatch, _ := json.Marshal(operations)
	glog.Infof("Adding patch %s for service %s in namespace %s", string(finalPatch), sv.service.ObjectMeta.Name, sv.namespace)
	resp.Patch = finalPatch
	resp.PatchType = func() *v1beta1.PatchType {
		pt := v1beta1.PatchTypeJSONPatch
		return &pt
	}()

	return resp
}
```
这样代码基本就写完了，剩下的是 webook 的部署
## push 镜像到仓库

首先来准备 Dockerfile:
```Dockerfile
FROM golang:1.23 AS build_base

RUN mkdir -p /opt/src/my-webhook
WORKDIR /opt/src/my-webhook
COPY go.mod .
COPY go.sum .
RUN go mod download

# go binary build
FROM build_base AS binary_builder
WORKDIR /opt/src/my-webhook
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /go/bin/my-webhook main.go


# final image, direct copy binary build result to a simple base image
FROM alpine
COPY --from=binary_builder /go/bin/my-webhook /bin/

# 使用了ENTRYPOINT就不能像CMD那样运行容器时自定义执行的命令了
ENTRYPOINT [ "my-webhook" ] 
```

build 好后把镜像给传到 dockerhub 上：
```shell
docker login -u f91org # login dockerhub
docker build -t f91org/simple-k8s-webhook .
docker push f91org/simple-k8s-webhook
```

## 准备 https 通信所需的证书

webook 和 api server 是 https 通信，所以需要我们准备 webook 端的私钥以及签发它的证书，这里做法有 2 种：
1. 用 k8s 集群的 CA，通过 `CertificateSigningRequest` 和 `kubectl certificate approve` 来签发证书。
2. 使用 `openssl` 命令来生成自签发的 CA，区别于 k8s 集群的 CA，这种情况下需要在 `WebhookConfiguration` 里来配置自签发 CA 的证书。
这里用第二种方法，即 openssl 来生成自签发 CA，用这个 CA 去签发证书。
```shell
# 生成CA的证书和私钥
openssl genrsa -out ca.key 2048
openssl req -x509 -new -nodes -key ca.key -subj "/CN=ca" -days 3650 -out ca.crt

# 再生成一个私钥和以及它的证书签名请求（CSR）
openssl genrsa -out webhook.key 2048
openssl req -new -key webhook.key -subj "/CN=my-webhook.default.svc" \
    -reqexts v3_req \
    -config <(cat <<EOF
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req

[req_distinguished_name]

[v3_req]
subjectAltName = @alt_names

[alt_names]
DNS.1 = my-webhook.default.svc
DNS.2 = my-webhook.default.svc.cluster.local
DNS.3 = *.default.svc
EOF
) -out webhook.csr

# 使用CA的key和证书签署CSR `webhook.csr`，得到由CA签署的和私钥对应的证书
openssl x509 -req -in webhook.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
    -out webhook.crt -days 365 -extensions v3_req \
    -extfile <(cat <<EOF
[req]
distinguished_name = req_distinguished_name

[v3_req]
subjectAltName = @alt_names

[alt_names]
DNS.1 = my-webhook.default.svc
DNS.2 = my-webhook.default.svc.cluster.local
EOF
)
```
后面在 yaml 文件中会用到 `ca.crt`, `wehbook.key` 和 `webhook.crt` 

## 部署 yaml 文件

然后是部署用的 yaml 文件，service, deployment, secret, sa, clusterrole, rolebinding 全套走起，以及 adminssionwebhook 需要用到的 2 个特殊的配置 ValidatingWebhookConfiguration，MutatingWebhookConfiguration
deploy 和 svc：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-webhook
  labels:
    app: my-webhook
spec:
  ports:
  - port: 443
    targetPort: 443
    name: https-my-webhook
  selector:
    app: my-webhook
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-webhook
  labels:
    app: my-webhook
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-webhook
  template:
    metadata:
      labels:
        app: my-webhook
    spec:
      automountServiceAccountToken: true
      containers:
        - name: my-webhook
          image: f91org/simple-k8s-webhook
          imagePullPolicy: IfNotPresent
          args:
            - -tlsCertFile=/etc/webhook/certs/cert.pem 
            - -tlsKeyFile=/etc/webhook/certs/key.pem
            - -alsologtostderr
            - -v=4
            - 2>&1
          volumeMounts:
            - name: webhook-certs
              mountPath: /etc/webhook/certs
              readOnly: true
          livenessProbe:
            httpGet:
              path: /healthcheck
              scheme: HTTPS
              port: 443
            initialDelaySeconds: 3
            periodSeconds: 3
          readinessProbe:
            httpGet:
              path: /healthcheck
              scheme: HTTPS
              port: 443
            initialDelaySeconds: 3
            periodSeconds: 3
      volumes:
        - name: webhook-certs
          secret:
            secretName: webhook-certs
```

将证书和私钥转换为 secret 所需要的 Base64 编码格式放在 secret 里：
```yaml
apiVersion: v1
data:
  cert.pem: LS0tLS1CRUdJTiBDRVJU... # cat webhook.crt | base64
  key.pem: LS0tLS1CRUdJTiBQUklW... # cat webhook.key | base64
kind: Secret
metadata:
  name: webhook-certs
```

然后是权限相关配置，role -> rolebinding <- sa，这里为了方便直接给所有权限：
```yaml
# CluserRole
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-admin
rules:
- apiGroups: ['*']
  resources: ['*']
  verbs: ['*']
---
# CluserRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: default
  namespace: default

# sa 用默认的就可以
```

最后是 ValidatingWebhookConfiguration 和 MutatingWebhookConfiguration，主要配置要对哪些资源进行验证和修改，这里需要配置 CA 的 cert，还是一样的，将之前生成的 CA 的证书编码为 base64 来放到配置里：
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: my-webhook
webhooks:
  - name: com.f91og.webhook  # 命名有要求为至少有3个点
    sideEffects: NoneOnDryRun
    failurePolicy: Fail
    objectSelector:
      matchExpressions:
      - key: "app"
        values:
        - "my-webhook"
        operator: NotIn
    admissionReviewVersions: ["v1"]
    clientConfig:
      service:
        name: my-webhook
        path: "/mutate"
      caBundle: LS0tLS1CRUdJTiBDRVJU... # cat ca.crt | base64
    rules:
    - operations: ["CREATE", "UPDATE"]
      apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["services", "pods"]
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: my-webhook
webhooks:
- name: com.f91og.webhook
  rules:
  - apiGroups: ["apps", ""]
    apiVersions: ["v1"]
    operations: ["CREATE", "UPDATE"]
    resources: ["services"]
  sideEffects: None
  failurePolicy: Fail
  admissionReviewVersions: ["v1"]
  objectSelector:
    matchExpressions:
    - key: "app"
      values:
      - "my-webhook"
      operator: NotIn
  clientConfig:
    caBundle: LS0tLS1CRUdJTiBDRVJU... # 太长省略
    service:
      name: my-webhook
      path: /validate
```

这样就部署完成了，最后是验证，尝试 deploy 了一个 service，在 webhook 的 pod 的日志里发现了：
```txt
service.go:124] Adding patch [{"op":"add","path":"/metadata/labels","value":{"enabled":"yes"}}] for service test-svc in namespace default
```
证明 webhook 已将开始工作了。
## 用 helm 来生成和管理证书和密钥

全套流程部署下来后，有一个麻烦的地方是证书这种敏感的数据不应该明文放在 yaml 里，这个时候可以用 helm 的模版以及内置的自签发证书生成函数 `genSelfSignedCert` 来动态渲染这个证书数据，这样就可以把 webhook 相关的 yaml 文件放心的存放在远程仓库了，大概就是这样：
```yaml
{{- $cn := printf "%s.%s.svc" .Values.service .Values.namespace }}
{{- $altName1 := printf "%s" .Values.service }}
{{- $altName2 := printf "%s.%s" .Values.service .Values.namespace }}
{{- $altName3 := printf "%s.%s.svc" .Values.service .Values.namespace }}
{{- $ca := genSelfSignedCert $cn nil (list $altName1 $altName2 $altName3) 5114 }}
apiVersion: v1
data:
  cert.pem: {{ ternary (b64enc (trim $ca.Cert)) (b64enc (trim .Values.webhook.crtPEM)) (empty .Values.webhook.crtPEM) }}
  key.pem: {{ ternary (b64enc (trim $ca.Key)) (b64enc (trim .Values.webhook.keyPEM)) (empty .Values.webhook.keyPEM) }}
kind: Secret
metadata:
  namespace: {{ .Values.namespace }}
  creationTimestamp: null
  name: {{ .Values.secret }}
```

