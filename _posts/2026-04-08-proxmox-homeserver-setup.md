---
layout: post
title: "미니 PC에 Proxmox 올려서 홈서버 구축한 후기"
date: 2026-04-08
categories: [homeserver]
tags: [Proxmox, Virtualization, HomeServer, Ryzen, Network]
---

집에서 놀고 있던 미니 PC(Ryzen 7 5825U)를 서버로 굴려보고 싶어서, Proxmox를 설치하고 가상화 환경을 세팅했다. 그냥 Docker만 띄울까 하다가 Web, WAS, DB, Storage를 각각 분리해서 실무처럼 구성해보기로 했다.

삽질이 꽤 있었는데, 특히 네트워크 쪽에서 시간을 많이 잡아먹었다.

## Proxmox를 선택한 이유

ESXi도 고려했는데, 라이선스 정책이 바뀌면서 개인 사용이 애매해졌다. Proxmox는 무료인데다 LXC 컨테이너까지 지원해서, 가벼운 서비스는 LXC로 무거운 건 VM으로 나눠 쓸 수 있다는 게 결정적이었다.

설치 자체는 간단하다. Rufus로 ISO를 DD 모드로 구워서 USB 부팅하면 끝. 다만 미니 PC 특성상 **후면 USB 포트**로 부팅해야 인식이 잘 됐고, BIOS에서 **Secure Boot는 꺼줘야** 한다.

## 네트워크 구성 - 여기서 삽질을 많이 했다

처음에는 벽면 단자에서 바로 스위치로 연결했더니, 본체(192.168.219.x)와 미니 PC(192.168.123.x)가 서로 다른 대역에 잡혀서 Proxmox 웹 콘솔에 접속이 안 됐다.

결국 아래처럼 공유기를 앞에 두는 구조로 바꿨다.

```
벽면 단자 → 공유기(AX3000Q) → 2.5G 스위치(HG25005T1) → 본체 & 미니 PC
```

이렇게 하면 공유기가 DHCP를 담당하니까 모든 기기가 `192.168.219.x` 대역으로 통일된다. 기기 간 통신도 되고 Proxmox 관리 페이지 접속도 문제없다.

Proxmox 쪽 네트워크 설정은 `/etc/network/interfaces`에서 브릿지를 잡아주면 된다.

```
auto lo
iface lo inet loopback

iface nic0 inet manual

auto vmbr0
iface vmbr0 inet static
  address 192.168.219.103/24
  gateway 192.168.219.1
  bridge-ports nic0
  bridge-stp off
  bridge-fd 0
```

게이트웨이를 공유기 주소(219.1)로 맞춰야 외부 통신이 된다. 여기서 대역이 안 맞으면 설치 후 네트워크가 먹통이 되니까 꼭 확인하자.

## 리소스 할당

32GB RAM을 어떻게 나눌지 고민을 좀 했다. VM과 LXC를 섞어서 다음과 같이 배분했다.

| 이름 | 타입 | 역할 | CPU | RAM | 비고 |
|------|------|------|-----|-----|------|
| Web | LXC | Nginx Proxy | 1 Core | 1GB | 외부 트래픽 관문 |
| WAS | VM | Docker + Spring | 6 Cores | 12GB | 메인 개발 환경 |
| DB | LXC | MySQL/PostgreSQL | 2 Cores | 4GB | 데이터 저장소 |
| Storage | VM | NAS/Files | 2 Cores | 4GB | HDD RAID-Z1 예정 |

몇 가지 포인트:

- **WAS VM의 CPU Type은 `Host`로 설정했다.** 기본값(kvm64)으로 두면 가상화 오버헤드가 크다. Host로 바꾸면 물리 CPU 명령어를 그대로 쓸 수 있어서 Java 빌드 속도가 체감될 정도로 빨라진다.
- **Memory Ballooning 활성화.** WAS에 12GB를 잡아뒀지만 항상 다 쓰는 건 아니니까, 안 쓸 때는 다른 VM에 양보하도록 했다.
- **LXC로 띄울 수 있는 건 LXC로.** Nginx나 DB처럼 커널을 공유해도 괜찮은 서비스는 LXC가 훨씬 가볍다.

Storage VM에는 나중에 HDD 3개를 PCI Pass-through로 연결해서 ZFS RAID-Z1을 구성할 계획이다.

## 삽질 기록

구축하면서 겪은 이슈들을 정리해둔다.

**VM이 삭제가 안 된다 (자물쇠 아이콘)**

Proxmox 웹 UI에서 VM을 지우려는데 잠금 상태라고 뜨는 경우가 있다. 쉘에서 아래 명령으로 해제하면 된다.

```bash
qm unlock [VMID]
```

**디스크 용량이 부족하다**

LVM 설치 시 ubuntu-lv가 기본으로 전체 용량을 안 잡는 경우가 있다. 설치할 때 디스크 할당 화면에서 용량을 Max로 직접 입력해줘야 한다.

## 다음 할 것

- Docker VM 위에 Jenkins 올려서 CI/CD 파이프라인 구성
- Nginx Proxy Manager로 리버스 프록시 + SSL 인증서 자동 갱신
- 외부 접속용 포트 포워딩 및 도메인 연결

다음 글에서는 WAS VM에 Docker를 설치하고 Spring Boot 앱을 배포하는 과정을 정리할 예정이다.
