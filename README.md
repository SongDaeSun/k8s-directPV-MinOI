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






