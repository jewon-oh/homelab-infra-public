# 스토리지

홈랩 스토리지는 두 축으로 구성됩니다. 영구 데이터의 원본은 **TrueNAS**(ZFS RAID-Z2)가, 클러스터 파드의 블록 스토리지는 **Longhorn**이 담당합니다.

## TrueNAS (ZFS RAID-Z2)

WTR-Pro(Ryzen 5825U) 호스트 위에 TrueNAS를 **VM**으로 올리고, HDD들을 ZFS RAID-Z2 풀로 묶어 NAS로 사용합니다. NFS로 다른 VM과 PBS 백업 저장소에 제공합니다.

### VM + 디스크 패스스루

ZFS는 체크섬 검증과 SMART 모니터링이 제대로 동작하려면 **디스크를 직접 제어**하는 것이 좋습니다. QEMU 가상 디스크 이미지를 거치면 이 계층이 가려지므로, 디스크를 패스스루 방식으로 VM에 연결했습니다. 단 컨트롤러 패스스루 디스크는 VM(TrueNAS)에서 SMART가 보이지만, by-id로 넘긴 디스크는 VM 대신 **Proxmox 호스트 측에서 SMART를 확인**합니다(아래 트레이드오프 참고).

디스크 4개를 **두 방식으로 혼합**해서 붙였습니다.

| 방식 | 대상 | 이유 |
|------|------|------|
| SATA 컨트롤러 패스스루 (IOMMU) | HDD 2개 | 컨트롤러째 VM에 넘겨 디스크를 직접 인식 |
| `/dev/disk/by-id` 개별 패스스루 | HDD 2개 | 컨트롤러는 호스트에 남기고 디스크만 개별 연결 (SMART는 VM이 아닌 호스트에서 모니터링) |

> **왜 통째로 안 넘겼나**: 5825U는 SATA 컨트롤러가 SoC에 통합돼 있어, 컨트롤러를 통째로 IOMMU 그룹으로 VM에 넘기면 **CPU 온도 센서까지 VM으로 함께 넘어가** Proxmox 호스트가 CPU 온도를 읽지 못합니다. 호스트가 온도를 못 읽으면 온도 기반 클럭 조절을 하지 못해 **CPU 클럭이 고정**되는 문제가 발생했습니다. 그래서 일부 디스크는 컨트롤러째, 일부는 by-id로 개별 연결하는 혼합 구성을 택했습니다.

```
Proxmox Host (WTR-Pro)
└── TrueNAS VM
    └── ZFS Pool (RAID-Z2)
        ├── HDD 1  (/dev/disk/by-id 개별 패스스루)
        ├── HDD 2  (/dev/disk/by-id 개별 패스스루)
        ├── HDD 3  (SATA Controller 패스스루)
        └── HDD 4  (SATA Controller 패스스루)
```

### 디스크 구성

- **WD Red 4TB** (NAS용 HDD)로 RAID-Z2 풀을 구성했습니다. NAS 전용 디스크라 24시간 가동과 RAID 환경(에러 복구 타임아웃 등)에 맞게 설계된 모델입니다.

### 왜 ZFS / RAID-Z2 인가

- **디스크 2개 동시 고장까지 견딤(RAID-Z2)**: 가족 사진·문서 등 원본 데이터를 두는 곳이라, 1개 여유(RAID-Z1)보다 2개 여유가 안전하다고 판단했습니다. 디스크 교체 중 두 번째 디스크가 죽는 상황까지 대비합니다.
- **스냅샷**: 데이터를 특정 시점으로 되돌릴 수 있어, 실수로 지우거나 덮어썼을 때 복구가 쉽습니다.
- **체크섬**: 블록마다 체크섬을 저장해 **사일런트 데이터 손상**(bit rot)을 탐지하고, 이중화된 데이터로 복구할 수 있습니다.

## Longhorn (클러스터 블록 스토리지)

파드의 영속 볼륨(PVC)은 Longhorn 분산 블록 스토리지로 제공합니다.

- StorageClass는 `longhorn` 하나로 통일했습니다. (기본값, 복제본 2, 회수 정책 Retain)
- 복제본 2개로 한 노드가 빠져도 볼륨이 유지됩니다.
- 노드의 추가 데이터 디스크(Terraform `extra_disks`)를 Longhorn 디스크로 사용합니다.

> 초기에 StorageClass를 용도별로 여러 개(`longhorn-fast`, `longhorn-1rep` 등) 두었다가, 관리 복잡도만 늘고 실익이 적어 하나로 통일했습니다. 기존 볼륨은 데이터 복사 없이 PV의 `claimRef`를 비우고 재바인딩하는 방식으로 옮겼습니다.

## 관련 문서

- [가상화 및 VM 프로비저닝](virtualization.md)
- [백업 전략](backup-strategy.md)
