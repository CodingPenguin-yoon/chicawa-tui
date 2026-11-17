# chicawa-tui

Kubernetes 리소스의 연결 관계와 장애 지점을 보여주는, 아기자기한 2D 도트 캐릭터 기반 TUI.

EKS에 애플리케이션을 올린 개발자가 현재 상태를 살펴보고, 접속이 막힌 지점을 찾고, 필요할 때 자신이 사용하는 Codex 또는 Claude Code로 조사를 이어갈 수 있게 만든다.

## 프로젝트 상태

현재 기획 단계다. 이 README는 제품 방향과 첫 구현 범위를 정리한 계획이며, 아래 화면과 기능은 구현 목표다. 제품명과 캐릭터 디자인은 가칭이다.

## 해결하려는 문제

배포 이후 다음과 같은 질문에 답하는 데 집중한다.

- Pod는 실행 중인데 왜 서비스에 접속할 수 없을까?
- Ingress가 가리키는 Service가 실제로 존재할까?
- Service가 어떤 Pod를 선택하고 있으며, 요청을 받을 준비가 된 대상은 몇 개일까?
- 현재 설정에서 연결이 끊긴 지점은 어디일까?
- 그 판단의 근거가 되는 설정, 이벤트, 로그는 무엇일까?

주요 사용자는 EKS에 애플리케이션을 배포하고 일상적인 운영과 장애 확인을 수행하는 개발자다. 첫 사용 장면은 **배포는 완료됐는데 접속이 안 되는 상황**으로 잡는다.

## 핵심 사용 흐름

1. Homebrew로 설치하고 기존 kubeconfig를 사용해 실행한다.
2. 클러스터와 네임스페이스를 선택한다.
3. 애플리케이션의 Ingress, Service 또는 워크로드를 선택한다.
4. 연결 관계와 각 리소스의 상태를 한 화면에서 확인한다.
5. 문제가 표시된 지점에서 참조 설정, 이벤트, 로그를 연다.
6. 추가 조사가 필요하면 `?`를 눌러 Codex 또는 Claude Code를 선택한다.
7. 공식 CLI에서 조사하고 종료하면 기존 TUI 화면으로 돌아와 상태를 갱신한다.

## 운영 화면

주 화면은 애플리케이션으로 들어오는 경로와 관련 리소스를 중심으로 구성한다. 각 Pod의 노드 배치는 상세 정보로 제공한다.

아래는 Ingress가 참조하는 Service가 없는 상황의 화면 예시다.

```text
eks-prod / shop                         api.example.com/payments

  ✓ Ingress payments
      └─ /payments → Service payments:80
  ✕ Service payments                  shop 네임스페이스에 없음

  같은 네임스페이스의 워크로드
  ✓ Deployment payments               2/2 Pods Ready
      ├─ payments-7f9-x2k             node-a
      └─ payments-7f9-q8m             node-b

  Ingress가 참조하는 Service를 찾을 수 없어요.
  참조 대상  shop / Service payments / port 80

  [Enter] 참조 확인  [e] 이벤트  [l] 로그  [?] 에이전트로 조사
```

화면에 함께 보이는 워크로드는 참조, selector, ownerReferences 등 확인 가능한 관계를 근거로 연결한다. Service가 없는 예시에서는 같은 네임스페이스의 Deployment를 참고 정보로 표시한다.

Service가 존재하는 경우에는 참조 포트, selector에 대응하는 Pod, EndpointSlice의 대상과 준비 상태까지 이어서 확인한다. Kubernetes 객체의 연결 상태와 관측한 근거를 함께 보여준다. [Kubernetes Service 디버깅](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/) · [EndpointSlice 문서](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)

## 첫 진단 범위

| 확인할 상황 | 판단 근거 | 화면에서 제공할 정보 |
| --- | --- | --- |
| Ingress가 참조하는 Service 없음 | Ingress의 backend 참조와 같은 네임스페이스의 Service 조회 결과 | 누락된 Service 이름과 참조 위치 |
| Ingress가 참조하는 Service 포트 없음 | backend의 포트 이름 또는 번호와 Service의 ports | 요청한 포트와 실제 제공하는 포트 |
| Service selector에 맞는 Pod 없음 | Service selector와 같은 네임스페이스의 Pod labels | selector와 현재 일치하는 Pod 수 |
| 요청을 받을 준비가 된 대상 0개 | EndpointSlice의 대상 및 조건과 관련 Pod 상태 | 대상 수, 준비 상태, 관련 이벤트 |
| 연결된 Pod의 실행 문제 | 컨테이너 상태, 종료 사유, 재시작 횟수, 이벤트와 로그 | CrashLoopBackOff, OOMKilled, Pending 등의 실제 상태와 근거 |

Service 누락은 명시적인 참조를 확인했을 때 표시한다. 조회 권한 부족, 연결 실패, 수집 중인 상태는 화면에 각각 표시하고, 진단에는 마지막 확인 시각을 제공한다. selector가 없는 Service와 외부 대상은 해당 구성에 맞는 정보와 진단 지원 범위를 표시한다.

Ingress의 Service backend는 동일 네임스페이스의 Service를 참조한다. 이 규칙을 첫 누락 검사에 사용한다. [Ingress API 문서](https://kubernetes.io/docs/reference/kubernetes-api/networking/ingress-v1/#IngressServiceBackend)

## 도트 캐릭터와 화면 디자인

- 작은 2D 도트 캐릭터가 워크로드 상태에 반응한다.
- 정상 상태에서는 쉬고, 롤아웃 중에는 상자를 나르고, 문제가 생기면 구조 깃발을 드는 식으로 표현한다.
- 문제가 생긴 리소스나 연결 가까이에 상태를 표시한다.
- 캐릭터 옆에는 `2/3 Ready`, `CrashLoopBackOff` 같은 실제 Kubernetes 상태를 함께 둔다.
- 이름, 상태, 연결 관계, 로그를 빠르게 읽을 수 있는 화면 밀도를 유지한다.
- 키보드로 목록 탐색, 상세 확인, 로그 열기, 에이전트 실행을 이어갈 수 있게 한다.

캐릭터의 세부 디자인과 애니메이션은 첫 운영 화면을 구현하면서 구체화한다.

## Codex와 Claude Code 연동

사용자가 설치한 공식 Codex CLI와 Claude Code를 대화형으로 실행한다. 사용자는 각 CLI의 공식 로그인 흐름에서 자신의 계정으로 로그인하고, 해당 계정의 구독과 사용 한도에 따라 이용한다.

TUI에서 에이전트를 선택하면 현재 클러스터 context, 네임스페이스, 선택한 리소스와 확인된 문제를 초기 조사 맥락으로 전달한다. CLI가 터미널을 사용하는 동안 사용자는 에이전트와 직접 대화한다. 종료 후에는 선택했던 리소스 화면으로 복귀한다.

공통 Kubernetes MCP 서버를 통해 두 CLI에 리소스 조회, 이벤트 확인, 로그 조회 도구를 제공할 계획이다. MVP의 TUI와 제공 MCP 도구는 조회 기능부터 구현한다. 실제 접근 가능 범위는 사용자의 Kubernetes 인증과 RBAC 권한을 따른다.

```text
chicawa-tui
  ├─ 리소스 연결과 상태 표시
  ├─ 설정과 상태에 기반한 진단
  └─ 선택한 공식 CLI 실행
       ├─ Codex CLI ───┐
       └─ Claude Code ─┴─ 공통 Kubernetes MCP ─ 선택한 클러스터 조회
```

Claude Code의 제품 내 실행에는 Anthropic의 원본 바이너리, 인증 방식, 사용자별 과금 및 배포 조건을 적용한다. 각 CLI의 구독 이용 범위와 연결 설정은 공식 문서를 기준으로 구현한다. [Codex 인증](https://learn.chatgpt.com/docs/auth) · [Codex MCP](https://learn.chatgpt.com/docs/extend/mcp?surface=cli) · [Claude Code 배포 조건](https://code.claude.com/docs/en/legal-and-compliance#can-customers-offer-claude-code-in-their-products) · [Claude Code CLI](https://code.claude.com/docs/en/cli-reference)

## EKS 확장

첫 버전은 Kubernetes API에서 확인할 수 있는 리소스와 연결 관계에 집중한다. 이후 AWS API 연동을 통해 ALB와 타깃 그룹의 헬스 체크 상태까지 조사 범위를 넓힌다.

EKS의 ALB 연결 화면은 클러스터가 사용하는 컨트롤러와 target type에 맞춰 구성한다. `ip` 모드는 ALB가 Pod IP로 직접 전달하고, `instance` 모드는 노드의 NodePort를 거친다. 화면의 트래픽 경로는 실제 설정을 기준으로 표시한다. [Amazon EKS 로드밸런싱](https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html)

Kubernetes API만으로 확인한 연결 관계에는 그 관측 범위를 표시한다. AWS 조회나 실제 요청 확인이 필요한 항목은 추가 확인 상태로 보여준다.

## 구현 순서

- [ ] 기존 kubeconfig를 읽고 context와 네임스페이스 선택
- [ ] Ingress, Service, Deployment, Pod, EndpointSlice 조회
- [ ] 연결 관계와 Pod 노드 배치 상세 화면
- [ ] 누락된 참조, 포트, selector와 대상 준비 상태 검사
- [ ] 이벤트, 현재 로그와 이전 컨테이너 로그 조회
- [ ] 도트 캐릭터의 기본 상태 표현
- [ ] 공식 Codex CLI와 Claude Code 실행 및 TUI 복귀
- [ ] 두 CLI에서 사용하는 공통 Kubernetes MCP 조회 도구
- [ ] Homebrew 자체 tap을 통한 설치와 릴리스 배포

후속 확장으로 ALB 타깃 상태 진단과 사용자가 확인한 운영 조작을 검토한다. 조작 기능을 추가할 때는 대상 context, 네임스페이스, 리소스와 변경 내용을 확인할 수 있게 한다.

## 첫 데모의 완료 기준

Ingress가 존재하고 해당 이름의 Service가 누락된 데모 클러스터에서 다음 흐름을 확인한다.

1. 애플리케이션을 선택하면 Ingress의 참조 대상과 누락된 Service가 표시된다.
2. 연결이 끊긴 위치와 근거를 같은 화면에서 확인할 수 있다.
3. `?`로 Codex 또는 Claude Code를 열면 같은 리소스를 조사할 수 있다.
4. CLI를 종료하면 기존 화면으로 돌아와 최신 상태를 확인할 수 있다.
5. Service를 정상 구성한 뒤 새로고침하면 갱신된 연결 관계와 진단 결과를 확인할 수 있다.
