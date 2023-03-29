# ConfigMap이란
key-value 형태로 데이터를 저장하는데 사용하는 api 오브젝트  
쿠버네티스 메타데이터를 파드에 실행 중인 애플리케이션에 노출하는데 사용됨  
설정 데이터를 저장하는 쿠버네티스 리소스를 일컫음   

  
파드 안에 환경변수를 정의하는 방식  
```
kind: Pod 
spec:
  containers:
  - image: luksa/fortune:env
    env:
    - name: INTERVAL
      value: "30"
```
환경변수는파드 레벨이 아닌 컨테이너 정의 안에 설정  
환경에 따라 다르거나 자주 변경되는 설정 옵션을 애플리케이션 소스 코드와 별도로 유지해야 함  
-> 즉, 설정을 파드 정의에서 빆으로 이동시 켜야 한다는 것  

쿠버네티스에서는 설정 옵션을 컨피그맵이라 부르는 별도 오브젝트로 분리  

컨피그맵의 내용은 컨테이너의 환경변수 또는 볼륨 파일로 전달  

파드는 컨피그맵을 이름으로 참조하기 때문에, 모든 환경에서 동일한 파드 정의를 사용해 각 환경에서 서로 다른 설정을 사용  


- 컨피그맵을 생성하는 방법  
```
kubectl create configmap fortune-config --from-literal=sleep-interval=25

# or

kubectl create -f fortune-config.yaml

# or

kubectl create configmap my-config --from-file=config-file.conf

# or
kubectl create configmap my-config --from-file=/path/to/dir
```


![스크린샷 2023-03-29 오후 7 20 34](https://user-images.githubusercontent.com/16679473/228504310-38856da3-80a9-4898-b5e6-bf5c985af00a.png)


<img width="470" alt="스크린샷 2023-03-29 오후 7 22 02" src="https://user-images.githubusercontent.com/16679473/228504627-eb0c96ec-df96-4fa7-980b-03fcc8db9433.png">



<img width="787" alt="스크린샷 2023-03-29 오후 7 25 18" src="https://user-images.githubusercontent.com/16679473/228505389-d8a8bcf1-e5b5-4a5b-8777-8c783ba03983.png">



### 컨테이너에 어떻게 전달할까?
1. env에 valueFrom 필드 사용
<img width="646" alt="스크린샷 2023-03-29 오후 7 26 35" src="https://user-images.githubusercontent.com/16679473/228505686-f1f4855d-d174-4f4b-b22a-4dd2e1efd345.png">


2. envFrom 필드 사용
```
spec:
  containers:
  - image: some-image
    envFrom: 
    - prefix: CONFIG_
    configMapRef:
      name: fortune-config
```
fortune-config 라는 컨피그맵에 정의된 모든 값을 CONFIG_ 접두사를 붙여 파드에 환경 변수로 노출시킴 (규칙에 맞는 경우만! 대시(-) 안됨, key=value 형태)
- 접두사는 선택사항, 생략하면 키와 동일한 이름으로 환경변수가 셋팅됨

3. 명령줄 인자로 전달  
 <img width="547" alt="스크린샷 2023-03-29 오후 7 35 38" src="https://user-images.githubusercontent.com/16679473/228507796-905af555-0e1a-4bbd-ad8e-e33ad6f3605b.png">
환경변수로 셋팅하고, 쿠버네티스가 해당 값을 컨테이너로 전달  
ex) xxx-loop.sh $(INTERNAL)


4. 컨피그맵 볼륨을 사용하여 컨피그맵 항목을 파일로 노출
![스크린샷 2023-03-29 오후 8 00 03](https://user-images.githubusercontent.com/16679473/228513464-b97e2134-346f-4cf5-b859-2565f9003433.png)

파드 내의 nginx 컨테이너에서 해당 컨피그맵의 값을 읽어 nginx.conf를 구성할 수 있도록 볼륨을 마운트
```
apiVersion: v1 
kind: Pod 
metadata:
  name: fortune-configmap-volume 
spec:
  containers:
  - image: nginx:alpine
    name: web-server 
    volumeMounts:
    ...
    - name: config
      mountPath: /etc/nginx/conf.d 
      readonly: true
  volumes:
  - name: config 
    configMap:
      name: test-config
```
위의 결과로, /etc/nginx/conf.d 해당 위치에 nginx.conf와 sleep-internal 파일이 마운트되게 됨  


</br>

불필요한 파일이 함께 마운트되었는데, 개별 컨피그맵 항목을 파일로 마운트하려면??
<img width="836" alt="스크린샷 2023-03-29 오후 8 23 32" src="https://user-images.githubusercontent.com/16679473/228520484-aa3d5a02-acee-4cb3-97cd-7ef74fdd6768.png">



!! 존재하지 않는 컨피그맵을 사용하는 파드는 컨테이너를 시작하는 데 실패함  
단, configMapKeyRef.optional: true로 지정하면, 컨피그맵이 존재하지 않아도 컨테이너는 시작됨


* 컨피그맵 볼륨의 파일 권한은 default 644 (-rw-r-r--)
 - defaultMode 키워드로 변경 가능

컨피그맵을 업데이트하면, 이를 참조하는 모든 볼륨 파일이 업데이트됨 (단, 최대 1분까지 업데이트가 걸릴 수 있음)
컨피그맵 볼륨안의 파일은 심볼링 링크로 되어있어, 모든 파일을 한 번에 효과적으로 변경함
 - 전체 볼륨인 경우만
 - 단일 파일을 컨테이너에 마운트한 경우 파일이 업데이트 되지 않음!

컨피그맵을 변경할 수 없도록 설정하는 옵션을 제공함
- 애플리케이션 중단을 일으킬 수 있는 우발적(또는 원하지 않는) 업데이트로부터 보호
- immutable로 표시된 컨피그맵에 대한 감시를 중단하여, kube-apiserver의 부하를 크게 줄임으로써 클러스터의 성능을 향상시킴
```
apiVersion: v1
kind: ConfigMap
metadata:
  ...
data:
  ...
immutable: true
```



</br></br></br></br></br></br></br>

## 시크릿??  
자격증명, 개인 암호화 키, 보안을 유지해야하는 유사한 데이터가 포함되는 경우 시크릿을 이용하여 저장   
컨피그맵과 비슷하게 key=value 쌍을 가진 맵  
노드는 시크릿을 항상 메모리에만 저장되게 함   
 - secret 볼륨은 시크릿 파일을 저장하는 데 인메모리 파일시스템(tmpfs)을 사용
 - 민감한 데이터를 노출시킬 수도 있는 디스크에 저장하지 않기 위해서
마스터 노드(구체적으로 etcd)에는 시크릿을 암호화되지 않은 형식으로 저장하므로, 시크릿에 저장한 민감한 데이터를 보호하려면 마스터 노드를 보호하는 것이 필요
  
컨테이너에 시크릿을 전달할 수 있는 방법
1. 시크릿 항목을 볼륨 파일로 노출
2. 환경변수로 시크릿 항목을 컨테이너에 전달

</br>


모든 파드에는 시크릿 볼륨이 자동으로 연결되어 있음
- deafult-token 시크릿  
![스크린샷 2023-03-29 오후 8 46 12](https://user-images.githubusercontent.com/16679473/228525690-31323587-cef2-44a7-9476-208a6c369d8f.png)

- deafult-token 시크릿이 갖고 있는 세 가지 항목(ca.crt, namespace, token)은 파드 안에서 쿠버네티스 API 서버와 통신할 때 필요한 모든 것을 나타냄  
![스크린샷 2023-03-29 오후 8 47 34](https://user-images.githubusercontent.com/16679473/228526044-9b4f4b52-fd13-4f49-bffd-70a8f07c1104.png)



시크릿 생성  
`kubectl create secret generic fortune-https --from-file=https.cert`
- fortune-https 이름을 가진 generic 시크릿 생성  
- 시크릿 항목의 내용은 Base64 인코딩 문자열로 표시됨  
![스크린샷 2023-03-29 오후 8 52 49](https://user-images.githubusercontent.com/16679473/228527245-bd3581ee-b112-4067-9d6d-de26f6976da8.png)  

왜 Base64 인코딩을 사용할까?
- 바이너리 값도 담을 수 있기 때문
- base64 인코딩은 바이너리 데이터를 일반 텍스트 형식인 YAML이나 JSON 안에 넣을 수 있기 때문
- 시크릿의 최대 크기는 1MB로 제한


일반 텍스트를 시크릿에 추가하는 방법?
stringData 필드를 사용(쓰기 전용 필드)
stringData 필드를 사용하여 시크릿을 생성하면, 다른 바이너리 형태와 동일하게 base64로 인코딩되어 data 항목 아래에 표시됨  
![스크린샷 2023-03-29 오후 9 01 16](https://user-images.githubusercontent.com/16679473/228529143-3efe960c-182f-41b6-b8c0-beefefc027bb.png)  


볼륨을 통해 시크릿을 컨테이너에 노출시키면, 시크릿 값은 디코딩되어 파일에 기록됨  
 -> 즉, 사용할 떄 신경쓰지 않아도 된다는 의미   


파드에서 시크릿을 사용하는 방법  
```
apiVersion: v1 
kind: Pod 
metadata:
  name: fortune-https
spec:
  containers:
  - image: nginx:alpine
    name: web-server 
    volumeMounts:
    - name: certs
      mountPath: /etc/nginx/certs 
      readonly: true
  volumes:
  - name: certs 
    secret:
      name: fortune-https
```

<img width="725" alt="스크린샷 2023-03-29 오후 9 10 02" src="https://user-images.githubusercontent.com/16679473/228531014-1508a401-eae2-41f7-b9ef-8474bfe22dc8.png">


</br>

환경변수로 시크릿을 노출하는 법
<img width="503" alt="스크린샷 2023-03-29 오후 9 12 32" src="https://user-images.githubusercontent.com/16679473/228531615-67b6d6f4-5576-469e-bbbe-de8ea382fec3.png">  
시크릿 개별 항목을 환경변수로 노출할 수 있음  
secretKeyRef를 사용해 시크릿을 참조   
좋은 방법은 아님  
 -> 환경변수를 기록하거나 어플리케이션을 시작하면서 로그에 환경변수를 남겨 의도치 않게 시크릿을 노출할 수 있음  


시크릿 종류
- generic
- tls
- docker-registry :  도커 레지스트리 인증을 위한 시크릿
```
kubectl create secret docker-registry mydockerhubsecret --docker-username=myusername --docker-password=mypassword
```


파드에서 도커 레지스트리 시크릿 사용  
<img width="496" alt="스크린샷 2023-03-29 오후 9 16 56" src="https://user-images.githubusercontent.com/16679473/228532652-c8b18faa-adec-4630-8c7d-d44dfdb41128.png">  
모든 파드에 설정할 필요 없음 (나중에 서비스어카운트를 통해 할 것)

