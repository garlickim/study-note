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
- 논리적으로 Young Gen, Old Gen,.. 등으로 나눔

