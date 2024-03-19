# IRSA (IAM Roles for Service Accounts)

## 목표
- Pod가 Amazon DynamoDB에 접근을 하기위해 권한을 붙여주는 작업 진행
- 접근을 하기 위한 Policy가 필요한데, 해당 Policy를 IRSA를 통해 붙여주는 작업

--- 
### 1. Security
1. Security
![image](https://github.com/devhyunuk/eks-security/assets/49749510/516ccbf4-07cd-4bdd-ba32-e470fe450599)

2. IRSA (IAM Roles for Service Accounts)
- kubernetes 안에 존재하는 pod가 ServiceAccount라는 계정 IAM Role를 활용하여 AWS Service에 접근/사용이 가능하게 만드 방식

3. 현재 kubernetes Architecture 상태
- Amazon DynamoDB 테이블 생성
- DynamoDB 테이블에 액세스하기 위해 AmazonEKS 워크로드에 대한 IAM 역할을 생성
- Amazon EKS 클러스터에 AWS Load Balancer Controller 설치

4. carts pod 확인
![image](https://github.com/devhyunuk/eks-security/assets/49749510/8f927edf-05b4-441e-8416-2320baf1a7b5)

- carts pod 확인
```
kubectl -n carts get pod 
```
![image](https://github.com/devhyunuk/eks-security/assets/49749510/e9f0ab54-b0a0-4e94-9aab-d1c6182a7047)
- carts pod는 lightweight DynamoDB service인 carts-dynamodb를 사용하고 있는 것을 확인

- carts의 환경변수 검사하여 애플리케이션이 이를 사용하고 있는지 확인
```
kubectl -n carts exec deployment/carts -- env | grep CARTS_DYNAMODB_ENDPOINT
```
![image](https://github.com/devhyunuk/eks-security/assets/49749510/cbd85538-f6ef-4b54-ac64-de37fb119911)

5. 현재 carts pod의 설정 yaml 확인
```
kubectl -n carts get -o yaml cm carts
```
![image](https://github.com/devhyunuk/eks-security/assets/49749510/4df756d2-e305-4763-ac59-d463ffbe0bc0)
- 현재는 로컬 DynamoDB를 바라고 있음




