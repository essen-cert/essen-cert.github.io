---
title: "[GNS3] Windows 11에서 GNS3 VM 구축 시 vmrun 및 Hypervisor 오류 해결"
date: 2026-09-17
categories:
  - Lab
tags:
  - GNS3
  - VMware
  - Windows 11
  - Virtualization
author_profile: true
---
## 1. 구축 환경

보안 실습 및 네트워크 테스트 환경을 구성하기 위해 Windows 11 환경에서 GNS3와 GNS3 VM을 구축하였습니다.

### 환경

- OS: Windows 11
- CPU: AMD Ryzen
- Hypervisor: VMware Workstation Pro
- Network Emulator: GNS3
- VM: GNS3 VM

GNS3 VM을 VMware Workstation과 연동하는 과정에서 두 가지 문제가 발생하였습니다.

1. `vmrun.exe`를 찾지 못하는 오류
2. Hypervisor가 감지되어 GNS3 VM이 정상적으로 실행되지 않는 문제

본 글에서는 각 오류가 발생한 환경과 원인을 확인하고 해결한 과정을 정리합니다.

## 2. vmrun.exe 오류

### 2.1. 오류 발생

기존에도 VMware Workstation과 GNS3 VM을 이용하여 실습 환경을 구축해 사용해 왔습니다.

이전 환경을 구축할 때는 VMware Workstation을 먼저 설치한 상태에서 GNS3와 GNS3 VM을 구성하였으며, 당시에는 `vmrun`과 관련된 오류가 발생하지 않았습니다.

이번에는 새로운 테스트 환경을 구성하면서 이전과 달리 VMware Workstation을 설치하기 전에 GNS3와 GNS3 VM을 먼저 설치하였습니다.

GNS3 VM을 VMware와 연동하는 과정에서 다음과 같은 오류가 발생하였습니다.

`VMware vmrun tool could not be found`

![GNS3 vmrun 오류 화면](/assets/images/gns3-vm/vmrun-error.png)

*GNS3에서 VMware vmrun 도구를 찾지 못하는 오류*

최초에는 VMware Workstation을 아직 설치하지 않은 상태였기 때문에 VMware 관련 구성 요소가 존재하지 않아 발생한 오류로 판단하였습니다.


### 2.2. vmrun.exe란?

`vmrun.exe`는 VMware에서 제공하는 명령줄 유틸리티로, 명령줄이나 외부 프로그램에서 VMware 가상 머신을 제어하는 데 사용됩니다.

GNS3 역시 VMware에서 실행되는 GNS3 VM을 제어하기 위해 `vmrun.exe`를 사용합니다.

따라서 VMware Workstation이 설치되어 있지 않거나 GNS3가 `vmrun.exe`의 위치를 정상적으로 확인하지 못하는 경우 GNS3 VM과의 연동 과정에서 오류가 발생할 수 있습니다.


### 2.3. VMware 설치 후에도 지속된 오류

최초 오류 발생 당시에는 VMware Workstation이 설치되어 있지 않았기 때문에 VMware Workstation Pro를 설치한 후 다시 GNS3 VM 연동을 시도하였습니다.

기존 환경에서는 VMware를 먼저 설치한 뒤 GNS3를 구성했을 때 별도의 `vmrun` 오류가 발생하지 않았기 때문에, 이번에도 VMware 설치를 통해 문제가 해결될 것으로 예상하였습니다.

그러나 VMware Workstation을 정상적으로 설치한 이후에도 동일한 `VMware vmrun tool could not be found` 오류가 계속 발생하였습니다.

이에 따라 단순히 VMware가 설치되어 있지 않아 발생한 문제가 아니라, GNS3가 설치된 `vmrun.exe`를 정상적으로 탐색하지 못하고 있을 가능성을 확인하였습니다.


### 2.4. vmrun.exe 경로 확인

VMware Workstation의 실제 설치 경로를 확인한 결과 `vmrun.exe` 파일 자체는 정상적으로 존재하고 있었습니다.

![vmrun.exe 파일 경로 확인](/assets/images/gns3-vm/vmrun-path.png)

*VMware Workstation 설치 경로에 존재하는 vmrun.exe*

즉, 오류 메시지만 보면 `vmrun` 도구가 설치되지 않은 것처럼 보일 수 있지만 실제로는 실행 파일이 존재하는 상태였습니다.

따라서 문제의 범위를 `vmrun.exe`의 설치 여부가 아닌 GNS3가 해당 실행 파일의 위치를 탐색하는 과정으로 좁힐 수 있었습니다.


### 2.5. VMware 설치 경로와 GNS3의 vmrun 탐색

![VMware Workstation 버전](/assets/images/gns3-vm/vmware-version.png)

사용 중인 VMware Workstation 버전에서는 `vmrun.exe`가 다음 VMware Workstation 설치 경로에 존재하였습니다.

`C:\Program Files\VMware\VMware Workstation\`

기존 VMware Workstation에서는 설치 경로가 `Program Files (x86)` 하위에 위치하는 경우가 있었으나, 64비트 전용 VMware Workstation 26H1에서는 기본 설치 위치가 `Program Files`로 변경되었습니다.

당시 사용한 GNS3에서는 변경된 VMware 설치 경로의 `vmrun.exe`를 자동으로 정상 탐색하지 못하는 문제가 보고되어 있었습니다.

따라서 이번 현상 역시 VMware가 설치되지 않아 발생한 문제가 아니라, VMware 설치 후에도 GNS3가 변경된 경로에 존재하는 `vmrun.exe`를 정상적으로 찾지 못하면서 발생한 것으로 판단하였습니다.


### 2.6. 환경 변수 Path 추가

```cmd
where vmrun

문제 해결을 위해 Windows 시스템 환경 변수 `Path`에 VMware Workstation 설치 경로를 추가하였습니다.

`C:\Program Files\VMware\VMware Workstation\`

![환경 변수 Path 설정 이미지](/assets/images/gns3-vm/vmrun-add-path.png)

*그림 4. VMware Workstation 설치 경로를 Windows Path 환경 변수에 추가*

Path 환경 변수에 해당 디렉터리를 등록하면 실행 파일의 전체 경로를 직접 입력하지 않아도 Windows에서 해당 경로의 실행 파일을 검색할 수 있습니다.

설정 후 GNS3를 다시 실행하여 GNS3 VM 연동을 시도하였습니다.


### 2.7. 결과

환경 변수 Path에 VMware Workstation 설치 경로를 추가한 이후 기존의 `VMware vmrun tool could not be found` 오류가 더 이상 발생하지 않았습니다.

이번 테스트에서는 설치 순서가 기존 환경과 달랐기 때문에 처음에는 VMware 미설치를 원인으로 판단하였으나, VMware를 설치한 이후에도 동일한 오류가 지속되었습니다.

이후 `vmrun.exe`의 실제 존재 여부와 설치 경로를 확인하고 Path를 추가하는 과정을 통해 문제를 해결하였습니다.

이를 통해 오류 메시지만을 기준으로 원인을 판단하기보다 실제 파일의 존재 여부와 프로그램이 해당 실행 파일을 탐색하는 경로까지 단계적으로 확인할 필요가 있음을 확인하였습니다.
