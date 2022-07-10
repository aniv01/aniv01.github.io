---
title: Role Based Access Control in EKS (K8s) and AWS IAM
categories: [k8s,aws]
tags: [k8s,rbac,aws,eks,iam]
---

## Role Based Access Control (RBAC) in k8s
---

In this post you will learn about enabling access to your EKS cluster using IAM roles. You can read more about kubernetes **R**ole **B**ased **A**ccess **C**ontrol system from the k8s official documentation [page](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

K8s uses an API group called `rbac.authorization.k8s.io`, which is used to drive authorization decisions.

## **RBAC** API objects
---

Below are the four API objects that are used when it comes to authorization in kubernetes.

- Role
- RoleBinding
- ClusterRole
- ClusterRoleBinding

## What is Role and ClusterRole?
---

A `Role` and the `ClusterRole` contains set of rules that grants access to different api groups.

Role sets permissions at `namespace` level. ClusterRole sets permissions at `cluster` level.

## What is RoleBinding and ClusterRoleBinding?
---

A `RoleBinding` and the `ClusterRoleBinding` grants the permissions defined in a role to a user or set of users.

It holds list of below subjects:

- User
- Group
- ServiceAccount

Let's try to implement the below scenario to understand the concepts better.

> Let's say you run a production EKS cluster in AWS, where you want to setup different levels of access control to different groups of people in your project.
>
>**admins:** should have full `admin` access to your cluster.
>
>**developers:** should have full `read` access and `write` access to some api-groups.
>
>**interns:** should have only `read` access to some api-groups. 
