# k8s-argocd

K3s 클러스터의 로그/메트릭 수집 인프라를 Helm 우산(umbrella) 차트 하나로 정의하고, ArgoCD가 이 저장소를 바라보며 GitOps 방식으로 동기화하는 저장소입니다.

## 로그 파이프라인 아키텍처

```
[전체 노드의 컨테이너 로그]
        │
   Filebeat (DaemonSet, ECK Beat CRD)
        │  k8s 메타데이터 부착 → Kafka 발행 (gzip 압축)
        ▼
   Kafka (Strimzi, KRaft 모드) ── topic: k8s-logs (파티션 3, 보존 1시간)
        │
   Logstash (Consumer, 3 threads)
        │  JSON 파싱 → SHA1 fingerprint 생성
        ▼
   Elasticsearch (ECK) ── 일자별 인덱스 k8s-logs-YYYY.MM.dd
        │
   Kibana (NodePort 30601)
```

### 설계 포인트

- **Kafka를 버퍼로 둔 이유**: Filebeat → ES 직결 구조는 ES 재시작/장애 시 로그가 유실되거나 수집기가 밀립니다. 중간에 Kafka를 두어 ES가 내려가도 로그를 1시간 동안 보존하고, 복구 후 Logstash가 이어서 소비합니다.
- **중복 적재 방지 (멱등 쓰기)**: Logstash가 `@timestamp + pod명 + message`로 SHA1 fingerprint를 만들어 ES `document_id`로 사용합니다. Kafka 재소비(rebalance, 재시작)가 발생해도 같은 로그는 같은 문서로 덮어써져 중복이 생기지 않습니다.
- **타임스탬프 정합성**: 앱이 JSON 로그를 남기면 본문을 파싱해 앱이 기록한 시각(`app_log.timestamp`)을 `@timestamp`로 사용합니다. 수집 시각이 아닌 발생 시각 기준으로 검색됩니다.

## 메트릭 스택

- **kube-prometheus-stack**: Prometheus + Grafana(NodePort) + 기본 대시보드
- **Thanos sidecar**: Prometheus 로컬 보존을 5분으로 짧게 잡고, 장기 데이터는 Thanos가 객체 스토리지로 내립니다. 단일 노드에서 Prometheus 메모리/디스크 부담을 줄이기 위한 구성입니다.
- **Kafka 모니터링**: kafka-exporter + ServiceMonitor로 컨슈머 lag 등 Kafka 지표를 Prometheus에 수집

## 보조 로그 경로 (Loki)

ELK와 별개로 fluent-bit → Loki 경로도 함께 구성되어 있습니다. Grafana에서 가볍게 조회하는 용도이며, 정밀 검색/집계는 ES, 빠른 tail 조회는 Loki로 용도를 나눕니다.

## 구성 요소

| 컴포넌트 | 배포 방식 | 역할 |
|---|---|---|
| Filebeat | ECK `Beat` CRD (DaemonSet) | 노드별 컨테이너 로그 수집 → Kafka 발행 |
| Kafka | Strimzi Operator (KRaft, ZooKeeper 없음) | 로그 버퍼링, topic `k8s-logs` |
| Logstash | Deployment + ConfigMap | 파싱, 중복 제거, ES 적재 |
| Elasticsearch | ECK Operator | 로그 저장/검색 (single-node, 4Gi) |
| Kibana | ECK Operator | 로그 조회 UI |
| Prometheus/Grafana | kube-prometheus-stack | 메트릭 수집/시각화 |
| Thanos | sidecar + query | 메트릭 장기 보관 오프로딩 |
| Loki/fluent-bit | loki-stack | 경량 로그 조회 경로 |

오퍼레이터(Strimzi, ECK)와 스택 차트는 모두 `Chart.yaml`의 `dependencies`로 선언되어 있어, 차트 하나로 전체 스택이 설치됩니다.

## 저장소 구조

```
my-elk/
├── Chart.yaml              # 우산 차트 + 의존성 6종 (Strimzi/ECK/kube-prometheus-stack/Thanos/Loki/fluent-bit)
├── values.yaml             # 전체 스택 설정 단일 진입점
└── templates/
    ├── filebeat/           # Beat CRD + RBAC
    ├── kafka/              # Kafka 클러스터, NodePool, Topic, exporter, ServiceMonitor
    ├── logstash/           # Deployment + pipeline ConfigMap
    ├── elasticsearch/      # Elasticsearch CRD
    ├── kibana/             # Kibana CRD
    └── thanos/             # 객체 스토리지 설정 Secret
```

## 배포

ArgoCD Application이 이 저장소의 `my-elk` 경로를 바라보고, main 브랜치 커밋을 감지해 자동 동기화합니다.

```bash
# 수동 설치 시
helm dependency build my-elk
helm install infra my-elk -n infra --create-namespace
```

> 단일 노드 K3s 기준 설정입니다 (replication factor 1, single-node ES). 멀티 노드 환경에서는 `values.yaml`의 복제 계수와 리소스를 조정해야 합니다.

## 관련 저장소

- [k8s-blog](https://github.com/youseonghyeon/k8s-blog) - 같은 클러스터에서 운영 중인 서비스 Helm 차트
