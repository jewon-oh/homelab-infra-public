# 가상화 및 VM 프로비저닝

Proxmox VE 기반 가상화 환경과, Terraform으로 VM 생성을 코드화한 과정을 정리합니다. 이 위에 K3s 클러스터가 올라가지만, 이 문서는 그 아래 계층인 **호스트 가상화와 VM 프로비저닝**에 초점을 둡니다.

## 물리 호스트

| 호스트 | CPU | RAM | 역할 |
|--------|-----|-----|------|
| UM880 Pro | Ryzen 8845HS (8C/16T) | 64GB | Proxmox VE 노드 |
| WTR-Pro | Ryzen 5825U (8C/16T) | 64GB | Proxmox VE 노드 (+ TrueNAS VM) |

- 두 미니 PC에 Proxmox VE를 설치해 가상화 호스트로 사용합니다.
- 각 호스트는 2개 물리 NIC를 **802.3ad 본딩**(bond0)으로 묶고, 그 위에 `vmbr0`(LAN) 브리지를 올려 VM에 연결합니다. 전 구간 MTU 1500으로 통일.

## VM 프로비저닝 (Terraform)

VM을 손으로 만들지 않고 `bpg/proxmox` provider로 코드화했습니다. 노드를 추가하거나 재생성할 때 동일한 사양으로 반복 생성할 수 있습니다.

### 프로비저닝 흐름

1. **Cloud-Init 템플릿 준비**: 각 Proxmox 노드에 Debian 12 cloud 이미지로 템플릿 VM(ID 9000)을 한 번 만들어 둡니다.
2. **Terraform으로 클론**: `for_each`로 server/worker VM을 정의하고, 템플릿을 클론해 생성합니다.
3. **Cloud-Init 주입**: IP, Gateway, DNS, 사용자 계정, SSH 공개키를 Terraform 변수로 주입해 부팅과 동시에 네트워크·접속이 설정됩니다.
4. **소프트웨어 설치는 Ansible 담당**: Terraform은 VM 생성까지만 맡고, K3s 설치·구성은 Ansible로 분리했습니다. (역할 분리)

```hcl
resource "proxmox_virtual_environment_vm" "k3s_server" {
  for_each  = var.servers
  node_name = each.value.proxmox_node
  vm_id     = each.value.vm_id

  clone { vm_id = each.value.template_vm_id }

  cpu    { cores = each.value.cpu_cores; type = "host" }
  memory { dedicated = each.value.memory; floating = each.value.memory }

  network_device { bridge = each.value.bridge; model = "virtio"; mtu = 1500 }

  initialization {
    ip_config { ipv4 { address = each.value.ip; gateway = each.value.gateway } }
    user_account { username = var.vm_user; keys = var.ssh_public_keys }
  }
}
```

### 디스크 / 메모리 튜닝

- **`scsi_hardware = "virtio-scsi-single"` + `iothread = true`**: 디스크마다 독립 I/O 스레드를 두어 디스크 I/O가 다른 작업을 막지 않도록 했습니다.
- **`aio = "threads"`, `cache = "none"`, `discard = "on"`, `ssd = true`**: SSD 환경에 맞춘 I/O 옵션과 TRIM 활성화.
- **메모리 ballooning 비활성화 (`floating = dedicated`)**: 호스트 RAM 여유가 충분해, VM이 할당된 메모리를 고정으로 쓰도록 했습니다. (동적 재분배로 인한 성능 변동 방지)
- **추가 데이터 디스크**: Longhorn 등에 쓸 디스크를 `extra_disks`로 정의하면 `scsi1`부터 순서대로 붙습니다.

## iGPU 패스스루

미디어 트랜스코딩(Jellyfin 등) 가속을 위해, GPU가 필요한 워커 VM에 호스트의 내장 GPU(iGPU)를 패스스루했습니다. IOMMU 그룹 분리나 vBIOS 처리는 메인보드·CPU마다 조건이 달라, 같은 8845HS 환경의 [커뮤니티 정리 글](https://atl.kr/dokuwiki/doku.php/amd_ryzen_8845hs_igpu_passthrough)을 참고해 적용했습니다.

**Terraform 쪽** — GPU 워커는 일반 워커와 부팅 방식이 달라, 조건부로 설정합니다.

- `machine = "q35"`, `bios = "ovmf"` (UEFI), `vga { type = "std" }`
- `hostpci`로 매핑된 GPU를 `xvga = true`로 연결
- `efi_disk`의 `pre_enrolled_keys = false`

**수동 관리 보호** — GPU 패스스루의 세부 설정(`hostpci`, `efi_disk`, `bios`, `machine` 등)은 호스트에서 수동으로 맞춰야 하는 부분이 있어, Terraform이 덮어쓰지 않도록 `lifecycle.ignore_changes`로 보호했습니다.

```hcl
lifecycle {
  ignore_changes = [ bios, machine, efi_disk, hostpci, cpu, vga ]
}
```

**호스트 쪽 필수 설정**
- `kvm.conf`: `options kvm ignore_msrs=1`
- `vfio.conf`: `options vfio-pci disable_idle_d3=1`
- `amdgpu`/`radeon` 모듈 블랙리스트 (호스트가 GPU를 선점하지 않도록)
- 게스트에 `firmware-amd-graphics` 설치

> 두 호스트의 iGPU가 서로 달라(8845HS의 Phoenix3, 5825U의 Barcelo) 각각 다른 vBIOS 파일을 사용합니다.

## 관련 문서

- [네트워크 구조](network-architecture.md)
- [스토리지 (TrueNAS / Longhorn)](storage.md)
- [K3s HA 설치 가이드](k3s-ha-setup.md)
