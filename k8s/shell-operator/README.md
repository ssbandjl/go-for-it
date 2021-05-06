<p align="center">
<img width="485" src="docs/shell-operator-logo.png" alt="Shell-operator logo" />
</p>

<p align="center">
<a href="https://hub.docker.com/r/flant/shell-operator"><img src="https://img.shields.io/docker/pulls/flant/shell-operator.svg?logo=docker" alt="docker pull flant/shell-operator"/></a>
 <a href="https://github.com/flant/shell-operator/discussions"><img src="https://img.shields.io/badge/GitHub-discussions-brightgreen" alt="GH Discussions"/></a>
<a href="https://t.me/kubeoperator"><img src="https://img.shields.io/badge/telegram-RU%20chat-179cde.svg?logo=telegram" alt="Telegram chat RU"/></a>
</p>
**Shell-operator** is a tool for running event-driven scripts in a Kubernetes cluster. 在k8s集群中, 基于事件驱动脚本执行的工具

This operator is not an operator for a _particular software product_ such as `prometheus-operator` or `kafka-operator`. Shell-operator provides an integration layer between Kubernetes cluster events and shell scripts by treating scripts as hooks triggered by events. Think of it as an `operator-sdk` but for scripts.

Shell-operator is used as a base for more advanced [addon-operator](https://github.com/flant/addon-operator) that supports Helm charts and value storages. 作为高级插件Operator的基础

Shell-operator provides: 功能

- __Ease of management of a Kubernetes cluster__: use the tools that Ops are familiar with. It can be bash, python, kubectl, etc. 简易管理集群: 类似bash/python/kubectl等运维操作
- __Kubernetes object events__: hook can be triggered by `add`, `update` or `delete` events. **[Learn more](HOOKS.md) about hooks.**   k8s对象事件: 由k8s资源的添加/更新/删除等事件触发钩子(脚本)执行
- __Object selector and properties filter__: shell-operator can monitor a particular set of objects and detect changes in their properties. 选择对象和属性过滤: shell-operator能够监控资源的特殊字段
- __Simple configuration__: hook binding definition is a JSON or YAML document on script's stdout. 配置简单: 脚本的标准输出(Json/YAML)就可以绑定到钩子的定义
- __Validating webhook machinery__: hook can handle validating for Kubernetes resources. 自动校验钩子: 钩子可以控制资源的校验
- __Conversion webhook machinery__: hook can handle version conversion for Kubernetes resources.  自动转换钩子: 钩子可以控制资源的版本切换

**Contents**:
* [Quickstart](#quickstart)
  * [Build an image with your hooks](#build-an-image-with-your-hooks)
  * [Create RBAC objects](#create-rbac-objects)
  * [Install shell-operator in a cluster](#install-shell-operator-in-a-cluster)
  * [It all comes together](#it-all-comes-together)
* [Hook binding types](#hook-binding-types)
  * [`kubernetes`](#kubernetes)
  * [`onStartup`](#onstartup)
  * [`schedule`](#schedule)
* [Prometheus target](#prometheus-target)
* [Examples and notable users](#examples-and-notable-users)
* [Articles & talks](#articles--talks)
* [Community](#community)
* [License](#license)

# Quickstart

> You need to have a Kubernetes cluster, and the `kubectl` must be configured to communicate with your cluster.

The simplest setup of shell-operator in your cluster consists of these steps: 最简单的shell-operator安装步骤

- build an image with your hooks (scripts) 将钩子(脚本集)构建为一个镜像
- create necessary RBAC objects (for `kubernetes` bindings) 创建RBAC对象
- run Pod or Deployment with the built image  用构建的镜像运行一个Pod或部署即可

For more configuration options see [RUNNING](RUNNING.md). 配置选项

## Build an image with your hooks

A hook is a script that, when executed with `--config` option, outputs configuration to stdout in YAML or JSON format. [Learn more](HOOKS.md) about hooks. 一个钩子就是一个脚本, 当脚本以--config选项执行时, 标准输出的JSON或YAML就是配置钩子的配置

Let's create a small operator that will watch for all Pods in all Namespaces and simply log the name of a new Pod.

示例: 让我们创建一个简易的oprator, 它会watch所有的Pod事件, 当新Pod创建时, 通过日志打印该信息

`kubernetes` binding is used to tell shell-operator about objects that we want to watch. Create the `pods-hook.sh` file with the following content:

```bash
#!/usr/bin/env bash

if [[ $1 == "--config" ]] ; then
  cat <<EOF
configVersion: v1
kubernetes:
- apiVersion: v1
  kind: Pod
  executeHookOnEvent: ["Added"]
EOF
else
  podName=$(jq -r .[0].object.metadata.name $BINDING_CONTEXT_PATH)
  echo "Pod '${podName}' added"
fi
```

Make the `pods-hook.sh` executable:

```shell
chmod +x pods-hook.sh
```

You can use a prebuilt image [flant/shell-operator:latest](https://hub.docker.com/r/flant/shell-operator) with `bash`, `kubectl`, `jq` and `shell-operator` binaries to build you own image. You just need to `ADD` your hook into `/hooks` directory in the `Dockerfile`.

Create the following `Dockerfile` in the directory where you created the `pods-hook.sh` file:

```dockerfile
FROM flant/shell-operator:latest
ADD pods-hook.sh /hooks
```

Build an image (change image tag according to your Docker registry):

```shell
docker build -t "registry.mycompany.com/shell-operator:monitor-pods" .
```

Push image to the Docker registry accessible by the Kubernetes cluster:

```shell
docker push registry.mycompany.com/shell-operator:monitor-pods
```

## Create RBAC objects

We need to watch for Pods in all Namespaces. That means that we need specific RBAC definitions for shell-operator:

```shell
kubectl create namespace example-monitor-pods  #创建命名空间example-monitor-pods
kubectl create serviceaccount monitor-pods-acc --namespace example-monitor-pods  #创建服务账户
kubectl create clusterrole monitor-pods --verb=get,watch,list --resource=pods  #创建集群角色monitor-pods
kubectl create clusterrolebinding monitor-pods --clusterrole=monitor-pods --serviceaccount=example-monitor-pods:monitor-pods-acc #创建集群角色绑定monitor-pods
```

## Install shell-operator in a cluster

Shell-operator can be deployed as a Pod. Put this manifest into the `shell-operator-pod.yaml` file:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shell-operator
spec:
  containers:
  - name: shell-operator
    image: registry.mycompany.com/shell-operator:monitor-pods
    imagePullPolicy: Always
  serviceAccountName: monitor-pods-acc
```

Start shell-operator by applying a `shell-operator-pod.yaml` file:

```shell
kubectl -n example-monitor-pods apply -f shell-operator-pod.yaml
```

## It all comes together 聚集在一起

Let's deploy a [kubernetes-dashboard](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/) to trigger  `kubernetes` binding defined in our hook:  让我们部署一个Pod来触发钩子执行

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/aio/deploy/recommended.yaml
```

Now run `kubectl -n example-monitor-pods logs po/shell-operator` and observe that the hook will print dashboard pod name:

```plain
...
INFO[0027] queue task HookRun:main                       operator.component=handleEvents queue=main
INFO[0030] Execute hook                                  binding=kubernetes hook=pods-hook.sh operator.component=taskRunner queue=main task=HookRun
INFO[0030] Pod 'kubernetes-dashboard-775dd7f59c-hr7kj' added  binding=kubernetes hook=pods-hook.sh output=stdout queue=main task=HookRun
INFO[0030] Hook executed successfully                    binding=kubernetes hook=pods-hook.sh operator.component=taskRunner queue=main task=HookRun
...
```

> *Note:* hook output is logged with output=stdout label.

To clean up a cluster, delete namespace and RBAC objects:

```shell
kubectl delete ns example-monitor-pods
kubectl delete clusterrole monitor-pods
kubectl delete clusterrolebinding monitor-pods
```

This example is also available in /examples: [monitor-pods](examples/101-monitor-pods).

# Hook binding types 钩子绑定类型

Every hook should respond with JSON or YAML configuration of bindings when executed with `--config` flag.

## kubernetes

This binding defines a subset of Kubernetes objects that shell-operator will monitor and a [jq](https://github.com/stedolan/jq/) expression to filter their properties. Read more about `onKubernetesEvent` bindings [here](HOOKS.md#kubernetes).

Example of YAML output from `hook --config`:

```yaml
configVersion: v1
kubernetes:
- name: execute_on_changes_of_namespace_labels
  kind: Namespace
  executeHookOnEvent: ["Modified"]
  jqFilter: ".metadata.labels"
```

> Note: it is possible to watch Custom Defined Resources, just use proper values for `apiVersion` and `kind` fields.

> Note: return configuration as JSON is also possible as JSON is a subset of YAML.

## onStartup 启动时

This binding has only one parameter: order of execution. Hooks are loaded at the start and then hooks with onStartup binding are executed in the order defined by parameter. Read more about `onStartup` bindings [here](HOOKS.md#onstartup).

Example `hook --config`:

```yaml
configVersion: v1
onStartup: 10
```

## schedule 调度(周期性执行)

This binding is used to execute hooks periodically. A schedule can be defined with a granularity of seconds. Read more about `schedule` bindings [here](HOOKS.md#schedule).

Example `hook --config` with 2 schedules:

```yaml
configVersion: v1
schedule:
- name: "every 10 min"
  crontab: "*/10 * * * *"
  allowFailure: true
- name: "Every Monday at 8:05"
  crontab: "5 8 * * 1"
  queue: mondays
```

# Prometheus target 普罗米修斯目标(暴露指标)

Shell-operator provides a `/metrics` endpoint. More on this in [METRICS](METRICS.md) document.

# Examples and notable users 案例

More examples of how you can use shell-operator are available in the [examples](examples/) directory.

Prominent shell-operator use cases include: 典型案例

* [Deckhouse](https://deckhouse.io/) Kubernetes platform where both projects, shell-operator and addon-operator, are used as the core technology to configure & extend K8s features;
* KubeSphere Kubernetes platform's [installer](https://github.com/kubesphere/ks-installer); 青云KubeSphere安装
* [Kafka DevOps solution](https://github.com/confluentinc/streaming-ops) from Confluent.

Please find out & share more examples in [Show & tell discussions](https://github.com/flant/shell-operator/discussions/categories/show-and-tell).

# Articles & talks 博客

Shell-operator has been presented during KubeCon + CloudNativeCon Europe 2020 Virtual (Aug'20). Here is the talk called "Go? Bash! Meet the shell-operator": 

* [YouTube video](https://www.youtube.com/watch?v=we0s4ETUBLc);
* [text summary](https://medium.com/flant-com/meet-the-shell-operator-kubecon-36c14ba2f8fe);
* [slides](https://speakerdeck.com/flant/go-bash-meet-the-shell-operator).

Official publications on shell-operator: 官方文档
* "[shell-operator v1.0.0: the long-awaited release of our tool to create Kubernetes operators](https://blog.flant.com/shell-operator-v1-release-for-kubernetes-operators/)" (Apr'21);
* "[shell-operator & addon-operator news: hooks as admission webhooks, Helm 3, OpenAPI, Go hooks, and more!](https://blog.flant.com/shell-operator-addon-operator-v1-rc1-changes/)" (Feb'21);
* "[Kubernetes operators made easy with shell-operator: project status & news](https://blog.flant.com/kubernetes-operators-made-easy-with-shell-operator-project-status-news/)" (Jul'20);
* "[Announcing shell-operator to simplify creating of Kubernetes operators](https://blog.flant.com/announcing-shell-operator-to-simplify-creating-of-kubernetes-operators/)" (May'19).

Other languages: 其他语言的博客
* Chinese: "[介绍一个不太小的工具：Shell Operator](https://blog.fleeto.us/post/shell-operator/)"; "[使用shell-operator实现Operator](https://cloud.tencent.com/developer/article/1701733)";
* Dutch: "[Een operator om te automatiseren – Hoe pak je dat aan?](https://www.hcs-company.com/blog/operator-automatiseren-namespace-openshift)";
* Russian: "[shell-operator v1.0.0: долгожданный релиз нашего проекта для Kubernetes-операторов](https://habr.com/ru/company/flant/blog/551456/)"; "[Представляем shell-operator: создавать операторы для Kubernetes стало ещё проще](https://habr.com/ru/company/flant/blog/447442/)".

# Community 社区

Please feel free to reach developers/maintainers and users via [GitHub Discussions](https://github.com/flant/shell-operator/discussions) for any questions regarding shell-operator.

You're also welcome to follow [@flant_com](https://twitter.com/flant_com) to stay informed about all our Open Source initiatives.

# License 许可证

Apache License 2.0, see [LICENSE](LICENSE).