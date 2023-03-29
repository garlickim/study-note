#시크릿??  
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
