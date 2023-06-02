---
title: 管理基于角色的访问控制
date: 2023-06-02 11:27:29
tags: ['IT', 'Kubernetes', 'K8S','CKA认证']
description: '关于CKA认证'
keywords: ['Kubernetes', 'CKA']
category: ['IT', 'Container', 'CKA认证']
---
[参考](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/rbac/)

# 启用RBAC鉴权
``` SHELL
kube-apiserver --authorization-mode=RBAC
```

# RBAC API 声明的四种 Kubernetes 对象
## 角色
`角色`中包含一组代表相关权限的规则。
- Role
- ClusterRole

## 角色绑定
`角色绑定`是将角色中定义的权限赋予一个或者一组用户
- RoleBinding
- ClusterRoleBinding




# Role
Role 是一个`命名空间作用域`的资源，总是用来在某个`命名空间作用域`内设置访问权限。

**示例:** 在 "default" 命名空间创建Role，可用来授予对 Pod 的读访问权限
``` YAML
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" 标明 core API 组
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

# ClusterRole
`ClusterRole` 则是一个`集群作用域`的资源。

## 用法:
1. 为`命名空间作用域`的资源定义权限，并在`某个(些)独立命名空间`内被授予访问权限。
2. 为`命名空间作用域`的资源定义权限，并在`跨全部命名空间`内被授予访问权限。
3. 为`集群作用域`的资源定义权限。

除了被用来像Role一样授予`命名空间作用域`资源的权限以外，也可以被用来授予以下对象的访问权限：
- 集群范围资源（比如节点（`Node`））
- 非资源端点（比如 `/healthz`）
- 跨命名空间访问的命名空间作用域的资源（如 `Pod`）。比如，你可以使用 `ClusterRole` 来允许某特定用户执行`kubectl get pods --all-namespaces`

**示例:** 为任一命名空间中，或者跨命名空间，的`Secret`授予读访问权限（取决于该角色是如何绑定的）：
``` YAML
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  # "namespace" 被忽略，因为 ClusterRoles 不受命名空间限制
  name: secret-reader
rules:
- apiGroups: [""]
  # 在 HTTP 层面，用来访问 Secret 资源的名称为 "secrets"
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
```

> Kubernetes 对象要么是`命名空间作用域`的，要么是`集群作用域`的，不可两者兼具。
> 如若想在某个`命名空间作用域`内定义一个角色，使用Role;
> 如果想在`集群作用域`定义一个角色，使用ClusterRole。


# RoleBinding

## 用法:
1. 一个 `RoleBinding` 可以引用同一的命名空间中的任何 `Role。` 
2. 一个 `RoleBinding` 可以引用某 `ClusterRole` 并将该 `ClusterRole` 绑定到 `RoleBinding` 所在的命名空间。

**示例:** `RoleBinding`引用一个 `Role`, 授予用户 "jane" 读取 "default" 命名空间内所有`Pod`的权限。
``` YAML
apiVersion: rbac.authorization.k8s.io/v1
# 此角色绑定允许 "jane" 读取 "default" 名字空间中的 Pod
# 你需要在该命名空间中有一个名为 “pod-reader” 的 Role
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
# 你可以指定不止一个“subject（主体）”
- kind: User
  name: jane # "name" 是区分大小写的
  apiGroup: rbac.authorization.k8s.io
roleRef:
  # "roleRef" 指定与某 Role 或 ClusterRole 的绑定关系
  kind: Role        # 此字段必须是 Role 或 ClusterRole
  name: pod-reader  # 此字段必须与你要绑定的 Role 或 ClusterRole 的名称匹配
  apiGroup: rbac.authorization.k8s.io
```

**示例:** `RoleBinding`引用一个 `ClusterRole`, 授予"dave"访问"development"命名空间内的`Secrets`对象。
``` YAML
apiVersion: rbac.authorization.k8s.io/v1
# 此角色绑定使得用户 "dave" 能够读取 "development" 名字空间中的 Secrets
# 你需要一个名为 "secret-reader" 的 ClusterRole
kind: RoleBinding
metadata:
  name: read-secrets
  # RoleBinding 的名字空间决定了访问权限的授予范围。
  # 这里隐含授权仅在 "development" 名字空间内的访问权限。
  namespace: development
subjects:
- kind: User
  name: dave # 'name' 是区分大小写的
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

# ClusterRoleBinding

## 用法:
1. 使用 `ClusterRoleBinding` 将某 `ClusterRole` 绑定到`集群中所有命名空间`。

**示例:** 允许 "manager" 组内的所有用户访问任何名字空间中的 `Secret`。
``` YAML
apiVersion: rbac.authorization.k8s.io/v1
# 此集群角色绑定允许 “manager” 组中的任何人访问任何名字空间中的 Secret 资源
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: manager      # 'name' 是区分大小写的
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

> 创建了绑定之后，绑定对象所引用的 Role 或 ClusterRole将不能再修改。
> 试图改变绑定对象的 roleRef 将导致合法性检查错误。 
> 如果你想要改变现有绑定对象中 roleRef 字段的内容，必须删除重新创建绑定对象。

命令 `kubectl auth reconcile` 可以创建或者更新包含 RBAC 对象的清单文件， 并且在必要的情况下删除和重新创建绑定对象，以改变所引用的角色。