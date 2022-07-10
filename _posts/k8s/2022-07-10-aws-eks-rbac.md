---
title: Role Based Access Control in EKS (K8s) and AWS IAM
categories: [k8s,aws]
tags: [k8s,rbac,aws,eks,iam]
---

## Role Based Access Control (RBAC) in k8s
---

In this post you will learn about enabling access to your EKS cluster using IAM roles. You can read more about kubernetes **R**ole **B**ased **A**ccess **C**ontrol system from the k8s official documentation [page](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

K8s uses an API group called `rbac.authorization.k8s.io`, which is used to drive authorization decisions.


Refer to this aws official [document](https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html) to read more about enabling user and role access to your cluster.

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

## Scenario
---

Let's try to implement the below scenario to understand the concepts better.

> Let's say you run a production EKS cluster in AWS, where you want to setup different levels of access controls to different groups of people in your project.
>
>**admins:** should have full `admin` access to your cluster.
>
>**developers:** should have full `read` access and `write` access to some api-groups.
>
>**interns:** should have only `read` access to some api-groups. 

### **Create roles in AWS IAM**

- `eks-admins`
- `developers`
- `interns`

Copy the below JSON block, replace the `<ACCOUNT_NUMBER>` with your AWS account number and save it as `trust_policy.json`

```json
{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": "sts:AssumeRole",
        "Principal": {
          "AWS": "<ACCOUNT_NUMBER>"
        },
        "Condition": {}
      }
    ]
  }
```

Run the below commands to create the necessary IAM roles.

```shell
aws iam create-role --role-name eks-admins --assume-role-policy-document file://trust_policy.json
```

```shell
aws iam create-role --role-name developers --assume-role-policy-document file://trust_policy.json
```

```shell
aws iam create-role --role-name interns --assume-role-policy-document file://trust_policy.json
```

On executing the above commands you will get a JSON response back which contains an `ARN`. Copy those and save it separately as we need those role ARN's in the later part.

**Example:**

- `arn:aws:iam::<ACCOUNT_NUMBER>:role/eks-admins`
- `arn:aws:iam::<ACCOUNT_NUMBER>:role/developers`
- `arn:aws:iam::<ACCOUNT_NUMBER>:role/interns`

### **Create k8s Roles and Rolebindings**

As per the official docs

>When you create an Amazon EKS cluster, the AWS Identity and Access Management (IAM) entity user or role, such as a federated user that creates the cluster, is automatically granted `system:masters` permissions in the cluster's role-based access control (RBAC) configuration in the Amazon EKS control plane.

Let's create a `ClusterRoleBinding` called `eks-admins-RoleBinding` and reference/tag it to one of the default cluster roles called `system:masters` which grants **un-restricted** access to the cluster.


```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: eks-admins-RoleBinding
subjects:
- kind: Group
  name: eks-admins
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:masters
  apiGroup: rbac.authorization.k8s.io
```

## Additional references:

- [https://blog.aquasec.com/kubernetes-authorization](https://blog.aquasec.com/kubernetes-authorization)