# Settings

--- 
### 1. Cloud9 생성
1. eksctl을 사용해서 Amazon EKS 클러스터를 생성 (eks-security-cluster.yaml를 활용)
```
export EKS_CLUSTER_NAME=eks-workshop

eksctl create cluster -f eks-security-cluster.yaml
```
![image](https://github.com/devhyunuk/eks-security/assets/49749510/b7c77f3f-f480-4fd3-a1cc-13c8b3a99576)
