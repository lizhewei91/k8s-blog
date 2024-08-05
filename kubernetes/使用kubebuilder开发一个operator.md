# Kubebuilder 介绍

[Kubebuilder](https://github.com/kubernetes-sigs/kubebuilder) 是一个使用 CRDs 构建 K8s API 的 SDK，主要是：

- 提供脚手架工具初始化 CRDs 工程，自动生成 boilerplate 代码和配置；
- 提供代码库封装底层的 K8s go-client；


方便用户从零开始开发 CRDs，Controllers 和 Admission Webhooks 来扩展 K8s。

本文主要基于 **kubebuilder v3.11.1**，从零开发一个 operator 应用。

## 准备工作

- [go](https://golang.org/dl/) version v1.20.0+
- [docker](https://docs.docker.com/install/) version 17.03+.
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) version v1.11.3+.
- 能够访问 Kubernetes v1.11.3+ 集群.

**安装kubebuilder**

```
os=$(go env GOOS)
arch=$(go env GOARCH)

curl -L -o kubebuilder https://go.kubebuilder.io/dl/3.11.1/${os}/${arch}
chmod +x kubebuilder && mv kubebuilder /usr/local/bin/
```

## 项目示例

### 初始化项目

```shell
mkdir -p /go/src/kubebuilder-project
cd /go/src/kubebuilder-project
kubebuilder init --domain chinaunicom.cn --repo=kubebuilder-project
```

>--domain: 设定域名，一般设定为公司域名
>
>--repo: 如果您的项目在GOPATH中初始化，则隐式调用go mod init将为您插入模块路径。否则必须设置--repo=<模块路径>。

### 创建 API 和 Controller

```shell
kubebuilder create api --group extensions --version v1 --kind UnitedDeployment --namespaced true
// 实际上不仅会创建 API，也就是 CRD，还会生成 Controller 的框架

Create Resource [y/n]
y
Create Controller [y/n]
y
```

**参数解读**：

* group:  表示该CRD 的 Group组前缀，group=<group>.<domain>, 则该CRD生成的 group=extensions.chinaunicom.cn

- version 一般分三种，按社区标准：
    - v1alpha1: 此 api 不稳定，CRD 可能废弃、字段可能随时调整，不要依赖；
    - v1beta1: api 已稳定，会保证向后兼容，特性可能会调整；
    - v1: api 和特性都已稳定；
- kind: 此 CRD 的类型，使用大驼峰单数形式；
- namespaced: 此 CRD 是k8s集群级别资源，还是 namespace隔离的

### 创建 webhook（可选）

```shell
// 创建UnitedSet的 MutatingAdmissionWebhook 和 ValidatingAdmissionWebhook
kubebuilder create webhook --group extensions --version v1 --kind UnitedDeployment --defaulting --programmatic-validation
```

**参数解读**：

* defaulting: 创建 MutatingAdmissionWebhook
* programmatic-validation: 创建 ValidatingAdmissionWebhook

至此，kubebuilder 脚手架搭建的 operator 框架已经完成，下面看一下项目结构：

```text
[root@localhost kubebuilder-project]# tree
.
|-- Dockerfile
|-- Makefile
|-- PROJECT
|-- README.md
|-- api
|   `-- v1
|       |-- groupversion_info.go
|       |-- uniteddeployment_types.go
|       |-- uniteddeployment_webhook.go
|       |-- webhook_suite_test.go
|       `-- zz_generated.deepcopy.go
|-- bin
|   `-- controller-gen
|-- cmd
|   `-- main.go
|-- config
|   |-- certmanager
|   |   |-- certificate.yaml
|   |   |-- kustomization.yaml
|   |   `-- kustomizeconfig.yaml
|   |-- crd
|   |   |-- kustomization.yaml
|   |   |-- kustomizeconfig.yaml
|   |   `-- patches
|   |       |-- cainjection_in_uniteddeployments.yaml
|   |       `-- webhook_in_uniteddeployments.yaml
|   |-- default
|   |   |-- kustomization.yaml
|   |   |-- manager_auth_proxy_patch.yaml
|   |   |-- manager_config_patch.yaml
|   |   |-- manager_webhook_patch.yaml
|   |   `-- webhookcainjection_patch.yaml
|   |-- manager
|   |   |-- kustomization.yaml
|   |   `-- manager.yaml
|   |-- prometheus
|   |   |-- kustomization.yaml
|   |   `-- monitor.yaml
|   |-- rbac
|   |   |-- auth_proxy_client_clusterrole.yaml
|   |   |-- auth_proxy_role.yaml
|   |   |-- auth_proxy_role_binding.yaml
|   |   |-- auth_proxy_service.yaml
|   |   |-- kustomization.yaml
|   |   |-- leader_election_role.yaml
|   |   |-- leader_election_role_binding.yaml
|   |   |-- role_binding.yaml
|   |   |-- service_account.yaml
|   |   |-- uniteddeployment_editor_role.yaml
|   |   `-- uniteddeployment_viewer_role.yaml
|   |-- samples
|   |   |-- extensions_v1_uniteddeployment.yaml
|   |   `-- kustomization.yaml
|   `-- webhook
|       |-- kustomization.yaml
|       |-- kustomizeconfig.yaml
|       `-- service.yaml
|-- go.mod
|-- go.sum
|-- hack
|   `-- boilerplate.go.txt
`-- internal
    `-- controller
        |-- suite_test.go
        `-- uniteddeployment_controller.go
```

### 定义 CRD 字段

自定义 UnitedDeploymentSpec 字段

```golang
type UnitedDeploymentSpec struct {
	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
	// Important: Run "make" to regenerate code after modifying this file

	// Foo is an example field of UnitedDeployment. Edit uniteddeployment_types.go to remove/update
	DeploymentName string `json:"deploymentName"`
	Replicas       *int32 `json:"replicas"`
}
```

自定义 UnitedDeploymentStatus 字段

```golang
type UnitedDeploymentStatus struct {
	// INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
	// Important: Run "make" to regenerate code after modifying this file
	AvailableReplicas *int32 `json:"availableReplicas"`
}
```

>**注意：**
>
>如果修改了 `apis/v1/uniteddeployment_types.go` ，需要执行以下命令来更新代码和manifests：

```
make && make manifests
```

### 使用 code-generator

#### 更新依赖版本

初始化项目后的go.mod：

```
module kubebuilder-project

go 1.21

require (
        github.com/onsi/ginkgo/v2 v2.9.5
        github.com/onsi/gomega v1.27.7
        k8s.io/api v0.27.2
        k8s.io/apimachinery v0.27.2
        k8s.io/client-go v0.27.2
        sigs.k8s.io/controller-runtime v0.15.0
)
```

需要将初始化的k8s库更新到要使用的版本，如：

```bash
K8S_VERSION=v0.28.0
go get k8s.io/api@$K8S_VERSION
go get k8s.io/client-go@$K8S_VERSION
go get k8s.io/apimachinery@$K8S_VERSION
```

#### 安装 code-generator

```bash
K8S_VERSION=v0.28.0
go get k8s.io/code-generator@$K8S_VERSION
go mod vendor
```

> **注意：**
>
> * code-generator 包版本, 需要与 kubernetes 相关包版本，保持一致
> * 需要将依赖复制到 vendor 中

#### 创建 && 修改所需文件

需要在api目录下创建code-generator所需的文件，并添加相关注释。

- 新增 `api/v1/doc.go`

  **注意：**修改groupName，package与api的version保持一致。

  ```go
  // +groupName=extensions.chinaunicom.cn
  package v1
  ```

- 新增 `api/v1/register.go`

  **注意：**package与api的version保持一致。

  ```go
  package v1
  
  import (
      "k8s.io/apimachinery/pkg/runtime/schema"
  )
  
  // SchemeGroupVersion is group version used to register these objects.
  var SchemeGroupVersion = GroupVersion
  
  // Resource takes an unqualified resource and returns a Group qualified GroupResource
  func Resource(resource string) schema.GroupResource {
      return SchemeGroupVersion.WithResource(resource).GroupResource()
  }
  ```

- 修改 `api/v1/{crd}_types.go` 文件，添加注释 `// +genclient`

  ```golang
  //+genclient
  //+kubebuilder:object:root=true
  //+kubebuilder:subresource:status
  
  // UnitedDeployment is the Schema for the uniteddeployments API
  type UnitedDeployment struct {
  	metav1.TypeMeta   `json:",inline"`
  	metav1.ObjectMeta `json:"metadata,omitempty"`
  
  	Spec   UnitedDeploymentSpec   `json:"spec,omitempty"`
  	Status UnitedDeploymentStatus `json:"status,omitempty"`
  }
  ```

#### 准备 hack 脚本

在项目 `hack` 目录下准备以下文件：

* 新建 `hack/tools.go` 文件导入 code-generator 包

```golang
//go:build tools
// +build tools

// This package imports things required by build scripts, to force `go mod` to see them as dependencies
package tools

import _ "k8s.io/code-generator"
```

* 新建 `hack/update-codegen.sh`，根据项目修改相应的变量

```shell
#!/usr/bin/env bash

set -o errexit
set -o nounset
set -o pipefail

SCRIPT_ROOT=$(dirname "${BASH_SOURCE[0]}")/..
CODEGEN_PKG=${CODEGEN_PKG:-$(cd "${SCRIPT_ROOT}"; ls -d -1 ./vendor/k8s.io/code-generator 2>/dev/null || echo ../code-generator)}

source "${CODEGEN_PKG}/kube_codegen.sh"

# generate the code with:
# --output-base    because this script should also be able to run inside the vendor dir of
#                  k8s.io/kubernetes. The output-base is needed for the generators to output into the vendor dir
#                  instead of the $GOPATH directly. For normal projects this can be dropped.

kube::codegen::gen_helpers \
    --input-pkg-root kubebuilder-project/apis \
    --output-base "$(dirname "${BASH_SOURCE[0]}")/../.." \
    --boilerplate "${SCRIPT_ROOT}/hack/custom-boilerplate.go.txt"

kube::codegen::gen_client \
    --with-watch \
    --input-pkg-root kubebuilder-project/apis \
    --output-pkg-root kubebuilder-project/pkg/generated \
    --output-base "$(dirname "${BASH_SOURCE[0]}")/../.." \
    --boilerplate "${SCRIPT_ROOT}/hack/custom-boilerplate.go.txt"
```

hack 目录下，新建 verify-codegen.sh

```shell
#!/usr/bin/env bash

set -o errexit
set -o nounset
set -o pipefail

SCRIPT_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd -P)"
DIFFROOT="${SCRIPT_ROOT}/pkg"
TMP_DIFFROOT="$(mktemp -d -t "$(basename "$0").XXXXXX")/pkg"

cleanup() {
  rm -rf "${TMP_DIFFROOT}"
}
trap "cleanup" EXIT SIGINT

cleanup

mkdir -p "${TMP_DIFFROOT}"
cp -a "${DIFFROOT}"/* "${TMP_DIFFROOT}"

"${SCRIPT_ROOT}/hack/update-codegen.sh"
echo "diffing ${DIFFROOT} against freshly generated codegen"
ret=0
diff -Naupr "${DIFFROOT}" "${TMP_DIFFROOT}" || ret=$?
if [[ $ret -eq 0 ]]; then
  echo "${DIFFROOT} up to date."
else
  echo "${DIFFROOT} is out of date. Please run hack/update-codegen.sh"
  exit 1
fi
```

hack目录脚本文件，可以参考，[sample-controller/hack](https://github.com/kubernetes/sample-controller/tree/v0.28.0/hack) 项目

#### 生成代码

**注意**：kubebuilder v3.11.1 版本生成的 api 目录结构, code-generator无法直接使用(将 api 由 `api/${VERSION}` 移动至 `apis/${GROUP}/${VERSION}` 即可)

修改 `Makefile` ，添加生成命令

```makefile
.PHONY: update-codegen
update-codegen:
	chmod +x ./hack/update-codegen.sh
	./hack/update-codegen.sh
```

项目根目录下执行`make update-codegen `即可，将生成如下代码结构：

```text
[root@localhost kubebuilder-project]# tree -L 4
.
|-- apis
|   `-- extensions
|       `-- v1
|           |-- doc.go
|           |-- groupversion_info.go
|           |-- register.go
|           |-- types.go
|           |-- uniteddeployment_webhook.go
|           |-- webhook_suite_test.go
|           `-- zz_generated.deepcopy.go
`-- pkg
    `-- generated
        |-- clientset
        |   `-- versioned
        |-- informers
        |   `-- externalversions
        `-- listers
            `-- extensions
```

这里有一个坑，需要把 `apis/extensions/v1`目录下，`unitddeployment_type.go  `修改为 `type.go`,否则执行命令后，无法生成代码。

### 本地调试运行

#### CRD安装

第一种方式：

在本地机器执行，make install，在 `config/crd/bases` 目录生成 crd.yaml，拷贝至 k8s 集群机器，执行

```shell
$ kubectl create -f extensions.chinaunicom.cn_uniteddeployments.yaml
```

第二种方式:
将代码拷贝至k8s集群机器

```shell
$ make install    // 先生成 config/crd/bases 文件，里面包含 crd 的YAML文件，然后kubectl apply
```

然后我们就可以看到创建的 CRD 了

```shell
$ kubectl get crd |grep united
uniteddeployments.extensions.chinaunicom.cn           2023-09-05T02:15:40Z
```

创建一个 uniteddeployment 资源

```shell
$ cd kubebuilder-project/config/samples

$ kubectl apply -f extensions_v1_uniteddeployment.yaml

$ kubectl get uniteddeployments.extensions.chinaunicom.cn
NAME                      AGE
uniteddeployment-sample   103s
```

# 总结

至此，通过使用 kubebuilder 与 code-generator 一个完整的 operator 开发框架就搭建好了，剩下的工作就主要是写 controller 与webhook 的业务逻辑。

# 参考资料

* kuberbuilder: https://github.com/kubernetes-sigs/kubebuilder/tree/v3.11.1

* code-generator: https://github.com/kubernetes/code-generator/tree/v0.28.0

* sample-controller: https://github.com/kubernetes/sample-controller/tree/v0.28.0

  