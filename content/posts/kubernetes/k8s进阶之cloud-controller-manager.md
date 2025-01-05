+++
title = 'k8s 进阶之 cloud-controller-manager'
date = 2025-01-02T09:35:28Z
draft = false
tags = ['k8s']
categories = ['Kubernetes']
+++

Cloud Controller Manager（CCM）是 Kubernetes 中控制平面的一个组件，专门用于与云服务提供商的基础设施交互，通过这个可以定制云厂商自己 node, lb svc, route 和 volume 的控制逻辑。

<!--more-->

看官网的介绍：[云控制器管理器 | Kubernetes](https://kubernetes.io/zh-cn/docs/concepts/architecture/cloud-controller/)
![](https://kubernetes.io/zh-cn/docs/images/components-of-kubernetes.svg)
可以实现的功能是：
- **节点控制（Node Controller）**
    检测节点是否被云环境删除，例如当云服务提供商的实例被删除时，自动从 Kubernetes 集群中移除对应的节点。
- **路由控制（Route Controller）**
	配置云提供商的路由表以实现 Kubernetes 集群中 Pod 的网络通信。
- **服务控制（Service Controller）**
    管理与云负载均衡器的交互，根据 Kubernetes Service 类型（LoadBalancer），在云环境中自动创建、更新或删除负载均衡器。
- **卷控制（Volume Controller）**
    管理云存储卷的生命周期，例如动态创建和删除与 Kubernetes PersistentVolume 相关联的存储。

上面这 4 个资源和具体的云服务厂商的实现有关，不属于 k8s 自己的管理范畴，如云负载均衡器。
CCM 将云相关的逻辑从 Kubernetes 控制平面的其他组件中分离出来（如 `kube-controller-manager` 和 `kubelet`），使得 Kubernetes 的核心组件能够保持**与云无关的通用性**。这种设计使 Kubernetes 更容易支持多种云服务提供商。

**使用场景**
- **多云支持**：通过 CCM，Kubernetes 可以在不同云服务提供商上以一致的方式运行。
- **自动化资源管理**：CCM 能够自动化处理云资源的创建、配置和清理，减少了运维负担。
- **扩展性**：云提供商可以通过实现自己的 CCM 插件来支持其特定功能。

在集群部署时，CCM 通常作为一个独立的进程运行，与 `kube-controller-manager`、`kube-scheduler` 等其他控制平面组件并列。它依赖 Kubernetes API Server 提供的接口，与云服务提供商的 API 进行交互。

开发 CCM 的流程和之前的 webhook 差不多，部署一个 deploy 到集群里，这里就不从 0 开始开发了，直接来分析组里的代码。

首先是 `main.go` 
```go
// ./main.go
package main

import (
	"fmt"
	"math/rand"
	"os"
	"time"

	_ "f91og.com/mycloud/pkg"
	_ "k8s.io/client-go/plugin/pkg/client/auth"
	"k8s.io/component-base/logs"
	_ "k8s.io/component-base/metrics/prometheus/clientgo" // load all the prometheus client-go plugins
	_ "k8s.io/component-base/metrics/prometheus/version"  // for version metric registration
	"k8s.io/klog"
	"k8s.io/kubernetes/cmd/cloud-controller-manager/app"
)

func main() {
	rand.Seed(time.Now().UnixNano())

	command := app.NewCloudControllerManagerCommand()

	logs.InitLogs()
	defer logs.FlushLogs()

	if err := command.Execute(); err != nil {
		klog.Error(fmt.Fprintf(os.Stderr, "error: %v\n", err))
		os.Exit(1)
	}
}
```
通过 `command := app.NewCloudControllerManagerCommand()` 来初始化 ccm 的启动命令，然后部署后作为 app 运行

然后是如何注册 cloud provider，这个通过 import 处的 `_ "f91og.com/mycloud/pkg"` 这里来实现。
```go
// ./pkg/mycloud.go
package pkg

import (
	"context"
	"errors"
	"fmt"
	"io"

	v1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/types"
	cloudprovider "k8s.io/cloud-provider"
	"k8s.io/klog"

	"net"
	"regexp"
	"strings"
	"sync"
	"time"
)

const (
	ProviderName = "mycloud"
)

var (
	version string
)

// 在这个个方法里注册我们自己的cloud provider
func init() {
	cloudprovider.RegisterCloudProvider(ProviderName, func(config io.Reader) (cloudprovider.Interface, error) {
		klog.Info("register cloud provider:" + ProviderName)
		klog.Info("roc ccm version:" + version)
		return newMyCloud(config)
	})
}

func newMyCloud(configReader io.Reader) (cloudprovider.Interface, error) {
	var c MyCloud
	config, err := parseConfig(configReader)
	if err != nil {
		return nil, err
	}
	if config == nil {
		return nil, nil
	}

	c.Config = config
	return &c, nil
}

type MyCloud struct {
	//.............
}
......
```
在上面的 `init()` 方法里注册自己的 cloud provider，而 init 方法会在所有的方法之前运行，从而在 `app.NewCloudControllerManagerCommand()` 开始前将 ccm 相关的控制逻辑注册好。

接下来就是 ccm 具体的对节点，路由，服务和卷的这 4 种资源的控制逻辑，这个是通过我们自己的 cloud provider 来实现指定接口来实现的，在上面的代码就是 `MyCloud` 实现接口 `cloudprovider.Interface`，看了下这个接口里声明的内容：
```go
type Interface interface {
	// Initialize provides the cloud with a kubernetes client builder and may spawn goroutines
	// to perform housekeeping or run custom controllers specific to the cloud provider.
	// Any tasks started here should be cleaned up when the stop channel closes.
	Initialize(clientBuilder ControllerClientBuilder, stop <-chan struct{})
	// LoadBalancer returns a balancer interface. Also returns true if the interface is supported, false otherwise.
	LoadBalancer() (LoadBalancer, bool)
	// Instances returns an instances interface. Also returns true if the interface is supported, false otherwise.
	Instances() (Instances, bool)
	// Zones returns a zones interface. Also returns true if the interface is supported, false otherwise.
	Zones() (Zones, bool)
	// Clusters returns a clusters interface.  Also returns true if the interface is supported, false otherwise.
	Clusters() (Clusters, bool)
	// Routes returns a routes interface along with whether the interface is supported.
	Routes() (Routes, bool)
	// ProviderName returns the cloud provider ID.
	ProviderName() string
	// HasClusterID returns true if a ClusterID is required and set
	HasClusterID() bool
}
```
其中除了 `Initialize`, `ProviderName`, `HasClusterID` 其他的几个要返回的都是接口，那么在 pkg 中的实现就是给我们自己的 `myCloud` 结构体实现上面 `LoadBalancer`, `Instances`, `Zones`, `Clusters`, `Routes` 这些接口中声明的所有方法，再直接返回就可以了：
```go
// ./pkg/mycloud.go

// Initialize passes a Kubernetes clientBuilder interface to the cloud provider
func (c *MyCloud) Initialize(clientBuilder cloudprovider.ControllerClientBuilder, stop <-chan struct{}) {
}

// HasClusterID returns true if the cluster has a clusterID
func (c *MyCloud) HasClusterID() bool {
	return true
}

// ProviderName returns the cloud provider ID.
func (c *MyCloud) ProviderName() string {
	if c.Provider == "" {
		return ProviderName
	}
	return c.Provider
}

// LoadBalancer returns a CPD implementation of LoadBalancer.
// Actually it just returns f itself.
func (c *MyCloud) LoadBalancer() (cloudprovider.LoadBalancer, bool) {
	c.LBClientSet = c.NewLBClientset(c.Config.LoadBalancer.Dlb.Version)
	return c, true
}

// Instances returns a CPD implementation of Instances.
// Actually it just returns f itself.
func (c *MyCloud) Instances() (cloudprovider.Instances, bool) {
	return c, true
}

func (c *MyCloud) Zones() (cloudprovider.Zones, bool) {
	return c, false
}

func (c *MyCloud) Routes() (cloudprovider.Routes, bool) {
	return c, false
}

func (c *MyCloud) Clusters() (cloudprovider.Clusters, bool) {
	return c, true
}
```

MyCloud 要实现 `cloudprovider` 中 `LoadBalancer`, `Instances`, `Zones`, `Clusters`, `Routes` 接口声明的所有方法：
```go
package cloudprovider

import (
	"context"
	"errors"
	"fmt"
	"strings"

	"k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/types"
	"k8s.io/client-go/informers"
	clientset "k8s.io/client-go/kubernetes"
	restclient "k8s.io/client-go/rest"
)

// ControllerClientBuilder allows you to get clients and configs for controllers
// Please note a copy also exists in pkg/controller/client_builder.go
// TODO: Make this depend on the separate controller utilities repo (issues/68947)
type ControllerClientBuilder interface {
	Config(name string) (*restclient.Config, error)
	ConfigOrDie(name string) *restclient.Config
	Client(name string) (clientset.Interface, error)
	ClientOrDie(name string) clientset.Interface
}

// Interface is an abstract, pluggable interface for cloud providers.
type Interface interface {
	// Initialize provides the cloud with a kubernetes client builder and may spawn goroutines
	// to perform housekeeping or run custom controllers specific to the cloud provider.
	// Any tasks started here should be cleaned up when the stop channel closes.
	Initialize(clientBuilder ControllerClientBuilder, stop <-chan struct{})
	// LoadBalancer returns a balancer interface. Also returns true if the interface is supported, false otherwise.
	LoadBalancer() (LoadBalancer, bool)
	// Instances returns an instances interface. Also returns true if the interface is supported, false otherwise.
	Instances() (Instances, bool)
	// Zones returns a zones interface. Also returns true if the interface is supported, false otherwise.
	Zones() (Zones, bool)
	// Clusters returns a clusters interface.  Also returns true if the interface is supported, false otherwise.
	Clusters() (Clusters, bool)
	// Routes returns a routes interface along with whether the interface is supported.
	Routes() (Routes, bool)
	// ProviderName returns the cloud provider ID.
	ProviderName() string
	// HasClusterID returns true if a ClusterID is required and set
	HasClusterID() bool
}

type InformerUser interface {
	// SetInformers sets the informer on the cloud object.
	SetInformers(informerFactory informers.SharedInformerFactory)
}

// Clusters is an abstract, pluggable interface for clusters of containers.
type Clusters interface {
	// ListClusters lists the names of the available clusters.
	ListClusters(ctx context.Context) ([]string, error)
	// Master gets back the address (either DNS name or IP address) of the master node for the cluster.
	Master(ctx context.Context, clusterName string) (string, error)
}

// (DEPRECATED) DefaultLoadBalancerName is the default load balancer name that is called from
// LoadBalancer.GetLoadBalancerName. Use this method to maintain backward compatible names for
// LoadBalancers that were created prior to Kubernetes v1.12. In the future, each provider should
// replace this method call in GetLoadBalancerName with a provider-specific implementation that
// is less cryptic than the Service's UUID.
func DefaultLoadBalancerName(service *v1.Service) string {
	//GCE requires that the name of a load balancer starts with a lower case letter.
	ret := "a" + string(service.UID)
	ret = strings.Replace(ret, "-", "", -1)
	//AWS requires that the name of a load balancer is shorter than 32 bytes.
	if len(ret) > 32 {
		ret = ret[:32]
	}
	return ret
}

// GetInstanceProviderID builds a ProviderID for a node in a cloud.
func GetInstanceProviderID(ctx context.Context, cloud Interface, nodeName types.NodeName) (string, error) {
	instances, ok := cloud.Instances()
	if !ok {
		return "", fmt.Errorf("failed to get instances from cloud provider")
	}
	instanceID, err := instances.InstanceID(ctx, nodeName)
	if err != nil {
		if err == NotImplemented {
			return "", err
		}

		return "", fmt.Errorf("failed to get instance ID from cloud provider: %v", err)
	}
	return cloud.ProviderName() + "://" + instanceID, nil
}

// LoadBalancer is an abstract, pluggable interface for load balancers.
//
// Cloud provider may chose to implement the logic for
// constructing/destroying specific kinds of load balancers in a
// controller separate from the ServiceController.  If this is the case,
// then {Ensure,Update}LoadBalancer must return the ImplementedElsewhere error.
// For the given LB service, the GetLoadBalancer must return "exists=True" if
// there exists a LoadBalancer instance created by ServiceController.
// In all other cases, GetLoadBalancer must return a NotFound error.
// EnsureLoadBalancerDeleted must not return ImplementedElsewhere to ensure
// proper teardown of resources that were allocated by the ServiceController.
// This can happen if a user changes the type of LB via an update to the resource
// or when migrating from ServiceController to alternate implementation.
// The finalizer on the service will be added and removed by ServiceController
// irrespective of the ImplementedElsewhere error. Additional finalizers for
// LB services must be managed in the alternate implementation.
type LoadBalancer interface {
	// TODO: Break this up into different interfaces (LB, etc) when we have more than one type of service
	// GetLoadBalancer returns whether the specified load balancer exists, and
	// if so, what its status is.
	// Implementations must treat the *v1.Service parameter as read-only and not modify it.
	// Parameter 'clusterName' is the name of the cluster as presented to kube-controller-manager
	GetLoadBalancer(ctx context.Context, clusterName string, service *v1.Service) (status *v1.LoadBalancerStatus, exists bool, err error)
	// GetLoadBalancerName returns the name of the load balancer. Implementations must treat the
	// *v1.Service parameter as read-only and not modify it.
	GetLoadBalancerName(ctx context.Context, clusterName string, service *v1.Service) string
	// EnsureLoadBalancer creates a new load balancer 'name', or updates the existing one. Returns the status of the balancer
	// Implementations must treat the *v1.Service and *v1.Node
	// parameters as read-only and not modify them.
	// Parameter 'clusterName' is the name of the cluster as presented to kube-controller-manager
	EnsureLoadBalancer(ctx context.Context, clusterName string, service *v1.Service, nodes []*v1.Node) (*v1.LoadBalancerStatus, error)
	// UpdateLoadBalancer updates hosts under the specified load balancer.
	// Implementations must treat the *v1.Service and *v1.Node
	// parameters as read-only and not modify them.
	// Parameter 'clusterName' is the name of the cluster as presented to kube-controller-manager
	UpdateLoadBalancer(ctx context.Context, clusterName string, service *v1.Service, nodes []*v1.Node) error
	// EnsureLoadBalancerDeleted deletes the specified load balancer if it
	// exists, returning nil if the load balancer specified either didn't exist or
	// was successfully deleted.
	// This construction is useful because many cloud providers' load balancers
	// have multiple underlying components, meaning a Get could say that the LB
	// doesn't exist even if some part of it is still laying around.
	// Implementations must treat the *v1.Service parameter as read-only and not modify it.
	// Parameter 'clusterName' is the name of the cluster as presented to kube-controller-manager
	EnsureLoadBalancerDeleted(ctx context.Context, clusterName string, service *v1.Service) error
}

// Instances is an abstract, pluggable interface for sets of instances.
type Instances interface {
	// NodeAddresses returns the addresses of the specified instance.
	// TODO(roberthbailey): This currently is only used in such a way that it
	// returns the address of the calling instance. We should do a rename to
	// make this clearer.
	NodeAddresses(ctx context.Context, name types.NodeName) ([]v1.NodeAddress, error)
	// NodeAddressesByProviderID returns the addresses of the specified instance.
	// The instance is specified using the providerID of the node. The
	// ProviderID is a unique identifier of the node. This will not be called
	// from the node whose nodeaddresses are being queried. i.e. local metadata
	// services cannot be used in this method to obtain nodeaddresses
	NodeAddressesByProviderID(ctx context.Context, providerID string) ([]v1.NodeAddress, error)
	// InstanceID returns the cloud provider ID of the node with the specified NodeName.
	// Note that if the instance does not exist, we must return ("", cloudprovider.InstanceNotFound)
	// cloudprovider.InstanceNotFound should NOT be returned for instances that exist but are stopped/sleeping
	InstanceID(ctx context.Context, nodeName types.NodeName) (string, error)
	// InstanceType returns the type of the specified instance.
	InstanceType(ctx context.Context, name types.NodeName) (string, error)
	// InstanceTypeByProviderID returns the type of the specified instance.
	InstanceTypeByProviderID(ctx context.Context, providerID string) (string, error)
	// AddSSHKeyToAllInstances adds an SSH public key as a legal identity for all instances
	// expected format for the key is standard ssh-keygen format: <protocol> <blob>
	AddSSHKeyToAllInstances(ctx context.Context, user string, keyData []byte) error
	// CurrentNodeName returns the name of the node we are currently running on
	// On most clouds (e.g. GCE) this is the hostname, so we provide the hostname
	CurrentNodeName(ctx context.Context, hostname string) (types.NodeName, error)
	// InstanceExistsByProviderID returns true if the instance for the given provider exists.
	// If false is returned with no error, the instance will be immediately deleted by the cloud controller manager.
	// This method should still return true for instances that exist but are stopped/sleeping.
	InstanceExistsByProviderID(ctx context.Context, providerID string) (bool, error)
	// InstanceShutdownByProviderID returns true if the instance is shutdown in cloudprovider
	InstanceShutdownByProviderID(ctx context.Context, providerID string) (bool, error)
}

// Route is a representation of an advanced routing rule.
type Route struct {
	// Name is the name of the routing rule in the cloud-provider.
	// It will be ignored in a Create (although nameHint may influence it)
	Name string
	// TargetNode is the NodeName of the target instance.
	TargetNode types.NodeName
	// DestinationCIDR is the CIDR format IP range that this routing rule
	// applies to.
	DestinationCIDR string
	// Blackhole is set to true if this is a blackhole route
	// The node controller will delete the route if it is in the managed range.
	Blackhole bool
}

// Routes is an abstract, pluggable interface for advanced routing rules.
type Routes interface {
	// ListRoutes lists all managed routes that belong to the specified clusterName
	ListRoutes(ctx context.Context, clusterName string) ([]*Route, error)
	// CreateRoute creates the described managed route
	// route.Name will be ignored, although the cloud-provider may use nameHint
	// to create a more user-meaningful name.
	CreateRoute(ctx context.Context, clusterName string, nameHint string, route *Route) error
	// DeleteRoute deletes the specified managed route
	// Route should be as returned by ListRoutes
	DeleteRoute(ctx context.Context, clusterName string, route *Route) error
}

var (
	DiskNotFound         = errors.New("disk is not found")
	ImplementedElsewhere = errors.New("implemented by alternate to cloud provider")
	InstanceNotFound     = errors.New("instance not found")
	NotImplemented       = errors.New("unimplemented")
)

// Zone represents the location of a particular machine.
type Zone struct {
	FailureDomain string
	Region        string
}

// Zones is an abstract, pluggable interface for zone enumeration.
type Zones interface {
	// GetZone returns the Zone containing the current failure zone and locality region that the program is running in
	// In most cases, this method is called from the kubelet querying a local metadata service to acquire its zone.
	// For the case of external cloud providers, use GetZoneByProviderID or GetZoneByNodeName since GetZone
	// can no longer be called from the kubelets.
	GetZone(ctx context.Context) (Zone, error)

	// GetZoneByProviderID returns the Zone containing the current zone and locality region of the node specified by providerID
	// This method is particularly used in the context of external cloud providers where node initialization must be done
	// outside the kubelets.
	GetZoneByProviderID(ctx context.Context, providerID string) (Zone, error)

	// GetZoneByNodeName returns the Zone containing the current zone and locality region of the node specified by node name
	// This method is particularly used in the context of external cloud providers where node initialization must be done
	// outside the kubelets.
	GetZoneByNodeName(ctx context.Context, nodeName types.NodeName) (Zone, error)
}

// PVLabeler is an abstract, pluggable interface for fetching labels for volumes
type PVLabeler interface {
	GetLabelsForVolume(ctx context.Context, pv *v1.PersistentVolume) (map[string]string, error)
}
```
通过实现以上的方法就可以定制我们自己的 node, lb, route, volume 的控制逻辑。

现在组里通过 ccm 实现的是：
- 自动给 lb 配置名字，规则为 `${CLUSTER_ID}-${NAMESPACE}-${SERVICE_NAME}`，lb 是 k8s 集群外的资源，和 lb svc 不同有自己的命名。
- 通过在 lb svc 中加入注解自动配置 lb，但创建类型为 lb 的 svc 时，ccm 会读取 svc 中的注解，比如 `service.beta.kubernetes.io/dlb-listener-maxqps: "100000"`， 然后 call 专门管 lb 的 team 的 api 来修改负责均衡器的配置。
- 自动配置负责均衡器的下游 server，通过给 node 添加注解让 k8s 集群中的一些 node 承接 ingress 流量 (ingress node)。
- 设置路由，这个不太清楚怎么弄的，有时间可以研究一下。