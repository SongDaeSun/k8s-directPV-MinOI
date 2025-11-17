# 1. DirectPV 설치
## 1-1. krew 설치
```
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
```
  
Terminal 추가
```
echo 'export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```
  
버전 확인
```
kubectl krew version
```
# 2. 디스크 자동 감지 및 자동 클레임
## 2-1. 드라이브 발견
```
kubectl directpv discover --all
```
**drives.yaml** 파일 생성.  
-> 사용할 파티션만 선택하여 수정할 것

## 2-2. directPV 등록
```
kubectl directpv init drives.yaml --dangerous
```

## 2-*. directPV 확인 및 수정
드라이브 확인
```
kubectl directpv  list drives
```
드라이브 삭제
```
kubectl directpv remove --drives /dev/sdb2 --nodes worker2
kubectl directpv remove --drives worker2-sdb2
```


# 3. MinIO Operator 설치
Helm으로 설치한다.
```
# Helm 저장소 추가
helm repo add minio-operator https://operator.min.io

# 저장소 업데이트
helm repo update

# MinIO Operator 설치
helm install \
  --namespace minio-operator \
  --create-namespace \
  operator minio-operator/operator
```

설치 확인:
```
kubectl get pods -n minio-operator
```

# 4. DirectPV를 사용하는 MinIO Tenant 생성
Tenant CRD를 직접 생성하는 것이 Helm을 통한 생성보다 더 낫다.

## 4-1. namespace 생성
```
kubectl create namespace minio-tenant
```

## 4-2. Secret 생성
MINIO_ROOT_USER, MINIO_ROOT_PASSWORD은 실재 사용환경에서는 변경하도록 하자.  
yaml로 생성하는 것을 추천한다.  
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: minio-creds-secret
  namespace: minio-tenant
type: Opaque
stringData:
  config.env: |
    export MINIO_ROOT_USER="admin"
    export MINIO_ROOT_PASSWORD="admin123456"
  # Console용 추가
  accesskey: "admin"
  secretkey: "admin123456"
```
kubectl apply -f minio-cred-secret.yaml


## 4-3. CRD생성
```
# latest 태그로 재생성
cat <<EOF | kubectl apply -f -
apiVersion: minio.min.io/v2
kind: Tenant
metadata:
  name: minio-cluster
  namespace: minio-tenant
spec:
  image: minio/minio:latest
  imagePullPolicy: IfNotPresent
  
  pools:
    - name: pool-0
      servers: 4
      volumesPerServer: 4
      
      volumeClaimTemplate:
        metadata:
          name: data
        spec:
          accessModes:
            - ReadWriteOnce
          storageClassName: directpv-min-io
          resources:
            requests:
              storage: 50Gi
      
      resources:
        requests:
          memory: 2Gi
          cpu: 250m
        limits:
          memory: 4Gi
          cpu: 1000m
      
      securityContext:
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        runAsNonRoot: true
      
      containerSecurityContext:
        runAsUser: 1000
        runAsGroup: 1000
        runAsNonRoot: true
        allowPrivilegeEscalation: false
        capabilities:
          drop:
            - ALL
        seccompProfile:
          type: RuntimeDefault
  
  mountPath: /export
  subPath: /data
  requestAutoCert: false
  
  configuration:
    name: minio-creds-secret
  
  features:
    bucketDNS: false
  
  podManagementPolicy: Parallel
EOF
```

## 4-4. 설치확인
```
# 1. Tenant 생성 확인
kubectl get tenant -n minio-tenant

# 2. Pod 상태 확인 (Running 되는지)
kubectl get pods -n minio-tenant -w

# 3. PVC확인
kubectl get pvc -n minio-tenant

# 4. DirectPV 볼륨 확인
kubectl directpv list volumes
```

# 5. 행벅! Object Storage를 사용하자!
```
$ kubectl get tenant -n minio-tenant
```

```
NAME            STATE         HEALTH   AGE
minio-cluster   Initialized   green    13m
```
위 결과가 나오면 성공이다.
