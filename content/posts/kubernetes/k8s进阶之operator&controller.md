+++
title = 'k8s进阶之 operator&controller'
date = 2024-09-07T06:32:46Z
draft = false
tags = ['k8s']
categories = ['Kubernetes']
+++

Operator 是一种自动化模式的概念，而 Controller 是执行这一模式的实际工具。

<!--more-->
## operator和controller简介

operator 和 controller 这2个概念经常在 k8s 中一起听到，可以简单的理解为 operator 是一种对如何管理复杂应用的生命周期的抽象，类似于一套理论来阐述如何编排容器，而 controller 就是实际执行管理操作的“工人”，类似于容器编排工具中的 k8s。

就实际层面来说 controller 就是 pod，让其来 watch 资源的现有状态，并通过 reconcile 将这些资源变成期望的状态，从而达到自动运维的效果。

现在公司的一个使用场景是当一个新的tenant来使用我们的platform，创建 tenanat 的 namespace的时候，我们希望相关的 k8s 资源（resourcequota, rbac, pdb等）能够自动被创建，并且可以触发 jenkins job 来设置 tenant 的 pipeline, slack channel 等。
## 开发 controller
一般会使用 [operator-sdk](https://github.com/operator-framework/operator-sdk) 这个框架来开发 k8s operator，接下来我们用这个框架来创建一个非常简单的 operator，当crd MyNamespace被创建的时候，自动创建同名的 namesapce 并在这个 namespace 下面创建默认的 resourcequota。

**安装 Operator SDK**
```shell
brew install operator-sdk  # MacOS
```

**项目初始化**
```shell
mkdir my-operator && cd my-operator
go mod init f91og.com
operator-sdk init
```
执行完上面的 init 后会在项目下面生成很多目录和文件，很多都不用管。

**创建 API 和 Controller**
```shell
operator-sdk create api --group mygroup --version v1 --kind MyNamespace --resource --controller
```
指定 gvk （k8s 的资源的定义由3部分组成，group，version，kind）就可以来指定我们需要让这个controller所管理的资源。这里我们用自定义资源（crd） MyNamespace 来实践 operator。  
operator会自动生成 crd 的yaml 文件，不需要我们提前创建 crd 的 yaml 文件。
上面的命令执行完成后命令行会提示用 `make manifests` 来生成 yaml 文件，如果想要修改crd，可以在 `api/v1/mynamespace_types.go` 文件中定义 crd 的结构：
```go
// MyNamespaceSpec defines the desired state of MyNamespace
type MyNamespaceSpec struct {
    // 添加你需要的字段
    DisplayName string `json:"displayName,omitempty"`
    // 其他你想管理的字段
}

// MyNamespaceStatus defines the observed state of MyNamespace
type MyNamespaceStatus struct {
    // 添加状态字段，例如创建状态
    Phase string `json:"phase,omitempty"`
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status

// MyNamespace is the Schema for the mynamespaces API
type MyNamespace struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   MyNamespaceSpec   `json:"spec,omitempty"`
    Status MyNamespaceStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true

// MyNamespaceList contains a list of MyNamespace
type MyNamespaceList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []MyNamespace `json:"items"`
}
```
然后来生成yaml并deploy：
```shell
make manifests

# 生成的yaml文件在目录下的config/crd/bases/中
kubectl apply -f config/crd/bases/mygroup.example.com_mynamespaces.yaml
```
\
**编写控制器逻辑**  
在生成的 `internal/controllers/mynamespace_controller.go` 文件中编写逻辑来处理 `MyNamespace` 的创建、更新、删除等事件，主要是实现 Reconcile 方法里的逻辑。
```go
package controller

import (
	"context"
	"fmt"

	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/errors"
	"k8s.io/apimachinery/pkg/api/resource"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/log"

	mygroupv1 "f91og.com/api/v1"
)

// MyNamespaceReconciler reconciles a MyNamespace object
type MyNamespaceReconciler struct {
	client.Client
	Scheme *runtime.Scheme
}

//+kubebuilder:rbac:groups=mygroup.my.domain,resources=mynamespaces,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=mygroup.my.domain,resources=mynamespaces/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=mygroup.my.domain,resources=mynamespaces/finalizers,verbs=update

// Reconcile is part of the main kubernetes reconciliation loop which aims to
// move the current state of the cluster closer to the desired state.
// TODO(user): Modify the Reconcile function to compare the state specified by
// the MyNamespace object against the actual cluster state, and then
// perform operations to make the cluster state reflect the state specified by
// the user.
//
// For more details, check Reconcile and its Result here:
// - https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.17.3/pkg/reconcile
func (r *MyNamespaceReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	logger := log.FromContext(ctx)

	// 获取 MyNamespace 资源
	myNamespace := &mygroupv1.MyNamespace{}
	err := r.Client.Get(ctx, req.NamespacedName, myNamespace)
	if err != nil {
		if errors.IsNotFound(err) {
			logger.Info("MyNamespace resource not found, ignoring since object must be deleted")
			return ctrl.Result{}, nil
		}
		logger.Error(err, "Failed to get MyNamespace")
		return ctrl.Result{}, err
	}

	err = r.createNamespaceAndQuota(ctx, myNamespace.Name)
	if err != nil {
		logger.Error(err, "Failed to create Namespace or ResourceQuota")
		return ctrl.Result{}, err
	}

	logger.Info(fmt.Sprintf("Reconciled MyNamespace: %s", myNamespace.Name))

	return ctrl.Result{}, nil
}

// 创建 Namespace 和默认 ResourceQuota 的函数
func (r *MyNamespaceReconciler) createNamespaceAndQuota(ctx context.Context, name string) error {
	namespace := &corev1.Namespace{
		ObjectMeta: metav1.ObjectMeta{
			Name: name,
		},
	}
	err := r.Client.Create(ctx, namespace)
	if err != nil && !errors.IsAlreadyExists(err) {
		return err
	}

	resourceQuota := &corev1.ResourceQuota{
		ObjectMeta: metav1.ObjectMeta{
			Name:      "default-quota",
			Namespace: name,
		},
		Spec: corev1.ResourceQuotaSpec{
			Hard: corev1.ResourceList{
				corev1.ResourceCPU:    resource.MustParse("0"),
				corev1.ResourceMemory: resource.MustParse("0Gi"),
				corev1.ResourcePods:   resource.MustParse("0"),
			},
		},
	}
	err = r.Client.Create(ctx, resourceQuota)
	if err != nil && !errors.IsAlreadyExists(err) {
		return err
	}

	return nil
}

// SetupWithManager sets up the controller with the Manager.
func (r *MyNamespaceReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&mygroupv1.MyNamespace{}).
		Complete(r)
}
```
\
**部署并测试operator**  
部署看 operator-sdk 在项目中生成的 `README` 就可以了。
```shell
# 创建本地集群用于测试
kind create cluster

# 部署 CRD
make install

# 构建 controller 的镜像并推送到镜像仓库
make docker-build IMG=<your-image-name>
make docker-push IMG=<your-image-name> # 要先在dockerhub上登陆创建自己的repo

# 部署
make deploy IMG=<your-image-name>

# 查看controller pod的运行状态
kg pods -n my-operator-system

NAME                                              READY   STATUS    RESTARTS   AGE
my-operator-controller-manager-66bb65c767-x7mrq   2/2     Running   0          76s
```
\
最后来测试下这个 controller 的行为，deploy一个 `MyNamespace` 资源：
```yaml
# my-namespace1.yaml
apiVersion: mygroup.my.domain/v1
kind: MyNamespace
metadata:
  name: my-ns1
spec:
# 用 operator-sdk 默认生成的crd里只有这个字段，定义在 api/v1/mynamespace_types.go 里
  foo: "example" 
```

发现没有自动创建同名的namesapce，尝试 log pods:
```shell
2024-09-06T08:15:50Z	ERROR	Reconciler error	{"controller": "mynamespace", "controllerGroup": "mygroup.my.domain", "controllerKind": "MyNamespace", "MyNamespace": {"name":"my-ns1","namespace":"default"}, "namespace": "default", "name": "my-ns1", "reconcileID": "91a8a4b4-0674-4358-990c-a7d8ac58f35b", "error": "namespaces is forbidden: User \"system:serviceaccount:my-operator-system:my-operator-controller-manager\" cannot create resource \"namespaces\" in API group \"\" at the cluster scope"}
```
缺少相应的权限，给 my-operator-controller-manager 这个 sa 绑定相应的 clusterRole 就可以了：
```yaml
# clusterRole-sa-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: ns-create-role
rules:
- apiGroups: [""]
  resources: ["namespaces", "resourcequotas"]
  verbs: ["create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ns-create-binding
subjects:
- kind: ServiceAccount
  name: my-operator-controller-manager
  namespace: my-operator-system  
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ns-create-role  
```

参考链接：
[【Docker】docker pushしたら denied: requested access to the resource is denied となった #DockerHub - Qiita](https://qiita.com/mihooo24/items/97791dde453529287770)
