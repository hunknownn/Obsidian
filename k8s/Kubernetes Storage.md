#k8s #pv #pvc

## Ephemeral
임시 볼륨, Pod가 종료되면 사라지는 볼륨
=> 파일 업로드/다운로드 기능에 주로 사용 (임시 파일)
=> 임시 볼륨 생성 후 컨테이너에 마운트해서 사용

## Persistence
영구 볼륨, Pod의 라이프사이클과 관계 없이 유지되는 볼륨
=> 여러 Pod 들이 서로 공유해야 하는 저장공간 == db, keycloak?

1개의 pod에 1개의 pv을 연결할 떄에는, statefulSet을 활용하는 구성.