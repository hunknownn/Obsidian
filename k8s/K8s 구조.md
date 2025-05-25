#k8s 
## Kubernetes Component
- k8s cluster은 Node + Control plane으로 구성
- k8s cluster은 컨테이너화된 어플리케이션을 실행하는 Node 라고 하는 워커 머신의 집합. 각 클러스터는 최소 1개의 워커 Node를 가진다.
- 워커 노드는 Pod을 호스트한다. 컨트롤 플레인은 Worker Node 와 클러스터 내 Pod을 관리한다. 일반적으로 Control Plane은 여러 PC에 걸쳐 실행된다.
![[components-of-kubernetes 1.svg]]
[쿠버네티스 클러스터 구성요소](https://kubernetes.io/images/docs/components-of-kubernetes.svg)

#### 워크로드
kubernetes에서 구동되는 application이다. 