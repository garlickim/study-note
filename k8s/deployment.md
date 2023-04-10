# deployment


### 디플로이먼트 생성
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```
nginx-deployment 이름으로 디플로이먼트가 생성됨  
3개의 레플리카 파드를 생성  
디플로이먼트는 파드 템플릿에 정의된 레이블(app:nginx)를 선택함  

</br> 

`kubectl apply -f nginx-deployment.yaml`

</br>

kubectl get deployments 을 실행해서 디플로이먼트가 생성되었는지 확인  

![스크린샷 2023-04-10 오후 9 31 54](https://user-images.githubusercontent.com/16679473/230902028-bfb9c645-ba9a-4563-b08e-d111ade76174.png)  
(생성중)  
![스크린샷 2023-04-10 오후 9 33 45](https://user-images.githubusercontent.com/16679473/230902249-daf7ecaa-e4ca-4482-a73d-df2c8165b145.png)
(생성 완료)  

NAME 은 네임스페이스에 있는 디플로이먼트 이름의 목록  
READY 는 사용자가 사용할 수 있는 애플리케이션의 레플리카의 수를 표시. ready/desired 패턴을 따름  
UP-TO-DATE 는 의도한 상태를 얻기 위해 업데이트된 레플리카의 수를 표시  
AVAILABLE 은 사용자가 사용할 수 있는 애플리케이션 레플리카의 수를 표시  
AGE 는 애플리케이션의 실행된 시간을 표시  

</br>


**디플로이먼트의 롤아웃 상태를 보려면?**
`kubectl rollout status deployment/nginx-deployment` 실행  
![스크린샷 2023-04-10 오후 9 35 20](https://user-images.githubusercontent.com/16679473/230902500-2a34285c-e9b8-4104-bbda-70cf25076b1f.png)  

</br>

**디플로이먼트로 생성된 레플리카셋(rs)을 보려면??**
kubectl get rs 를 실행  
![스크린샷 2023-04-10 오후 9 36 55](https://user-images.githubusercontent.com/16679473/230902663-dbe17ddd-4a77-41cc-8a1d-14417614be66.png)  



레플리카셋의 이름은 항상 [DEPLOYMENT-NAME]-[HASH] 형식   
HASH 문자열은 레플리카셋의 pod-template-hash 레이블과 같음



</br>


### 디플로이먼트 업데이트
디플로이먼트의 파드 템플릿(spec.template)이 변경된 경우에만 디플로이먼트의 롤아웃이 트리거(trigger) 됨   
템플릿의 레이블이나 컨테이너 이미지가 업데이트된 경우  

```
#### [1] 
# nginx 1.16.1 이미지를 사용하도록 파드 업데이트 (1.14.2 -> 1.16.1)
kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.16.1

# OR
kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1

# OR 디플로이먼트를 edit하여 image를 직접 수정
kubectl edit deployment/nginx-deployment


#### [2] 롤아웃 상태 확인
kubectl rollout status deployment/nginx-deployment
```

kubectl get rs 으로 레플리카셋을 확인하면, 아래와 유사하게 출력됨   
이후, 기존 레플리카셋은 제거됨   
![스크린샷 2023-04-10 오후 9 44 22](https://user-images.githubusercontent.com/16679473/230903684-5575aac9-37dd-4bd3-85af-6d7e2acaece2.png)   


디플로이먼트는 업데이트되는 동안 일정한 수의 파드만 중단되도록 보장  
기본적으로 적어도 의도한 파드 수의 75% 이상이 동작하도록 보장(최대 25% 불가)   

먼저 새로운 파드를 생성한 다음, 이전 파드를 삭제하고, 또 다른 새로운 파드를 만듦  
충분한 수의 새로운 파드가 나올 때까지 이전 파드를 죽이지 않으며, 충분한 수의 이전 파드들이 죽기 전까지 새로운 파드를 만들지 않음   


### 디플로이먼트 세부정보 확인
```
kubectl describe depolyments
```
![스크린샷 2023-04-10 오후 9 49 25](https://user-images.githubusercontent.com/16679473/230904388-62129755-dd8e-4180-92c0-564683bf8bca.png)  
디플로이먼트에 일어난 이벤트 이력을 확인할 수 있음   


### Rollover(multiple updates in-flight)
디플로이먼트 컨트롤러는 각 시간마다 새로운 디플로이먼트에서 레플리카셋이 의도한 파드를 생성하고 띄우는 것을 주시
만약 디플로이먼트가 업데이트되면, 기존 레플리카셋에서 spec.selector 레이블과 일치하는 파드를 컨트롤 하지만, 템플릿과 spec.template 이 불일치하면 스케일 다운이 됨   
예를 들어, 디플로이먼트로 nginx:1.14.2 레플리카를 5개 생성함  
하지만 nginx:1.14.2 레플리카 3개가 생성되었을 때 디플로이먼트를 업데이트해서 nginx:1.16.1 레플리카 5개를 생성성하도록 업데이트를 한다고 가정  
이 경우 디플로이먼트는 즉시 생성된 3개의 nginx:1.14.2 파드 3개를 죽이기 시작하고 nginx:1.16.1 파드를 생성하기 시작  

</br>


### 레이블 셀렉터 업데이트
일반적으로 레이블 셀렉터를 업데이트 하는 것을 권장하지 않으며 셀렉터를 미리 계획하는 것을 권장
**API 버전 apps/v1 에서 디플로이먼트의 레이블 셀렉터는 생성 이후에는 변경할 수 없음**
</br>

셀렉터 추가 시 디플로이먼트의 사양에 있는 파드 템플릿 레이블도 새 레이블로 업데이트해야 함. 그렇지 않으면 유효성 검사 오류가 반환됨  
- 이 변경은 겹치지 않는 변경으로 새 셀렉터가 이전 셀렉터로 만든 레플리카셋과 파드를 선택하지 않게 되고, 그 결과로 모든 기존 레플리카셋은 고아가 되며, 새로운 레플리카셋을 생성하게됨  

셀렉터 업데이트는 기존 셀렉터 키 값을 변경하며, 결과적으로 추가와 동일한 동작을 함  

셀렉터 삭제는 디플로이먼트 셀렉터의 기존 키를 삭제하며 파드 템플릿 레이블의 변경을 필요로 하지 않음
- 기존 레플리카셋은 고아가 아니고, 새 레플리카셋은 생성되지 않으나 제거된 레이블은 기존 파드와 레플리카셋에 여전히 존재한다는 점


### 디플로이먼트 롤백
모든 디플로이먼트의 롤아웃 기록은 시스템에 남아있어 언제든지 원할 때 롤백이 가능  

디플로이먼트의 수정 버전은 디플로이먼트 롤아웃시 생성됨   
디플로이먼트 파드 템플릿 (spec.template)이 변경되는 경우에만 새로운 수정 버전이 생성된다는 것을 의미   
(디플로이먼트의 스케일링과 같은 다른 업데이트시 디플로이먼트 수정 버전은 생성되지 않음)   

이전 수정 버전으로 롤백을 하는 경우에 디플로이먼트 파드 템플릿 부분만 롤백된다는 것을 의미   


### 디플로이먼트의 롤아웃 기록 확인
```
kubectl rollout history deployment/nginx-deployment
```
![스크린샷 2023-04-10 오후 10 02 11](https://user-images.githubusercontent.com/16679473/230906255-297d95aa-283f-430d-b568-f81fa4f124e6.png)   

</br>

각 수정 버전의 세부 정보를 보려면 아래와 같이 실행   
```
kubectl rollout history deployment/nginx-deployment --revision=2
```
![스크린샷 2023-04-10 오후 10 04 21](https://user-images.githubusercontent.com/16679473/230906580-f3556a32-90da-44f3-9bda-e485f5c26fd2.png)   


### 이전 수정 버전으로 롤백
디플로이먼트를 현재 버전에서 이전 버전인 버전 2로 롤백
```
# 현재 롤아웃의 실행 취소 및 이전 수정 버전으로 롤백
kubectl rollout undo deployment/nginx-deployment

# 특정 수정 버전으로 롤백하려면 --to-revision 옵션에 해당 수정 버전을 명시
kubectl rollout undo deployment/nginx-deployment --to-revision=2
```

## 디플로이먼트 스케일링
```
kubectl scale deployment/nginx-deployment --replicas=10
```

클러스터에서 horizontal Pod autoscaling를 설정 한 경우 디플로이먼트에 대한 오토스케일러를 설정할 수 있음
그리고 기존 파드의 CPU 사용률을 기준으로 실행할 최소 파드 및 최대 파드의 수를 선택할 수 있음
```
kubectl autoscale deployment/nginx-deployment --min=10 --max=15 --cpu-percent=80
```

### 비례적 스케일링(Proportional Scaling)
디플로이먼트 롤링업데이트는 여러 버전의 애플리케이션을 동시에 실행할 수 있도록 지원함   
사용자 또는 오토스케일러가 롤아웃 중에 있는 디플로이먼트 롤링 업데이트를 스케일링 하는 경우(진행중 또는 일시 중지 중),   
디플로이먼트 컨트롤러는 위험을 줄이기 위해 기존 활성화된 레플리카셋(파드와 레플리카셋)의 추가 레플리카의 균형을 조절함   
이것을 proportional scaling 라 함  

</br>

예를 들어, 10개의 레플리카를 디플로이먼트로 maxSurge=3, 그리고 maxUnavailable=2 로 실행
kubectl get deploy

