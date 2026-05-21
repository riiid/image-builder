# Riiid image-builder Fork — 변경 이력 및 메커니즘

이 fork 는 [`kubernetes-sigs/image-builder`](https://github.com/kubernetes-sigs/image-builder) 의 fork 로,
Riiid office IDC 환경 (Proxmox + Ubuntu 22.04) 에서 안정적 빌드를 위한 4개 fix 를 포함합니다.

## 왜 fork 인가

OSS `kubernetes-sigs/image-builder` 를 직접 clone 해서 사용하면 매 K8s minor 업그레이드마다
같은 fix 를 working tree 에 수동 적용해야 합니다. 이 학습 비용 (회당 30분~수 시간 디버깅)
누적은 비효율이므로 fork 후 fix 들을 commit 으로 보존합니다.

자세한 ROI 분석 및 OSS upstream 에 PR 보낼 수 없는 이유는 본 문서 마지막 [학습 인사이트](#학습-인사이트) 섹션 참조.

---

## 빌드 전체 흐름 — 4개 fix 가 어디서 작동하는가

```
1. VM 생성 ────────────────────── Fix #1 (qemu_agent: true)
2. Ubuntu autoinstall (subiquity)
3. cloud-init wait + apt update ── Fix #2 (policy-rc.d + needrestart 차단)
4. ansible firstboot.yml ───────── (Fix #2 효과)
5. reboot ──────────────────────── Fix #3 (nohup sleep 5)
6. SSH 재연결 대기 (sshd ready) ── Fix #4 (pause_before 90s)
                                   (Fix #1 효과 — IP 재조회)
7. ansible node.yml ───────────── (Fix #2 효과)
8. goss 검증 + shutdown + template 변환
```

**각 fix 가 다른 단계를 보호. 하나라도 빠지면 그 단계에서 빌드 실패.**

---

## Fix #1 — `qemu_agent: true`

**파일**: `images/capi/packer/proxmox/packer.json.tmpl` (builder block, line 52)

```diff
  "vm_id": "{{user `vmid`}}",
+ "qemu_agent": true
```

### 무엇이 일어나는가

Packer 가 Proxmox API 에 VM 생성 요청 시 `agent: 1` 옵션 전달.
Proxmox 가 VM 의 QMP(QEMU Machine Protocol) 소켓을 열어 qemu-guest-agent 데몬과 통신할 수 있게 만듦.

### 왜 필요한가 — 메커니즘

Packer 가 VM 의 IP 를 알아내는 방법은 2가지:

| 방법 | qemu_agent 설정 | 동작 |
|------|---------------|------|
| **ARP 기반** (default) | false | VM 생성 시 받은 초기 IP 만 캐싱. **재부팅 후 IP 바뀌면 못 따라감** |
| **Guest Agent 기반** | true | Proxmox API 의 `qm guest cmd <vmid> network-get-interfaces` 로 현재 IP 조회 |

**office IDC DHCP non-sticky lease** 환경에서:

1. VM 생성 → DHCP IP `A` 할당 → packer 가 `A` 캐싱
2. 재부팅 → DHCP 가 새 IP `B` 할당
3. packer 가 SSH 재연결 시도 → `A` 로 시도 → `no route to host` 실패

→ `qemu_agent: true` 면 packer 가 Proxmox API 통해 새 IP `B` 조회 가능.

### 작동 조건

- VM 안에 `qemu-guest-agent` 데몬 동작 중 (Ubuntu autoinstall user-data 의
  `packages` 에 명시되어 자동 설치됨)
- Proxmox API 응답 가능 (token 유효)

### OSS upstream 에서 안 들어가는 이유

기본값 `false` 가 보수적 선택. cloud(AWS/GCP) 의 sticky DHCP 환경에서는 불필요.
**office IDC 처럼 DHCP lease 가 짧고 MAC-based 가 아닌 환경** 에서만 필요. 환경 의존성이라 OSS default 안 됨.

---

## Fix #2 — policy-rc.d + needrestart 차단

**파일**: `images/capi/packer/proxmox/packer.json.tmpl` (provisioner 1 inline, line 78-81)

```diff
  "sudo apt-get -qq update",
+ "sudo sh -c \"echo '#!/bin/sh' > /usr/sbin/policy-rc.d && echo 'exit 101' >> /usr/sbin/policy-rc.d && chmod +x /usr/sbin/policy-rc.d\"",
+ "sudo mkdir -p /etc/needrestart/conf.d",
+ "sudo sh -c \"echo '\\$nrconf{restart} = \\\"l\\\";' > /etc/needrestart/conf.d/no-restart.conf\"",
  "echo done"
```

### 왜 두 개 다 필요한가 — Ubuntu service 재시작 경로 2가지

Ubuntu 22.04 의 apt 가 서비스를 자동 재시작하는 경로:

| 경로 | 트리거 | 차단 메커니즘 |
|------|------|-----------|
| **invoke-rc.d** | dpkg 가 패키지 설치 후 `invoke-rc.d <service> start` 호출 | `/usr/sbin/policy-rc.d` 가 exit code 101 반환하면 차단 |
| **needrestart** | apt 완료 후 hook 으로 자동 실행. 업데이트된 라이브러리 사용 중인 service 감지 → systemctl 직접 호출 | `policy-rc.d` 우회 → `$nrconf{restart} = "l"` 로 list-only 모드 강제 |

→ **두 경로 모두 차단해야 sshd 재시작 100% 방지**

### 빌드 도중 sshd 재시작이 왜 치명적인가

```
1. packer SSH connect (sshd 의 child process)
2. apt install libssl3 같은 패키지
   └─ dpkg post-install → invoke-rc.d 또는 needrestart 트리거
   └─ "sshd 재시작 필요" 판단 → systemctl restart sshd
3. sshd 재시작 시 모든 child process (packer SSH session) 종료
4. packer 의 ansible provisioner 가 SSH 끊김 감지 → 빌드 실패
```

### 변경 위치가 왜 provisioner 1 인가

provisioner 0 (cloud-init wait) 의 마지막 단계. **ansible firstboot.yml (provisioner 2) 가 실행되기 전**에
차단 정책 설정. 만약 ansible firstboot 도중 차단 정책 없으면 — kubelet/containerd 설치 도중
또는 dist-upgrade 시 sshd 재시작 발생.

### policy-rc.d 의 exit code 의미

```bash
#!/bin/sh
exit 101
```

- `exit 0`: 허용
- `exit 101`: 차단 (관행. dpkg-policy-rc-d-helper 가 이 코드 인식)
- 다른 값: undefined

### needrestart 의 `$nrconf{restart} = "l"` 의미

needrestart 설정은 Perl-style:

- `"l"` (list only): 어떤 서비스 재시작 필요한지 안내만 출력
- `"a"` (auto): 자동 재시작 (Ubuntu 22.04 기본 — **위험**)
- `"i"` (interactive): 사용자 입력 대기 (대화형 빌드 불가)

`"l"` 로 설정해서 안내 메시지만 출력, 실제 재시작 안 함.

### OSS upstream 에서 안 들어가는 이유

CI/CD 환경에서 ubuntu 빌드 시 보통 packer pause 후 ansible 가 빨리 끝나서 sshd 재시작
시간 안 됨. 또는 빌드 머신 spec 이 약해서 apt 가 느려 우연히 회피. **office 의 빠른 IDC 네트워크
+ 강력한 RTX3080 호스트** 라서 apt 가 빠르고 needrestart 가 시간 안에 발동.

---

## Fix #3 — `sudo reboot now` → `nohup bash -c 'sleep 5 && reboot' &`

**파일**: `images/capi/packer/proxmox/packer.json.tmpl` (provisioner 3, line 115)

```diff
- "sudo reboot now"
+ "nohup bash -c 'sleep 5 && reboot' &"
```

### 왜 즉시 reboot 이 위험한가 — Packer cleanup race condition

packer 의 provisioner 실행 flow:

1. SSH 로 명령 실행
2. 명령 종료 (exit code 수신)
3. **cleanup phase**: temp script 삭제 (`/tmp/packer-shell-XXX.sh`), stdout/stderr 마무리
4. 다음 provisioner 진행 또는 종료

`sudo reboot now` 의 동작:

- systemd 가 즉시 SIGTERM 모든 process
- SSH session 즉시 종료 (수 ms 이내)
- packer 가 cleanup 단계 (step 3) 시도 시 SSH 이미 끊겨있음
- **에러**: `Error removing temporary script at /tmp/script_XXX.sh: dial tcp ...: no route to host`

`expect_disconnect: true` 옵션은 "연결 끊김을 예상" 한다는 의미지만, cleanup 자체는 시도.
그 cleanup 시점에 이미 VM 이 down 상태면 실패.

### 변경 후 동작

```bash
nohup bash -c 'sleep 5 && reboot' &
```

각 토큰의 역할:

| 토큰 | 역할 |
|------|------|
| `nohup` | SIGHUP 무시. SSH 세션 종료해도 child process 살아남음 |
| `bash -c '...'` | subshell 에서 명령 실행 (한 명령으로 묶음) |
| `sleep 5` | 5초 대기 (packer cleanup 시간 확보) |
| `&&` | sleep 성공 후 reboot 실행 |
| `&` | background 실행 → 즉시 shell return |

흐름:

```
T+0s:     ssh 명령 실행 "nohup bash -c '...' &"
T+0.001s: subshell 이 background 로 fork
T+0.002s: ssh shell return → packer 가 명령 종료 인지
T+0.003s ~ T+5s: packer cleanup (temp script 삭제) — 여전히 SSH 연결 살아있음 ✅
T+5s:     백그라운드 subshell 의 sleep 완료 → reboot 실행
T+5.1s:   VM 종료 시작 → SSH 끊김 (이미 cleanup 끝남)
```

### 왜 5초가 적당한가

경험적 값. packer cleanup 의 일반 소요 시간:

- temp script 삭제: ~100ms
- stdout flush + connection close: ~500ms
- 합계 1초 이내가 보통

5초는 안전 마진. 10초까지 늘려도 무방하지만 빌드 시간 증가 (체감 가능). 1-2초는 race 가능.

### OSS upstream 에서 안 들어가는 이유

이건 우리만의 환경 의존성이 아니라 **일반적 race condition**.
OSS 에 PR 가치 있을 수 있음. 단, 일부 환경 (slow network, slow VM) 에서는
race 가 발생 안 해서 OSS maintainer 가 reproduce 불가 → PR 보류 또는 reject 가능성.

GitHub Issues 에 비슷한 보고가 [#1631](https://github.com/kubernetes-sigs/image-builder/issues/1631)
등으로 존재. 환경별로 다른 fix 제안이 있어 표준 fix 가 안 정해진 상태.

---

## Fix #4 — `pause_before: "10s"` → `"90s"`

**파일**: `images/capi/packer/proxmox/packer.json.tmpl` (provisioner 4, line 138)

```diff
- "pause_before": "10s",
+ "pause_before": "90s",
```

### 무엇이 일어나는가

ansible-playbook (node.yml) provisioner 시작 전 packer 가 강제 sleep.

### 왜 10초는 부족한가 — VM boot 시간 분석

직전 provisioner (reboot) 후 VM 의 boot 단계:

| 단계 | 시간 (Proxmox + Ubuntu 22.04) |
|------|----------------------------|
| 1. BIOS POST | ~2s |
| 2. GRUB → kernel load | ~3s |
| 3. initrd → rootfs mount | ~3s |
| 4. systemd 초기화 | ~5s |
| 5. networking (netplan + DHCP) | ~5s (DHCP lease 시간 변동) |
| 6. sshd 시작 | ~3s |
| 7. cloud-init 후처리 | ~5s |
| **합계** | **약 25-30s 일반적, 최악 시 40s** |

10s 후 packer 가 SSH connect 시도 → sshd 아직 시작 안 됨 → `connection refused`.
retry timeout 까지 실패 누적.

### 90초 의미

- 30s (보통 boot)
- + 30s (안 좋은 케이스 마진)
- + 30s (Proxmox VM I/O 슬로우 케이스 마진)
- = **90s 안전**

빌드 총 시간에 80초만 추가 (10s → 90s). 18분짜리 빌드에서 7% 증가. 안전성 대비 합리적.

### `pause_before` 동작 원리

packer 의 ansible provisioner 옵션. provisioner 시작 직전 `time.Sleep(duration)` 실행.
ssh retry/connection 시도 전에 무조건 sleep.

**대안**: `ssh_timeout: 2h` 같은 옵션으로 retry timeout 만 늘리는 방법도 있음.
단, retry 가 누적되면 connection log 가 지저분해지고 sshd 가 retry 시 brute-force 의심해서 차단할 수도.
**무조건 sleep 이 단순/안전**.

### OSS upstream 에서 안 들어가는 이유

10s 가 OSS default 인 이유:

- AWS/GCP 의 ephemeral VM 은 boot 가 매우 빠름 (~5s)
- 클라우드 baseimage 가 cloud-init 최적화됨
- maintainer 들이 주로 cloud 환경에서 테스트

Proxmox + Ubuntu 22.04 desktop ISO 기반 빌드는 ISO 부팅 + autoinstall 거쳐서 일반 cloud-init 보다 느림.
환경 특수성.

---

## 검증 결과

| 버전 | 날짜 | 빌드 시간 | 결과 | 비고 |
|------|------|---------|------|------|
| v1.32.13 | 2026-05-15 | 17분 6초 (실패) → fix 후 성공 | ✅ | 6시간 디버깅 후 4개 fix 도출 |
| v1.34.8 | 2026-05-20 | 18분 20초 | ✅ | fix 재적용 즉시 빌드 성공 |

각 fix 가 실제로 도움이 되는지 검증:

- **Fix #1 미적용** → SSH `no route to host` (재부팅 후 IP 변경 시)
- **Fix #2 미적용** → SSH connection reset (apt 도중 sshd 재시작 시)
- **Fix #3 미적용** → `Error removing temporary script` (cleanup 단계)
- **Fix #4 미적용** → `dial tcp ... connection refused` (reboot 후 너무 빨리 connect)

각각 1.32.13 빌드(2026-05-15) 에서 실제 경험한 에러.
1.34.8 빌드(2026-05-20) 는 4개 fix 모두 적용 후 18분 만에 성공.

---

## 학습 인사이트

### 4개 fix 의 카테고리 분류

| Fix | 환경 의존성 | OSS 보편성 |
|-----|----------|---------|
| #1 qemu_agent | office DHCP non-sticky | 환경별 |
| #2 needrestart 차단 | Ubuntu 22.04+ 의 빠른 apt | 환경별 |
| #3 nohup sleep | 빠른 SSH 환경의 race | **보편적** (PR 가치 있음) |
| #4 pause_before 90s | Proxmox + ISO 기반 VM 부팅 | Proxmox-specific |

### OSS fork 의 가치 — 3가지 패턴

1. **순수 환경 특수 (#1, #2, #4)** — OSS upstream PR 보내도 reject 가능. fork 에 영구 유지.
2. **보편적이지만 OSS reproduce 어려움 (#3)** — PR 시도 가치 있음. 단, upstream maintainer 환경 미재현 시 보류 가능.
3. **upstream 의 보수적 default (#4 의 10s)** — "정상 환경에선 충분" 한 값. 환경별 override 가 사용자 몫.

→ **fork + 분기별 upstream sync** 가 정공. 우리 fix 보존하면서 upstream 의 새 기능
(예: 새 Ubuntu 버전 지원, 새 provider 옵션) 통합 가능.

### "OSS 공식 권장 ≠ 우리 환경 정공"

image-builder 공식 문서는 **DHCP 사용 권장** ("172.x.x.x 정적 IP 회피"). 그러나 우리 office IDC 는:

- DHCP non-sticky lease (재부팅 시 새 IP)
- 172.16.10.0/16 대역 사용

→ 공식 권장이 우리 환경과 정확히 충돌. fork 후 환경 맞춤이 필수.

---

## Upstream sync 방법

분기별 또는 K8s major release 시 OSS 변경 사항 가져오기:

```bash
cd /path/to/image-builder

# upstream remote 설정 (최초 1회)
git remote add upstream https://github.com/kubernetes-sigs/image-builder.git

# upstream 최신 fetch
git fetch upstream

# main 으로 merge
git checkout main
git merge upstream/main

# conflict 발생 시 (보통 packer.json.tmpl)
# 우리 fix (qemu_agent, needrestart 차단, nohup sleep, pause_before 90s) 보존
# upstream 새 변경 통합

git push origin main
```

**Conflict 가능성 높은 파일**: `images/capi/packer/proxmox/packer.json.tmpl` (upstream 이 자주 수정)

---

## 사용법

### 1.35 등 다음 빌드 시

```bash
# mgmt 노드에서 fork clone
cd /root
git clone https://github.com/riiid/image-builder.git
cd image-builder/images/capi

# 의존성 설치
apt install -y python3.11-venv unzip build-essential jq
python3 -m venv /root/image-builder-venv
source /root/image-builder-venv/bin/activate && pip install ansible
make clean && make deps-proxmox

# 환경변수 설정 (cluster 별 다름)
export PROXMOX_URL="https://<host-ip>:8006/api2/json"
export PROXMOX_USERNAME='capmox@pve!capi'
export PROXMOX_TOKEN="$(awk '/capmox@pve!capi/{print $2}' /etc/pve/priv/token.cfg)"
export PROXMOX_NODE="<node-name>"
export PROXMOX_ISO_POOL="local"
export PROXMOX_BRIDGE="vmbr0"
export PROXMOX_STORAGE_POOL="nfs"   # 또는 local-lvm

# kubernetes.json 만 새 버전으로 수정
vim packer/config/kubernetes.json
# kubernetes_semver: "v1.35.X"
# kubernetes_series: "v1.35"
# kubernetes_deb_version: "1.35.X-1.1"
# kubernetes_rpm_version: "1.35.X"

# 빌드 실행 — fix 4개가 이미 적용된 상태라 즉시 가능
PACKER_FLAGS="--var 'vmid=500'" make build-proxmox-ubuntu-2204
```

### 후속 단계 (반드시 수행 — cascade #6 차단)

빌드 후 base template 을 NFS 에서 local-lvm 으로 이동:

```bash
# 1. NFS → local-lvm 이동
qm move-disk <VMID> scsi0 local-lvm --delete 1

# 2. NFS 잔여 정리 (chattr 실패로 파일 남음)
rm -rf /mnt/pve/nfs/images/<VMID>/

# 3. efidisk0 잔여 entry 제거 (SeaBIOS 라 efidisk 불필요)
qm set <VMID> --delete efidisk0

# 4. cpu max + template 확인
qm set <VMID> --cpu max
qm config <VMID> | grep -E '^(scsi0|template|cpu|bios)'

# 5. node02 분배 (vzdump + qmrestore — qm restore 아님)
vzdump <VMID> --compress zstd --storage local --mode stop
scp /var/lib/vz/dump/vzdump-qemu-<VMID>-*.vma.zst root@<node02-ip>:/var/lib/vz/dump/
ssh root@<node02-ip> "qmrestore /var/lib/vz/dump/vzdump-qemu-<VMID>-*.vma.zst <node02-VMID> --storage local-lvm --unique 1"
ssh root@<node02-ip> "qm set <node02-VMID> --template 1 --cpu max"
```

---

## 관련 문서 (riiid/k8s-on-premise)

- `devops-wiki/04-issues/image-builder.md` — fix 5종 + 정확한 파일 경로 + cascade #6 RCA
- `devops-wiki/02-context/image-builder.md` — VMID 매핑 + storage pool 결정 + 후속 4단계
- `devops-wiki/04-postmortems/2026-05-19-office-v1.33.12-upgrade.md` — 8가지 cascade 함정
- `devops-wiki/04-postmortems/2026-05-20-office-v1.34.8-upgrade.md` — 11가지 함정 + 정공 해결책

---

## TODO — 향후 개선 영역

### user-data network 섹션 static IP

현재 packer 빌드가 user-data 의 passwd 를 동적 generate 하면서 우리가 추가한 network 섹션을 reset.
다음 메커니즘 검토:

- A. packer.json.tmpl 의 http_directory 변경 (별도 user-data 경로)
- B. boot_command 에 `ip=` kernel parameter 직접 전달
- C. preseed/cloud-init 의 다른 위치에 network 정의

검증 환경: office IDC (172.16.10.0/16) DHCP non-sticky

### upstream PR 시도 (Fix #3)

`nohup sleep && reboot` 패턴은 보편적 race condition 해결책.
OSS PR 시도 가치 있음 (보류/reject 가능성 인지하고).

### 분기별 upstream sync 자동화

cron 또는 GitHub Actions 로 분기마다 upstream/main fetch + conflict notification.
