# Settings

--- 
### 1. EKS Cluster 생
1. eksctl을 사용해서 Amazon EKS 클러스터를 생성 (eks-security-cluster.yaml를 활용)
```
export EKS_CLUSTER_NAME=eks-workshop

eksctl create cluster -f eks-security-cluster.yaml
```
- 3개의 가용영역을 포함한 VPC 생성
- 한개의 EKS 클러스터 생성
- IAM OIDC provider 생성 (iam: withOIDC: true)
- managedNodeGroups(관리형 노드 그룹) 생성 (name : default)

![image](https://github.com/devhyunuk/eks-security/assets/49749510/b7c77f3f-f480-4fd3-a1cc-13c8b3a99576)

