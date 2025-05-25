#k8s  #Pod
#Container #Volume #Network

pod는 kubernetes에서 사용되는 가장 작은 배포가능한 단위이다
pod는 storage, network 자원을 공유하는 1개 이상의 container들의 집합이다

pod는 2가지 방식으로 주로 사용된다.
1. 단일 Container을 실행 - 'one-container-per-Pod' model은 single container을 wrapping한 pod로 보면 되고, k8s에서는 container을 직접 관리하기보다는 Pods을 관리하는 거로 보면된다.
2. 다중 Container을 실행 - 이 모델은 강하게 결합되어있고, 리소스를 공유해야 하는 여러 컨테이너들로 구성된 application을 캡슐화 한다. 이 패턴은 컨테이너가 tightly coupled된 경우에 사용한다.
