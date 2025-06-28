# Discourse

이 프로젝트는 [타이니랙 포럼](https://forum.tinyrack.net) 에서 사용하는 Discourse 의 이미지 빌드 프로젝트에요.

Discourse는 다른 소프트웨어 대비 조금 특이한 컨테이너 배포 방식을 가지고 있어요. 공식적인 방법은 사전에 빌드된 이미지를 사용하지 않고 배포할 때 이미지를 동적으로 생성하는 방식이에요. 그리고 이 과정에서 운영중인 데이터베이스와 레디스 서버에 접근해 마이그레이션을 수행해요.

이 방식은 쿠버네티스 배포를 까다롭게 만드는데 다음의 문제점이 생기게 돼요.

1. 표준화된 단일 컨테이너가 없어서 모두가 각자의 이미지를 빌드해 사용해야 해요.
2. 이미지 빌드 과정에서 운영 데이터베이스 접근이 필요하므로, CI/CD 프로세스를 구축하기 어려워요.
3. 이미지를 빌드할 때 데이터베이스 마이그레이션이 수행되므로, 이전 이미지로 운영중인 포럼이 다운될 수 있어요.

이 문제를 해결하기 위해 여러 커뮤니티 프로젝트가 나와 있긴 하지만, 대부분 개발이 오랫동안 지속되지 못하거나 Discourse 업스트림 변경 사항을 따라가지 못하는 상태에요.

그래서 저는 공식적인 이미지 빌드 프로세스를 따르되, 테일스케일을 활용해 빌드 과정에서도 운영 데이터베이스를 안전하게 접근하는 방식을 사용하고 있어요. 빌드 과정은 다음과 같아요.

1. 공식 빌더 프로젝트([Discourse Docker](https://github.com/discourse/discourse_docker))를 클론해요.
2. 빌드 설정 파일(`app.yml`)을 클론한 프로젝트에 복제해요.
3. 테일스케일을 통해 운영중인 쿠버네티스 클러스터에 접근해요.
4. 운영 데이터베이스와 레디스에 연결해 새로운 이미지를 빌드해요.
5. 이미지를 레지스트리에 푸시해요.
6. 쿠버네티스는 빌드된 이미지를 사용해 앱을 업데이트해요.

제 쿠버네티스 배포 설정은 [이곳](https://github.com/tinyrack-net/infrastructure)에서 확인할 수 있어요.

# 참고 자료

- https://hub.docker.com/r/bitnami/discourse
- https://meta.discourse.org/t/can-discourse-ship-frequent-docker-images-that-do-not-need-to-be-bootstrapped/33205
- https://meta.discourse.org/t/installing-on-kubernetes/49329
