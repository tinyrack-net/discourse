# Discourse

이 프로젝트는 [타이니랙 포럼](https://forum.tinyrack.net) 에서 사용하는 Discourse 이미지를 빌드해
`ghcr.io/tinyrack-net/discourse` 로 푸시하는 저장소에요.

Discourse 는 다른 소프트웨어 대비 조금 특이한 컨테이너 배포 방식을 가지고 있어요. 공식적인 방법은
사전에 빌드된 이미지를 사용하지 않고 배포할 때 이미지를 동적으로 생성하는 방식이고, 이 과정에서
운영 중인 데이터베이스와 레디스에 접근해 마이그레이션까지 수행해요.

이 방식은 쿠버네티스 배포를 까다롭게 만들어요.

1. 표준화된 단일 컨테이너가 없어서 모두가 각자의 이미지를 빌드해 사용해야 해요.
2. 이미지 빌드 과정에서 운영 데이터베이스 접근이 필요하므로 CI/CD 프로세스를 구축하기 어려워요.
3. 이미지를 빌드할 때 데이터베이스 마이그레이션이 수행되므로, 이전 이미지로 운영 중인 포럼이 다운될 수 있어요.

## 운영 DB/Redis 무접속 빌드

이 저장소는 위 문제를 **빌드와 운영을 분리**하는 방식으로 해결해요. 업스트림 Launcher V2
(`discourse/launcher` v2.6.x) 의 `build` 단계는 문서 그대로 "DB 에 접속할 필요가 없고
postgres/redis 가 떠 있을 필요도 없으며", 코드상 `pups --skip-tags=precompile,migrate,db` 로 실행돼요.

그래서 빌드 단계에서는 다음까지만 수행해요.

- Discourse 소스를 `params.version` ref 로 체크아웃
- 플러그인 클론 (`hooks.after_code`)
- gem / pnpm 의존성 설치
- **Ember 빌드와 플러그인 JS 빌드** (`assets:precompile:build`, `SKIP_DB_AND_REDIS=1`)

마이그레이션과 나머지 프리컴파일은 이미지에 굽지 않고 파드가 뜰 때 수행해요. `containers/app.yml`
의 `MIGRATE_ON_BOOT=1`, `PRECOMPILE_ON_BOOT=1` 이 이미지 ENV 로 구워지고, `/etc/service/unicorn/run`
이 부팅 시 `db:migrate` → `assets:precompile`(`SKIP_EMBER_CLI_COMPILE=1`) → unicorn 순으로 실행해요.
공식 `discourse/discourse` 이미지와 동일한 모델이에요.

**자체 이미지가 계속 필요한 이유**: 플러그인 JS 는 Ember 빌드에 포함되어야 하는데, 부팅 시
프리컴파일 경로는 `SKIP_EMBER_CLI_COMPILE=1` 로 Ember 빌드를 건너뛰어요. 즉 공식 이미지 +
런타임 플러그인만으로는 플러그인 프론트엔드를 만들 수 없어요.

### 유지하는 플러그인

| 플러그인 | 비고 |
| --- | --- |
| `discourse-doc-categories` | |
| `discourse-bbcode` | |
| `discourse-translator` | |

`docker_manager` 는 k8s 에서 동작하지 않아 제거했어요. `discourse-category-headers` 는
`plugin.rb` 가 없는 테마 컴포넌트라 `plugins/` 로 클론해도 효과가 없어 제거했어요.
`discourse-prometheus` 는 클러스터에서 이 파드의 메트릭을 스크레이프하지 않아 제거했어요.

## 태깅 규약

| 태그 | 의미 | 불변성 |
| --- | --- | --- |
| `v<discourse-version>` (예: `v2026.9.0`) | 해당 Discourse 버전의 최초 빌드. **운영 매니페스트에 넣는 값** | 불변 |
| `v<discourse-version>.<n>` (예: `v2026.9.0.1`) | 같은 Discourse 버전을 플러그인/베이스 변경 후 재빌드 | 불변 |
| `stable` | 최신 stable 빌드를 가리키는 이동 태그 | 이동 |

운영 매니페스트에는 반드시 불변 태그(`v<version>` 또는 `v<version>.<n>`)를 사용해요. `stable` 은
편의용이라 언제든 바뀔 수 있어요. `latest` 와 빌드 번호 태그는 만들지 않아요.

이미지에는 확인용 OCI 라벨이 구워져요. `docker inspect` 만으로 이 이미지가 무엇인지 알 수 있어야 해요.

- `org.opencontainers.image.version` — Discourse 버전
- `org.opencontainers.image.revision` — `discourse_docker@<sha>`, `discourse@<sha>`
- `org.opencontainers.image.source`, `org.opencontainers.image.created`
- `org.tinyrack.discourse.base-image`
- `org.tinyrack.discourse.plugins`

## 빌드 워크플로

`.github/workflows/build.yml`

- **트리거**: 매일 03:00 KST 스케줄 1회 + 수동 실행(`force`, `version` 입력)
- **resolve**: `discourse/discourse` 태그 API 를 페이지네이션하며 `^v[0-9]{4}\.[0-9]+\.[0-9]+$` 에
  맞는 최고 버전을 고르고, GHCR 에 같은 태그가 있으면 건너뛰어요. `force` 면 `.<n>` 을 자동 증가시켜
  새로 빌드해요.
- **build**: `discourse_docker` 를 클론하고 Launcher V2 바이너리를 설치한 뒤
  `discourse/base:web-only-stable` 기반으로 빌드해요. 이때 이미지는 **러너 로컬에만** 있어요.
- **verify**: 푸시하기 전에 로컬 이미지로 5가지를 검사해요.
  1. 플러그인 3종 디렉터리/`plugin.rb` 존재, 제거한 플러그인 부재
  2. `frontend/discourse/dist/BUILD_INFO.json` 과 `dist/assets`, `app/assets/generated/<plugin>` 존재
  3. `nginx -t`
  4. `SKIP_DB_AND_REDIS=1 RAILS_ENV=production LOAD_PLUGINS=1 RAILS_DB=nonexistent bin/rails runner`
     (디스코드 자체 CI 와 동일한 게이트 — 플러그인이 부팅 시 DB 를 건드리면 여기서 실패)
  5. 이미지 `.Config.Env` 에 `DISCOURSE_DB_*`/`DISCOURSE_REDIS_*`/`DISCOURSE_SMTP_*`/`DISCOURSE_HOSTNAME`
     같은 런타임 시크릿이 없음
- **push**: 5가지를 모두 통과한 뒤에야 불변 태그를 푸시하고 `stable` 이동 태그를 같은 다이제스트로
  옮겨요. 이미 존재하는 불변 태그는 절대 덮어쓰지 않아요.

워크플로가 끝나면 Job Summary 에 붙여넣기용 `kubectl set image` 한 줄이 출력돼요.

> Launcher V2 는 positional config 인자 뒤의 플래그를 전부 `docker build` 로 그대로 전달해요.
> `--set`/`--tag` 는 반드시 config(`app`) **앞**에 와야 해요.

### 최초 1회 설정: GHCR 패키지 연결

`GITHUB_TOKEN` 은 저장소에 **연결(link)** 된 패키지만 접근할 수 있어요. `ghcr.io/tinyrack-net/discourse`
패키지가 저장소에 연결되어 있지 않으면 빌드는 성공하지만 푸시가 `denied` 로 실패해요.

1. GitHub → 조직 `tinyrack-net` → **Packages** → `discourse` → **Package settings**
2. **Manage Actions access** → **Add repository** → `tinyrack-net/discourse` 를 **Write** 역할로 추가

> GHCR 은 `push-by-digest` 를 지원하지 않아서 "다이제스트로 먼저 올리고 검증 후 태그" 하는 방식을 쓸 수
> 없어요. 그래서 검증은 러너 로컬 이미지로 수행하고, 통과한 뒤에만 푸시해요.

## 운영 반영 (수동)

인프라 저장소(`tinyrack-net/infrastructure`)의 배포 파일은 **수동으로** 갱신해요.

1. `apps/base/discourse/discourse.deployment.yaml` 의 `image` 를 Job Summary 의 불변 태그로 바꿔요.
2. `strategy: {type: Recreate}` 를 둬요. 단일 레플리카 + Longhorn RWO 에서 순서를 보장하기 위함이에요.
3. **`startupProbe` 가 필수예요.** 부팅 중 마이그레이션/프리컴파일 동안 기존 `livenessProbe`
   (15초 + 20초×3회)가 컨테이너를 죽여 크래시 루프에 빠져요.

   ```yaml
   startupProbe:
     httpGet:
       path: /
       port: 80
     periodSeconds: 10
     failureThreshold: 60   # 최대 10분
   ```

4. 부팅 플래그(`MIGRATE_ON_BOOT`, `PRECOMPILE_ON_BOOT`)는 이미지에 구워져 있으므로 Deployment env 는
   바꾸지 않아요. 런타임 `DISCOURSE_*` env/Secret 은 그대로 유지해요.

```bash
kubectl --context tinyrack -n discourse-system set image deployment/discourse \
  discourse=ghcr.io/tinyrack-net/discourse:v<X.Y.Z>
```

### 주의

- 부팅 시 마이그레이션이 실행되므로 배포에 1~3분 다운타임이 생겨요. 파드 재시작 시에도 프리컴파일을 다시 해요.
- 롤백 시 되돌릴 이전 태그를 커밋 메시지에 남겨요. **새 버전 마이그레이션이 이미 실행됐다면 롤백이
  불가할 수 있어요.**

## 참고 자료

- https://meta.discourse.org/t/can-discourse-ship-frequent-docker-images-that-do-not-need-to-be-bootstrapped/33205
- https://meta.discourse.org/t/installing-on-kubernetes/49329
- https://github.com/discourse/discourse_docker
- https://github.com/discourse/launcher
