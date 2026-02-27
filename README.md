
# Kubernetes 실습 총정리 (깃허브용)

## 1️⃣ Kubernetes 핵심 개념

| 개념             | 설명                          | 공식문서 링크                                                                                 | 예시 YAML                                                                                                                                                                                                                                         |
| -------------- | --------------------------- | --------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Pod**        | 컨테이너 최소 실행 단위               | [Pods](https://kubernetes.io/ko/docs/concepts/workloads/pods/)                          | `yaml apiVersion: v1 kind: Pod metadata: name: my-pod spec: containers: - name: my-container image: nginx:1.28 ports: - containerPort: 80 `                                                                                                     |
| **Deployment** | Pod 복제/배포/업데이트 관리           | [Deployments](https://kubernetes.io/ko/docs/concepts/workloads/controllers/deployment/) | `yaml apiVersion: apps/v1 kind: Deployment metadata: name: app1-deployment spec: replicas: 1 selector: matchLabels: app: app1 template: metadata: labels: app: app1 spec: containers: - name: app1-container image: app:1 containerPort: 8000 ` |
| **Service**    | Pod를 클러스터 내부/외부에서 접근 가능하게 함 | [Services](https://kubernetes.io/ko/docs/concepts/services-networking/service/)         | `yaml apiVersion: v1 kind: Service metadata: name: app1-service spec: type: LoadBalancer ports: - port: 80 targetPort: 8000 selector: app: app1 `                                                                                               |
| **ConfigMap**  | 환경 변수, 설정 정보 저장             | [ConfigMap](https://kubernetes.io/ko/docs/concepts/configuration/configmap/)            | `yaml apiVersion: v1 kind: ConfigMap metadata: name: app1-config data: CLIENT_ID: "abc" REDIRECT_URI: "http://example.com/callback" `                                                                                                           |
| **Secret**     | 민감 정보 저장                    | [Secrets](https://kubernetes.io/ko/docs/concepts/configuration/secret/)                 | `yaml apiVersion: v1 kind: Secret metadata: name: app1-secret type: Opaque data: PASSWORD: cGFzc3dvcmQ= `                                                                                                                                       |

---

## 2️⃣ 볼륨 및 스토리지

| 종류                             | 설명                           | 공식문서 링크                                                                                                            | 예시                                                                                                                                                                                                                                                                                                                                                           |
| ------------------------------ | ---------------------------- | ------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **hostPath**                   | 노드 로컬 디렉토리 마운트, 테스트용         | [hostPath](https://kubernetes.io/ko/docs/concepts/storage/volumes/#hostpath)                                       | `yaml volumes: - name: local-vol hostPath: path: /tmp/data type: DirectoryOrCreate `                                                                                                                                                                                                                                                                         |
| **emptyDir**                   | Pod 시작 시 빈 디렉토리, Pod 종료 시 삭제 | [emptyDir](https://kubernetes.io/ko/docs/concepts/storage/volumes/#emptydir)                                       | `yaml volumes: - name: tmp-vol emptyDir: {} `                                                                                                                                                                                                                                                                                                                |
| **PersistentVolume(PV)**       | 실제 스토리지 자원                   | [PersistentVolume](https://kubernetes.io/ko/docs/concepts/storage/persistent-volumes/)                             | `yaml apiVersion: v1 kind: PersistentVolume metadata: name: local-pv spec: capacity: storage: 5Gi accessModes: [ReadWriteOnce] storageClassName: local-storage persistentVolumeReclaimPolicy: Retain local: path: /data nodeAffinity: required: nodeSelectorTerms: - matchExpressions: - key: kubernetes.io/hostname operator: In values: [docker-desktop] ` |
| **PersistentVolumeClaim(PVC)** | Pod가 PV를 요청                  | [PersistentVolumeClaim](https://kubernetes.io/ko/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) | `yaml apiVersion: v1 kind: PersistentVolumeClaim metadata: name: local-pvc spec: accessModes: [ReadWriteOnce] resources: requests: storage: 1Gi storageClassName: local-storage `                                                                                                                                                                            |

### PV/PVC 실습 팁

* PV: 아직 사용 전 → USB 같은 저장공간
* PVC: Pod와 PV 연결
* `ReadWriteOnce`: 한 노드에서 읽기/쓰기 가능
* `storageClassName`: 어떤 스토리지 유형 사용할지
* `persistentVolumeReclaimPolicy`: `Retain` / `Delete` / `Recycle`
* 로컬 PV 사용 시 **Node Affinity** 필요

---

## 3️⃣ InitContainer + Build 환경

* Git clone → 빌드 → Nginx 컨테이너 배포
* Pod 간 볼륨 공유 가능

```yaml
initContainers:
  - name: git-clone
    image: alpine/git
    command: ["git", "clone", "repo-url", "/app"]
    volumeMounts:
      - name: react-vol
        mountPath: /app
containers:
  - name: nginx-container
    image: nginx:1.28
    ports:
      - containerPort: 80
    volumeMounts:
      - mountPath: /usr/share/nginx/html
        name: react-vol
volumes:
  - name: react-vol
    emptyDir: {}
```

---

## 4️⃣ MariaDB 실습

* ConfigMap으로 환경 변수 관리
* PVC로 데이터 영속화
* ConfigMap으로 my.cnf 설정

```yaml
volumeMounts:
  - name: mariadb-vol
    mountPath: /var/lib/mysql
  - name: mariadb-config-volume
    mountPath: /etc/mysql/conf.d/my.cnf
    subPath: my.cnf
volumes:
  - name: mariadb-vol
    persistentVolumeClaim:
      claimName: mariadb-pvc
  - name: mariadb-config-volume
    configMap:
      name: mariadb-my-config
```

---

## 5️⃣ OAuth 실습 (FastAPI)

* ConfigMap으로 Kakao Client ID, Secret, Redirect URI 관리
* Redis로 access_token / refresh_token 관리
* FastAPI + httpx + OAuth 구현

```python
@app.get("/login/kakao")
async def kakaoLogin(request: Request):
    return await oauth.kakao.authorize_redirect(request, settings.redirect_uri)
```

**Tip:**

* root_path, 쿠키 도메인, secure 설정 확인 필수
* 포트 충돌 주의
* `500 Internal Server Error` → 세션/쿠키/redis 설정 확인

---

## 6️⃣ 실습 팁

1. Pod/Service 이름 중복 주의

```bash
kubectl delete pod <name>
kubectl delete service <name>
```

2. YAML 문법 주의

   * 소문자 `containers`, `initContainers`
   * 들여쓰기, `volumeMounts` 확인
3. 볼륨 선택

   * 테스트: hostPath / emptyDir
   * 배포: PV + PVC
4. ConfigMap 변경 → Pod 재시작 필요
5. Node Affinity

   * 로컬 PV 사용 시 필수
6. Service 타입

   * ClusterIP: 내부용
   * LoadBalancer: 외부 접속용 (Docker Desktop: localhost로 접근 가능)

---

## 7️⃣ 공식문서 링크 정리

| 주제                    | 공식문서                                                                                                                                                                                   |
| --------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Pods                  | [https://kubernetes.io/ko/docs/concepts/workloads/pods/](https://kubernetes.io/ko/docs/concepts/workloads/pods/)                                                                       |
| Deployments           | [https://kubernetes.io/ko/docs/concepts/workloads/controllers/deployment/](https://kubernetes.io/ko/docs/concepts/workloads/controllers/deployment/)                                   |
| Services              | [https://kubernetes.io/ko/docs/concepts/services-networking/service/](https://kubernetes.io/ko/docs/concepts/services-networking/service/)                                             |
| ConfigMap             | [https://kubernetes.io/ko/docs/concepts/configuration/configmap/](https://kubernetes.io/ko/docs/concepts/configuration/configmap/)                                                     |
| Secrets               | [https://kubernetes.io/ko/docs/concepts/configuration/secret/](https://kubernetes.io/ko/docs/concepts/configuration/secret/)                                                           |
| Volumes               | [https://kubernetes.io/ko/docs/concepts/storage/volumes/](https://kubernetes.io/ko/docs/concepts/storage/volumes/)                                                                     |
| PersistentVolume      | [https://kubernetes.io/ko/docs/concepts/storage/persistent-volumes/](https://kubernetes.io/ko/docs/concepts/storage/persistent-volumes/)                                               |
| PersistentVolumeClaim | [https://kubernetes.io/ko/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims](https://kubernetes.io/ko/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) |

---




