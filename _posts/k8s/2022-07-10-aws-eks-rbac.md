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

## What is `aws-auth` configmap?

## Scenario
---

Let's try to implement the below scenario to understand the concepts better.

> Let's say you run a production EKS cluster in AWS, where you want to setup different levels of access controls to different groups of people in your project.
>
>**eks-admins:** should have full `admin` access to your cluster.
>
>**developers:** should have full `read` access and `write` access to some api-groups.
>
>**interns:** should have only `read` access to some api-groups. 

### **Create roles in AWS IAM**
---

Let's create the below roles in IAM.

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

On executing the above commands you will get an JSON response back which contains an `ARN`. Copy those and save it separately as we need those role ARN's in the later part.

**Example:**

- `arn:aws:iam::<ACCOUNT_NUMBER>:role/eks-admins`
- `arn:aws:iam::<ACCOUNT_NUMBER>:role/developers`
- `arn:aws:iam::<ACCOUNT_NUMBER>:role/interns`

### **Create k8s Roles and Rolebindings**

As per the official docs

>When you create an Amazon EKS cluster, the AWS Identity and Access Management (IAM) entity user or role, such as a federated user that creates the cluster, is automatically granted `system:masters` permissions in the cluster's role-based access control (RBAC) configuration in the Amazon EKS control plane.

Continue with the below steps with the same IAM principal using which the EKS cluster was created.

- For full admin access we need to map our IAM role `arn:aws:iam::<ACCOUNT_NUMBER>:role/eks-admins` to `cluster-admin` [ClusterRole](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#user-facing-roles)

Edit the `aws-auth` config map from `kube-system` namespace.

```shell
kubectl edit configmap aws-auth -n kube-system
```

Map the role `arn:aws:iam::<ACCOUNT_NUMBER>:role/eks-admins` to `cluster-admin` group.

```yaml
apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::<ACCOUNT_NUMBER>:role/eksctl-demo-nodegroup-default-NodeInstanceRole-1TSDWTWUWYBS6
      username: system:node:{{EC2PrivateDNSName}}
    - groups:
      - system:masters
      rolearn: arn:aws:iam::<ACCOUNT_NUMBER>:role/eks-admins
      username: eks-admins
  mapUsers: |
    []
kind: ConfigMap
metadata:
  creationTimestamp: "2022-07-11T03:19:02Z"
  name: aws-auth
  namespace: kube-system
  resourceVersion: "2203"
  uid: 9636c623-5c4a-4fbc-b5f7-94f2d842af90
```

- For `developers` we need to create a `ClusterRole` and `ClusterRoleBinding` and map it to `arn:aws:iam::<ACCOUNT_NUMBER>:role/developers` IAM role.

Copy the below YAML block and save it as `developers_rbac.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: developers_role
rules:
- apiGroups: ["*"]
  resources: ["pods", "services"]
  verbs: ["list", "get", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: developers_role_binding
subjects:
  - kind: Group
    name: developers
roleRef:
  kind: ClusterRole
  name: developers_role
  apiGroup: rbac.authorization.k8s.io
```

```shell
kubectl apply -f developers_rbac.yaml
```

Added the role mapping under `aws-auth` config map

```shell
kubectl edit configmap aws-auth -n kube-system
```

```yaml
apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::<ACCOUNT_NUMBER>:role/eksctl-demo-nodegroup-default-NodeInstanceRole-1TSDWTWUWYBS6
      username: system:node:{{EC2PrivateDNSName}}
    - groups:
      - system:masters
      rolearn: arn:aws:iam::<ACCOUNT_NUMBER>:role/eks-admins
      username: eks-admins
    - groups:
      - developers
      rolearn: arn:aws:iam::<ACCOUNT_NUMBER>:role/developers
      username: developers
  mapUsers: |
    []
kind: ConfigMap
metadata:
  creationTimestamp: "2022-07-11T03:19:02Z"
  name: aws-auth
  namespace: kube-system
  resourceVersion: "2203"
  uid: 9636c623-5c4a-4fbc-b5f7-94f2d842af90
```

## Configuring AWS CLI

```shell
aws configure --profile developers
```

```shell
cat ~/.aws/config
[default]
region = us-east-1
output = json

[profile eks-devs]
role_arn = arn:aws:iam::<ACCOUNT_NUMBER>:role/developers
source_profile = developers
region = us-east-1
output = json

[profile developers]
region = us-east-1
output = json
```

## Accessing the cluster using roles

Run the below command

```shell
aws eks update-kubeconfig --region us-east-1 --name acg-demo --profile eks-devs
```

```yaml
cat ~/.kube/config
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUMvakNDQWVhZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJeU1EY3hNVEF6TURjeE5sb1hEVE15TURjd09EQXpNRGN4Tmxvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTlE0CnRab3FuTy9RUG96N0I0Mmg3Vk1yVGVKaDg0VXJkbHZYOVgvbzYrMlRHcVBka0g3VHRnVWpNbGJPVTBiaXFvNTQKYkorNURsbXRlS1daOEc2NEJIM3c4dmpKQWxKSFlMYkhjR0pIOXRaUFBYbHcwVGNiMDlVWEVjcSszK3pBdnhCMgp2OHZwVVVnWGowU0d0UXdxK2d1WkJyci8xU3dUYlBMZmFvcllKZmkybFQ2dEg3QlkvVUx5WjcwWStwc2hMRW83Cnp6YXdMNW02ZmtJR2ZxMnVZRWJRYnpaZG52MlZUWDV5KzdMeXQ3YkJxK3E0L3EvTERkY3h2eDZyb0ZRZVVvM0QKNk0vcFFpVjE1ZnpFKzNvMEVhbmlWZTJMa0xJcE05ajlXelR3Ni83TEcvOWppVnVXdTFKT2svckRxZm9xNkRpaApwT09CNHl4eTlkM2F0U0tUU3lNQ0F3RUFBYU5aTUZjd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0hRWURWUjBPQkJZRUZJMlE5YmdRSGxXUDVDd2lTL2J3UjhWSkJLbGFNQlVHQTFVZEVRUU8KTUF5Q0NtdDFZbVZ5Ym1WMFpYTXdEUVlKS29aSWh2Y05BUUVMQlFBRGdnRUJBSHV0WFJwMEdSbkpEWEY1b05ncQo2bXFObXVEN3RoZGRyNDgweWZOdk1IQXRZOXBzTTc0TDlFNGZVQUZPam1YMUIrZnNHZkhBMUd3Z3VhKys2MnpTCmdVWEZpTXB4emJpS3llaTFIQTh5QThQamJQUG1CTFMwRHBLODVVeHJJMnBibVdkN0R3SUpGU2VicFhUa28wQlAKNHpnbTlScU5tM1J5eDFId1Y1bVZVekx0Y0Jva1AzTkZISTQwWFBLU1ZVeW5JRnJMRUZBMG1FampHQXgrQ0J5RwpweXFsa2ZIeEszM0RrcFV6OVpUVnhXSkh6a2d1KzlTcjlxcndhZ2FlTmZXK3dNM1pmNkJUdkxxUDNrdHVtMUNVCkYvTTdiY3Jsbzd0cVNuZUN1eU5na2R4STY4cUpPdE9oalVmWWUvWkNQMXhKMGQ2RHFlNTRSTFVhQU95Ry9iUVkKUEw4PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    server: https://937E5541B0D5FBF0EF9ED16D09FE0B79.gr7.us-east-1.eks.amazonaws.com
  name: arn:aws:eks:us-east-1:<ACCOUNT_NUMBER>:cluster/acg-demo
contexts:
- context:
    cluster: arn:aws:eks:us-east-1:<ACCOUNT_NUMBER>:cluster/acg-demo
    user: arn:aws:eks:us-east-1:<ACCOUNT_NUMBER>:cluster/acg-demo
  name: arn:aws:eks:us-east-1:<ACCOUNT_NUMBER>:cluster/acg-demo
current-context: arn:aws:eks:us-east-1:<ACCOUNT_NUMBER>:cluster/acg-demo
kind: Config
preferences: {}
users:
- name: arn:aws:eks:us-east-1:<ACCOUNT_NUMBER>:cluster/acg-demo
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      args:
      - --region
      - us-east-1
      - eks
      - get-token
      - --cluster-name
      - acg-demo
      command: aws
      env:
      - name: AWS_PROFILE
        value: eks-devs
```

```
kubectl get deployments
Error from server (Forbidden): deployments.apps is forbidden: User "developers" cannot list resource "deployments" in API group "apps" in the namespace "default"
```

We will get `Forbidden` error as we have not granted access to developers group to get deployments

## Practice Exercise
---

Create `Role` and a `RoleBinding` for interns and grant then to list pods and deployments to only `production` namespace.

Map the `interns` group to IAM role `arn:aws:iam::<ACCOUNT_NUMBER>:role/interns`

## Additional references:
---

- [https://blog.aquasec.com/kubernetes-authorization](https://blog.aquasec.com/kubernetes-authorization)
- [https://aws.amazon.com/blogs/containers/kubernetes-rbac-and-iam-integration-in-amazon-eks-using-a-java-based-kubernetes-operator/](https://aws.amazon.com/blogs/containers/kubernetes-rbac-and-iam-integration-in-amazon-eks-using-a-java-based-kubernetes-operator/)
- [https://www.weave.works/blog/kubernetes-rbac-101](https://www.weave.works/blog/kubernetes-rbac-101)