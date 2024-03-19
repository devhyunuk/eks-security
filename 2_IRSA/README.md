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

6. configmap에 DYNAMODB_TABLENAME 변
```
echo $CARTS_DYNAMODB_TABLENAME

kubectl kustomize ~/environment/eks-workshop/modules/security/irsa/dynamo | envsubst | kubectl apply -f-
```
![image](https://github.com/devhyunuk/eks-security/assets/49749510/c8d1a96f-abad-4a71-8fb3-9a9d2c9e3b57)

- configmap이 변경되었음을 확인
```
kubectl get -n carts cm carts -o yaml
```
![image](https://github.com/devhyunuk/eks-security/assets/49749510/b614bd9b-c2e5-4aea-8f77-a95eafc5f508)
- CARTS_DYNAMODB_TABLENAME: eks-workshop-carts 로 변경이 되어있음을 확인

7. carts deployment를 재시작 하여 변경된 configmap을 적용
```
kubectl rollout restart -n carts deployment/carts
kubectl rollout status -n carts deployment/carts
```
![image](https://github.com/devhyunuk/eks-security/assets/49749510/013f82e4-a678-4d01-a853-a31f36d37aff)
- 새로운 configmap을 적용 후 LB에 접속하면 500에러 발생
![image](https://github.com/devhyunuk/eks-security/assets/49749510/236a7bb1-5cc7-4b8e-84e4-71fa571e034b)
- 500 에러 발생 이유 : 로컬 DDB로 잘 사용하고 있다가 AWS DDB로 변경시 권한이 없어서 에러가 발생 (권한 부족)

8. 500 에러 로그 확인
```
kubectl logs -n carts deployment/carts
```
![image](https://github.com/devhyunuk/eks-security/assets/49749510/e058ad45-5cbd-4ac4-9de2-130f095b7d25)
- not authorized to perform: dynamodb 에러 확인

9. IRSA를 활용하여 Pod에 권한을 적용
- OIDC Provider를 EKS에 활성화 하도록 설정된 상태
- OIDC Provider가 있어야 IRSA를 적용가
![image](https://github.com/devhyunuk/eks-security/assets/49749510/d799ada2-8d35-4e00-8544-ba1a363a253a)

- OIDC Provider 확인
```
hyunwook.kang:~/environment $ aws iam list-open-id-connect-providers
{
    "OpenIDConnectProviderList": [
        {
            "Arn": "arn:aws:iam::308943070041:oidc-provider/oidc.eks.us-west-2.amazonaws.com/id/98F1AA37C4FF74DE45421DFB7242A84B"
        }
    ]
}
```
- Amazon EKS 클러스터와의 연관성을 검증 확인
```
hyunwook.kang:~/environment $ aws eks describe-cluster --name ${EKS_CLUSTER_NAME} --query 'cluster.identity'
{
    "oidc": {
        "issuer": "https://oidc.eks.us-west-2.amazonaws.com/id/98F1AA37C4FF74DE45421DFB7242A84B"
    }
}
```

- 서비스가 DynamoDB 테이블을 읽고 쓰는 데 필요한 권한을 제공하는 IAM 역할이 carts생성. 정책 확인
```
hyunwook.kang:~/environment $ aws iam get-policy-version \
>   --version-id v1 --policy-arn \
>   --query 'PolicyVersion.Document' \
>   arn:aws:iam::${AWS_ACCOUNT_ID}:policy/${EKS_CLUSTER_NAME}-carts-dynamo | jq .
{
  "Statement": [
    {
      "Action": "dynamodb:*",
      "Effect": "Allow",
      "Resource": [
        "arn:aws:dynamodb:us-west-2:308943070041:table/eks-workshop-carts",
        "arn:aws:dynamodb:us-west-2:308943070041:table/eks-workshop-carts/index/*"
      ],
      "Sid": "AllAPIActionsOnCart"
    }
  ],
  "Version": "2012-10-17"
}
```

- Carts 구성 요소에 대한 ServiceAccount인 한 EKS 클러스터와 연결된 OIDC 공급자가 이 역할을 맡을 수 있도록 적절한 신뢰 관계로 역할 구성 확인.
```
hyunwook.kang:~/environment $ aws iam get-role \
>   --query 'Role.AssumeRolePolicyDocument' \
>   --role-name ${EKS_CLUSTER_NAME}-carts-dynamo | jq .

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::308943070041:oidc-provider/oidc.eks.us-west-2.amazonaws.com/id/98F1AA37C4FF74DE45421DFB7242A84B"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.us-west-2.amazonaws.com/id/98F1AA37C4FF74DE45421DFB7242A84B:sub": "system:serviceaccount:carts:carts"
        }
      }
    }
  ]
}
```

- 사용할 IAM 역할을 확인한 후 > Kustomize를 실행 > 서비스 계정에 변경 사항을 적용
```
hyunwook.kang:~/environment $ kubectl kustomize ~/environment/eks-workshop/modules/security/irsa/service-account \
>   | envsubst | kubectl apply -f-
namespace/carts unchanged
serviceaccount/carts configured
configmap/carts unchanged
service/carts unchanged
service/carts-dynamodb unchanged
deployment.apps/carts unchanged
deployment.apps/carts-dynamodb unchanged
```

- 적용 사항
#### - Kustomization.yaml
```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../dynamo
patches:
- path: carts-serviceAccount.yaml
```

#### - carts-serviceAccount.yaml
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: carts
  namespace: carts
  annotations:
    eks.amazonaws.com/role-arn: ${CARTS_IAM_ROLE}
```

- 
```
hyunwook.kang:~/environment $ kubectl describe sa carts -n carts | grep Annotations
Annotations:         eks.amazonaws.com/role-arn: arn:aws:iam::308943070041:role/eks-workshop-carts-dynamo
```

```
hyunwook.kang:~/environment $ kubectl rollout restart -n carts deployment/carts
deployment.apps/carts restarted

hyunwook.kang:~/environment $ kubectl rollout status -n carts deployment/carts
Waiting for deployment "carts" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "carts" rollout to finish: 1 old replicas are pending termination...
deployment "carts" successfully rolled out
```

- AWS DynamoDB 접속 확인
![image](https://github.com/devhyunuk/eks-security/assets/49749510/693380c0-f760-40b5-8c4f-a2c57d50e53d)





























