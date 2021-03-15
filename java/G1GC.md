# GC란?
동적으로 할당된 메모리 영역 중 더이상 사용하지 않는 영역(unreachable object)에 대하여 메모리를 해제하는 기능

## Reachable Object 탐색
- GC Root로 부터 object를 탐색
- GC Root가 될 수 있는 조건은?
  - stack 영역의 데이터
  - method 영역의 static 데이터
  - JNI에 의해 생성된 객체

## 일반적인 GC 과정
**Mark And Sweep** : GC Root로 부터 변수를 스캔 -> 해당 변수가 참조하는 객체를 찾아 **마킹** -> 마킹되지 않은 객체는 Heap 메모리에서 제거  
**Compact** : sweep 이후 파편화되어 있는 메모리 조각을 모음

</br>

# G1GC
- jdk9+ 부터 기본 GC
- 기존 GC는 메모리의 연속된 공간을 나눠서 사용했다면, G1GC는 대략 2048개의 region으로 이루어짐
- region은 메모리 할당/확보의 단위
- 논리적으로 Young Gen(eden, survivor), Old Gen,.. 등으로 나눔  
![heap layout](/img/g1gc-layout.png)  
출처 [https://docs.oracle.com/en/java/javase/11/gctuning/garbage-first-garbage-collector](https://docs.oracle.com/en/java/javase/11/gctuning/garbage-first-garbage-collector.html#GUID-15921907-B297-43A4-8C48-DC88035BC7CF)

- 그림에서 보여지는 "H"는 Humagous Object로 1 region 크기의 절반보다 큰 데이터로 여러 영역을 차지하고 있는 객체
- 일반적으로 데이터는 young gen에 할당 됨. humongous objects만 old gen에 directly로 할당 됨.


## Allocation (Evacuation) Failure
- GC 수행 중, 라이브 데이터를 copy 하여 새로운 영역에 할당하려 할 때, 더이상 사용 가능한 영역을 찾을 수 없는 경우 stop-the-world(full GC)가 일어남

## Floating Garbage
- G1GC는 snapshot-at-the-beginning (SATB) 라 불리는 기술을 사용하여 GC를 수행함  
  > GC 시작 전, 객체에 대한 snapshop을 뜨고 해당 snapshot을 기반으로 객체를 탐색함
  > snapshot을 뜨는 당시, 라이브 객체였지만 실제 수행중 가비지가 되었더라도 해당 객체는 GC 대상이 되지 않음
  > 즉, G1GC는 실시간 GC가 아님
  
## Card Tables and Concurrent Phases
- Card Table 이란?
  > 바이트 배열로 이루어진 메모리 구조, old gen 에서 young gen을 참조하는 객체의 포인터 정보를 가지고 있음 (dirty card)
  > young gen의 GC 실행시, card table을 참조하여 GC 대상을 파악함 (dirty card만 검색하면 됨)
- Concurrent Phases
  > Concurrent marking
    - initial mark 에서 살아 있는 객체로 판단한 객체들의 참조를 따라가면서 마킹 (다른 스레드와 동시에 진행)
    - 이어서 새로 추가되거나 참조가 끊긴 remark 단계까지 진행 (다른 스레드와 동시에 진행)
  > Concurrent cleanup
    - unreachable object를 지우고, 비워진 영역을 사용 가능한 영역의 목록에 추가함

## Remembert set
- region 내의 참조를 관리하는 방법
- 전체 heap의 5% 미만의 공간을 remember set으로 할당함
    
## Starting a Concurrent Collection Cycle
- Concurrent Collection Cycle(concurrent marking phase)이 시작되는 시점은 JDK8 기준 -XX:InitiatingHeapOccupancyPercent=\<NN\> 해당 옵션에 의해 결정 됨  
  (default 45%)

## Pause Time Goal
- MaxGCPauseMillis 옵션을 통해 중지시간 목표를 정할 수 있음 (default 200ms)
- MaxGCPauseMillis 사용은 G1이 collection의 young gen의 수를 조정할 것임
  > young gen의 size를 줄여야 잦은 GC 를 돌아 중지 시간을 짧게 가져갈 것임
- 이때, young gen의 size를 지정하는 옵션을 사용한다면 G1이 목표 중지 시간을 달성하는데 문제가 될 수 있음
  
## Young Garbage Collections
- eden, survivor region의 라이브 데이터가 새로운 region으로 옮겨지거나 지워짐
- 데이터의 age에 따라, age 값이 높은 경우 old gen으로 promotion(승격) 됨  
  age 가 낮은 경우 survivor resion으로 옮겨지고 다음번 young/mixed garbage collection의 CSet(collection set)에 포함됨
  
## Mixed Garbage Collections
- Concurrent Collection Cycle이 끝나면, G1은 young collection에서 mixed collection으로 전환 됨
- young(eden, survivor) region 뿐만 아니라 old region도 collection이 일어남 
  (old 영역에 해당되는 region의 marking 정보가 있어서 가능)
- old region이 충분히 collect되면 G1은 다음 marking cycle이 끝날때까지 Young GC를 시도함

## Phases of the Marking Cycle
![marking_cycle](/img/g1gcCycle.png)   
출처 [https://johngrib.github.io/post-img/java-g1gc/g1gc-cycle.png](https://johngrib.github.io/post-img/java-g1gc/g1gc-cycle.png)
- 크게보면 2단계로 나뉘어짐 
  - Young only phase : young gen → old gen으로의 메모리를 채우는 GC
  - Space Reclamation : young gen의 공간을 확보, 추가로 old gen 영역도 재확보
  - Young only phase ---- IHOP 임계치 도달시 ---→ Space Reclamation


1. Initial marking phase 
    - root들을 마킹함. young gc(stw)에 포함된다
2. Root region scanning phase
    - initial marking 단계에서 마킹된 survivor region을 스캔하여 old gen을 참조하는 object를 마킹함
    - 애플리케이션 실행과 동시에(not stw) 일어나며, 다음 stw가 일어나기전에 완료 되어야 함
3. Concurrent marking phase
    - 전체 heap을 대상으로 reachable object 를 찾음
    - 어플리케이션 실행과 동시에 일어나지만, young gc의 stw에 의해 방해받을 수 있음
4. Remark phase
    - marking 마무리, global 참조 처리 및 class 언로드를 수행
    - 완전히 빈 영역을 회수, 내부 데이터 구조를 정리 
    - STW 수집이며 마킹주기를 완료하는 데 도움
    - SATB 버퍼를 비우고 방문하지 않은 라이브 객체를 추적하며 참조 처리를 수행
5. Cleanup phase
    - free region과 mixed gc 후보군을 식별
    - reset되고 빈 region들을 리턴할 때, 부분적으로 동시에 수행됨


</br></br>

## Initiating Heap Occupancy Percent (IHOP)
- Initial Mark collection가 트리거되는 임계치
- jdk8 기준 default 45%,  jdk11 기준 old gen size의 percentage로 결정됨
- marking 시간과 marking 하는 동안 old gen에 할당된 메모리 양을 바탕으로 최적의 IHOP를 자동으로 결정 (Adaptive IHOP)
- Adaptive IHOP가 활성화 된 경우, 임계값을 예측할 값이 부족하다면 현재 old gen의 percentage로 결정됨
- -XX:-G1UseAdaptiveIHOP 옵션을 통해 Disable 가능
  - 이때는 -XX:InitiatingHeapOccupancyPercent 해당 옵션을 통해 임계치를 결정함

- old gen 점유율이 ( 현재 최대 old gen 사이즈 - XX:G1HeapReservePercent ) 일 때, Adaptive IHOP는 공간 회수 단계의 첫번째 mixed gc를 위해 초기 힙 점유율을 set 함



</br>

## Marking
- SATB(Snapshot-At-The-Beginning) 알고리즘 사용
- 초기 Mark 일시 중지시에 heap의 가상 스냅샷을 찍음
- 마킹하는 도중에 죽는 객체도 라이브 객체로 보는 특징이 존재
- Remark 단계에서 적은 지연시간을 제공함

</br>


## Behavior in Very Tight Heap Situations
- 어플리케이션의 많은 메모리 사용으로 더 이상 copy할 region을 찾을 수 없는 경우, allocation failure 발생
- 이미 이동시킨 객체는 그대로 유지, 이동 시키지 않은 객체는 copy 하지 않는 방향으로 진행
- 일반적인 young gc 만큼 속도가 빠름
- GC 마무리 단계에 allocation failure에 대한 조치가 마무리된다는 가정하에 어플리케이션을 무리없이 실행 가능
  - 만약 위와 같은 가정이 깨진다면, Full GC가 일어남

</br>

## Humongous Objects
- resion의 절반 사이즈 보다 크거나 같은 object들
- XX:G1HeapRegionSize 옵션을 사용하여 region 사이즈를 지정하지 않으면, Ergonomic Defaults 에 따라 결정됨
- Humongous Object 는 old gen region에 순서가 있는 연속된 region을 할당 받음
- 시퀀스의 마지막 region의 남은 공간은 전체 object가 회수될 때까지 할당을 위해 손실됨
- unreachable 상태가 되고, cleanup 중지 또는 full gc의 marking이 끝날때만 회수가 가능함
(humongous objects can be reclaimed only at the end of marking during the Cleanup pause, or during Full GC if they became unreachable. )
- bool, 모든 종류의 정수, 부동 소수점 값과 같은 primitive 타입의 배열에 대하여 특별한 provision(규정)이 존재
  - (young,old 어떤 gc든)GC 일시중지 시, 많은 object에 의해 참조되지 않는다면 G1은 humongous object를 회수하려고 시도함
  - XX:G1EagerReclaimHumongousObjects 해당 옵션으로 비활성화가 가능 (default는 활성)
- humongous object의 할당은 gc 일시중지를 조기에(prematurely) 발생 시킬 수 있음
- humongous object 할당시, Initiating Heap Occupancy 임계치를 확인함. 현재 점유율이 임계치를 초과하면 initial mark young collection을 즉시 실행할 수 있음
- Full GC 일지라도 humongous object는 움직이지 않음 --> 공간의 조각화 발생
  - 이에 따라 느린 full gc가 일어나거나 unexpected out-of-memory 발생할 수 있음


</br>

## Young-Only Phase Generation Sizing
- collection 대상은 오직 young generation region
- normal young collection의 끝에 youn generation 사이즈를 측정
  - 실제 pause 시간을 장기간 관찰하여 XX : MaxGCPauseTimeMillis 과 XX : PauseTimeIntervalMillis 에 의해 설정된 pause 목표를 충족함
  - 수집하는 정보에는 collection 하는 동안 복사해야하는 object의 양과 object들이 어떻게 상호 연결되어있는지에 대한 정보가 포함되어 있음
- 따로 고려된 값이 없다면, pause 시간을 충족하기 위해 XX:G1NewSizePercent 과 XX:G1MaxNewSizePercent이 결정한 값 사이에서 사이즈를 조정함


</br>

## Space-Reclamation Phase Generation Sizing
- G1은 single gc pause시 old gen에서 재확보되는 공간의 양을 최대화하고자 함
- young gen region의 사이즈는 XX:G1NewSizePercent에 의해 결정되는 최소값으로 설정됨
- pause 목표 시간을 초과한다고 결정할 때까지 old gen region을 추가함
- 각각의 gc pause시 회수 효율성, 가장 높은 우선 순위, 그리고 final collection set을 얻기 위해 남은 사용 가능한 시간을 순서대로 이전 세대 지역을 추가함
- GC당 take할 old gen region의 수는 collect(수집)할 old gen region의 후보 수(collection set candidate regions) / XX:G1MixedGCCountTarget 에 설정된 사이즈로 결정됨
- 공간 재확보 단계 시작시 점유율이 XX:G1MixedGCLiveThresholdPercent 보다 낮은 모든 old gen region이 collection set candidate region 됨
- 재확보할 수 있는 공간이 XX:G1HeapWastePercent 설정된 값보다 작으면 공간재확보 단계는 종료됨

</br>

## Ergonomic Defaults for G1 GC
- G1이 사용하는 기본값에 대한 정보
- 추가 옵션없이 G1 사용시, 예상되는 동작과 리소스 사용량에 대한 정보를 제공

| Option and Default Value | Description |  
| ------------------------------ | ---------------- |  
| -XX:MaxGCPauseMillis=200 | The goal for the maximum pause time. |  
| -XX:GCPauseTimeInterval=\<ergo\> | The goal for the maximum pause time interval. By default G1 doesn’t set any goal, allowing G1 to perform garbage collections back-to-back in extreme cases. |  
| -XX:ParallelGCThreads=\<ergo\> | The maximum number of threads used for parallel work during garbage collection pauses. This is derived from the number of available threads of the computer that the VM runs on in the following way: if the number of CPU threads available to the process is fewer than or equal to 8, use that. Otherwise add five eighths of the threads greater than to the final number of threads. </br> </br> At the start of every pause, the maximum number of threads used is further constrained by maximum total heap size: G1 will not use more than one thread per -XX:HeapSizePerGCThread amount of Java heap capacity. | 
| -XX:ConcGCThreads=\<ergo\> | The maximum number of threads used for concurrent work. By default, this value is -XX:ParallelGCThreads divided by 4. |  
| -XX:+G1UseAdaptiveIHOP   </br>  -XX:InitiatingHeapOccupancyPercent=45 | Defaults for controlling the initiating heap occupancy indicate that adaptive determination of that value is turned on, and that for the first few collection cycles G1 will use an occupancy of 45% of the old generation as mark start threshold. |  
| -XX:G1HeapRegionSize=\<ergo\> | The set of the heap region size based on initial and maximum heap size. So that heap contains roughly 2048 heap regions. The size of a heap region can vary from 1 to 32 MB, and must be a power of 2. |  
| -XX:G1NewSizePercent=5  </br>  -XX:G1MaxNewSizePercent=60 | The size of the young generation in total, which varies between these two values as percentages of the current Java heap in use. |
| -XX:G1HeapWastePercent=5 | The allowed unreclaimed space in the collection set candidates as a percentage. G1 stops the space-reclamation phase if the free space in the collection set candidates is lower than that. |
| -XX:G1MixedGCCountTarget=8 | The expected length of the space-reclamation phase in a number of collections. |  
| -XX:G1MixedGCLiveThresholdPercent=85 | Old generation regions with higher live object occupancy than this percentage aren't collected in this space-reclamation phase. |  

*** \<ergo\> 값은 환경에 따라 ergonomic하게 실제값이 결정됨 ***
