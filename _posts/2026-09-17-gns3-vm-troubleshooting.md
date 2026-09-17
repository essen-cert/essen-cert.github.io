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

### 2.1 오류 발생

GNS3를 먼저 설치한 뒤 GNS3 VM을 VMware와 연동하려고 하자 `vmrun.exe`를 찾을 수 없다는 오류가 발생하였습니다.

![GNS3 vmrun 오류 화면](/assets/images/gns3-vm/vmrun-error.png)

*그림 2. GNS3에서 VMware vmrun 도구를 찾지 못하는 오류*

### 2.2 vmrun.exe란?

`vmrun.exe`는 VMware에서 제공하는 명령줄 유틸리티로, 외부 프로그램이나 명령줄에서 VMware 가상 머신을 제어할 때 사용됩니다.

GNS3는 VMware에서 실행되는 GNS3 VM을 제어하기 위해 `vmrun.exe`를 사용합니다.

따라서 GNS3가 설치되어 있더라도 VMware Workstation이 설치되어 있지 않거나 `vmrun.exe`의 경로를 정상적으로 확인하지 못하면 GNS3 VM과의 연동이 정상적으로 이루어지지 않을 수 있습니다.

### 2.3 원인 확인

오류 발생 당시에는 GNS3만 설치되어 있었으며 VMware Workstation은 설치하지 않은 상태였습니다.

따라서 시스템에 `vmrun.exe`가 존재하지 않았고, GNS3가 VMware를 통한 GNS3 VM 제어 기능을 사용할 수 없는 상태였습니다.

### 2.4 해결

VMware Workstation Pro를 설치한 뒤 다시 GNS3의 GNS3 VM 설정을 확인하였습니다.

[VMware 설치 후 GNS3 VM 설정 화면 이미지]

*그림 3. VMware Workstation 설치 후 GNS3 VM 연동 확인*

VMware 설치 이후 GNS3에서 VMware 관련 기능을 정상적으로 사용할 수 있었으며, 기존의 `vmrun.exe` 관련 오류도 더 이상 발생하지 않았습니다.
