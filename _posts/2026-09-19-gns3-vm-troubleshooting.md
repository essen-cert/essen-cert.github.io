---
title: "[GNS3] Windows 11에서 GNS3 VM 구축 시 vmrun 및 Hypervisor 오류 해결: Hypervisor"
date: 2026-09-19
categories:
  - Lab
tags:
  - GNS3
  - VMware
  - Windows 11
  - Hypervisor
  - Hyper-V
  - VBS
author_profile: true
---


![GNS3 VM AMD-V RVI 오류](/assets/images/gns3-vm/vmrun-finish.png)
*GNS3 VM 실행 시 발생한 Virtualized AMD-V/RVI 오류*

## 1. 오류 발생

이전 글에서 환경 변수 `Path`에 VMware Workstation 설치 경로를 추가하여
`VMware vmrun tool could not be found` 오류를 해결하였습니다.

그러나 `vmrun` 문제를 해결한 뒤 GNS3 VM을 실행하자 새로운 오류가 발생하였습니다.

```text
Virtualized AMD-V/RVI is not supported on this platform.

Continue without virtualized AMD-V/RVI?
```

![GNS3 VM AMD-V RVI 오류](/assets/images/gns3-vm/hypervisor-error.png)

*그림 1. GNS3 VM 실행 시 발생한 Virtualized AMD-V/RVI 오류*

오류 메시지에서는 현재 플랫폼에서 `Virtualized AMD-V/RVI`를 지원하지 않는다고 표시되고 있었습니다.

`AMD-V`는 AMD CPU에서 제공하는 하드웨어 가상화 기술이며, VMware에서 GNS3 VM과 같은 가상 머신 내부에 다시 가상화 기능을 제공하려면 중첩 가상화(Nested Virtualization)를 사용할 수 있어야 합니다.

따라서 단순히 GNS3 VM 자체의 문제가 아니라, Windows 호스트의 가상화 환경과 VMware의 가상화 기능 간에 문제가 발생하고 있을 가능성을 확인할 필요가 있었습니다.
