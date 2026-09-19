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


## 1. 오류 발생

이전 글에서 환경 변수 `Path`에 VMware Workstation 설치 경로를 추가하여
`VMware vmrun tool could not be found` 오류를 해결했다.

그러나 `vmrun` 문제를 해결한 뒤 GNS3 VM을 실행하자 새로운 오류가 발생했다.

```text
Virtualized AMD-V/RVI is not supported on this platform.

Continue without virtualized AMD-V/RVI?
```

![GNS3 VM AMD-V RVI 오류](/assets/images/gns3-vm/vmrun-finish-vmware.png)
*GNS3 VM 실행 시 발생한 Virtualized AMD-V/RVI 오류*


## 2. 오류 원인 확인

오류 메시지에서는 현재 플랫폼에서 `Virtualized AMD-V/RVI`를 지원하지 않는다고 표시되고 있었다

`AMD-V`는 AMD CPU의 하드웨어 가상화 기술이며, `RVI(Rapid Virtualization Indexing)`는 AMD의 메모리 가상화 지원 기술이다.

VMware Workstation에서 `Virtualized AMD-V/RVI` 옵션을 사용하면 호스트 CPU의 가상화 기능을 게스트 VM에 노출하여 VM 내부에서도 가상화 기능을 사용할 수 있다.

즉, 현재 오류는 **VMware가 GNS3 VM에 중첩 가상화(Nested Virtualization)를 제공하지 못하고 있는 상태**로,

GNS3 VM 자체의 문제가 아니라, Windows 호스트의 가상화 환경과 VMware의 가상화 기능 간에 문제가 발생하고 있을 가능성을 확인할 필요가 있었다.

따라서 먼저 Windows 호스트에서 가상화 기능과 Hypervisor가 어떤 상태로 동작하고 있는지 확인했다.


## 3. 가상화 설정 확인

### 3.1. VMware 가상화 설정 확인

먼저 GNS3 VM의 VMware 설정을 확인했다.

VMware Workstation에서 GNS3 VM의 `Settings → Processors` 항목을 확인한 결과, `Virtualize Intel VT-x/EPT or AMD-V/RVI` 옵션은 이미 활성화되어 있었다.

![GNS3 VM Virtualization Engine 설정](/assets/images/gns3-vm/virtualization-engine.png)
*GNS3 VM의 Virtualization Engine 설정*

해당 옵션은 호스트의 하드웨어 가상화 기능을 게스트 VM에 노출하기 위한 설정이다.

따라서 단순히 VMware의 중첩 가상화 옵션이 비활성화되어 발생한 문제는 아니었다.

다음으로 Windows 호스트에서 하드웨어 가상화 및 Hypervisor가 어떤 상태로 동작하고 있는지 확인했다.


### 3.2. Windows 가상화 상태 확인

VMware의 가상화 옵션이 이미 활성화되어 있었기 때문에 Windows 호스트의 가상화 상태를 확인했다.

시스템 정보의 `시스템 요약`을 확인한 결과 다음과 같은 상태가 표시되어 있었다.

- 가상화 기반 보안: `실행 중`
- 가상화 기반 보안 서비스 구성: `하이퍼바이저 적용 코드 무결성`
- 가상화 기반 보안 서비스 실행 중: `하이퍼바이저 적용 코드 무결성`
- 하이퍼바이저가 검색되었습니다. Hyper-V에 필요한 기능이 표시되지 않습니다.

![Windows 시스템 정보에서 확인한 가상화 상태](/assets/images/gns3-vm/msinfo-hypervisor.png)
*msinfo32에서 확인한 VBS 및 Hypervisor 실행 상태*

이를 통해 Windows에서 VBS(Virtualization-Based Security)가 실행 중이며, Windows Hypervisor 역시 활성화되어 있음을 확인했다.

VMware의 `Virtualize Intel VT-x/EPT or AMD-V/RVI` 옵션이 활성화되어 있음에도 중첩 가상화 오류가 발생하고 있었기 때문에, 다음으로 VBS와 관련된 Windows 보안 기능을 확인했다.


### 3.3. 메모리 무결성 비활성화

`msinfo32`에서 VBS와 하이퍼바이저 적용 코드 무결성이 실행 중인 것을 확인한 뒤, Windows 보안의 `코어 격리` 설정을 확인했다.

메모리 무결성은 활성화되어 있었으며, 해당 기능이 현재 가상화 환경에 영향을 주고 있는지 확인하기 위해 테스트 목적으로 비활성화했다.

`Windows 보안 > 장치 보안 > 코어 격리`에서 `메모리 무결성`을 `끔`으로 변경했다.

![Windows 코어 격리 메모리 무결성 비활성화](/assets/images/gns3-vm/memory-integrity-off.png)
*메모리 무결성 비활성화*


### 3.4. 메모리 무결성 비활성화 후 상태 확인

메모리 무결성을 비활성화한 뒤 시스템을 재부팅하고 다시 `msinfo32`를 실행했다.

재부팅 전에는 `가상화 기반 보안`과 `하이퍼바이저 적용 코드 무결성` 서비스가 실행 중인 상태였으나,

재부팅 후에는 기존에 표시되던 VBS 관련 실행 정보가 더 이상 나타나지 않았다.

![메모리 무결성 비활성화 후 시스템 정보](/assets/images/gns3-vm/msinfo-memory-integrity-off.png)
*메모리 무결성 비활성화 및 재부팅 후 가상화 상태*

하지만 화면 하단에는 여전히 다음 메시지가 표시되고 있었다.

```text
하이퍼바이저가 검색되었습니다. Hyper-V에 필요한 기능이 표시되지 않습니다.
```

즉 메모리 무결성을 비활성화한 이후에도 Windows Hypervisor는 계속 동작하고 있었다.

따라서 메모리 무결성만 비활성화하는 것으로는 문제가 해결되지 않았으며,
Windows Hypervisor가 활성화되는 다른 원인이 있는지 추가로 확인했다.


### 3.5. Hyper-V 및 VBS 관련 설정 확인

DISM을 이용해 Hyper-V 및 가상화 관련 Windows 기능의 활성화 상태를 확인했다.

```cmd
dism /online /get-features /format:table | findstr /I "Hyper-V VirtualMachinePlatform HypervisorPlatform"
```

확인 결과 Hyper-V 관련 기능은 모두 `사용 안 함` 상태였다.

![Hyper-V 관련 Windows 기능 확인](/assets/images/gns3-vm/hyperv-features.png)
*DISM을 이용한 Hyper-V 및 가상화 관련 기능 확인*

Windows 선택적 기능의 Hyper-V가 직접 활성화되어 Hypervisor가 실행되고 있는 상황은 아니었다.

다음으로 부팅 구성에서 Hypervisor 및 VSM(Virtual Secure Mode)의 실행 설정을 확인했다.

```cmd
bcdedit /enum {current} | findstr /I "hypervisorlaunchtype vsmlaunchtype"
```

그러나 해당 명령어에서는 `hypervisorlaunchtype`이나 `vsmlaunchtype` 값이 별도로 출력되지 않았다.

![Hypervisor 부팅 설정 확인](/assets/images/gns3-vm/hypervisor-boot-config.png)
*BCDEdit을 이용한 Hypervisor 부팅 설정 확인*

Hyper-V 관련 Windows 기능이 비활성화되어 있음에도 Hypervisor가 계속 실행되고 있었기 때문에, 다음으로 VBS와 관련된 Device Guard 레지스트리 설정을 확인했다.

```cmd
reg query "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard" /s
```

확인 결과 메모리 무결성과 관련된 `HypervisorEnforcedCodeIntegrity`의 `Enabled` 값은 `0x0`으로 비활성화되어 있었다.

반면 Device Guard 영역에서는 다음과 같은 설정을 확인할 수 있었다.

```text
EnableVirtualizationBasedSecurity    REG_DWORD    0x1
```

메모리 무결성 자체는 비활성화되어 있었지만, VBS(Virtualization-Based Security)를 활성화하는 설정은 여전히 남아 있었다.

![Device Guard VBS 설정 확인](/assets/images/gns3-vm/device-guard-vbs.png)
*Device Guard 레지스트리에서 확인한 VBS 관련 설정*


## 4. 문제 해결

### 4.1. VBS 비활성화

Device Guard의 `EnableVirtualizationBasedSecurity` 값은 `0x1`로 설정되어 있었다.

VBS가 Windows Hypervisor 실행에 영향을 주고 있는지 확인하기 위해 해당 값을 비활성화했다.

레지스트리를 직접 변경하기 전에 문제가 발생할 경우 기존 상태로 복구할 수 있도록 Device Guard 설정을 먼저 백업한 후 비활성화했다.

```cmd #DeviceGuard 백업
reg export "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard" "%USERPROFILE%\Desktop\DeviceGuard-backup.reg"
```

```cmd #EnableVirtualizationBasedSecurity 비활성화
reg add "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard" /v EnableVirtualizationBasedSecurity /t REG_DWORD /d 0 /f
```

```cmd #확인
reg query "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard" /v EnableVirtualizationBasedSecurity
```

확인 결과 값이 다음과 같이 변경되어 있었다.

```text
EnableVirtualizationBasedSecurity    REG_DWORD    0x0
```

![Device Guard VBS 비활성화](/assets/images/gns3-vm/vbs-disable.png)
*Device Guard 레지스트리 백업 및 VBS 비활성화*

이로써 레지스트리상 VBS 활성화 값은 `0`으로 변경되었다.

변경 사항이 실제 부팅 환경에 반영되는지 확인하기 위해 시스템을 재부팅한 뒤 다시 가상화 상태를 확인했다.


### 4.2. VBS 비활성화 후 Hypervisor 상태 확인

`EnableVirtualizationBasedSecurity` 값을 `0`으로 변경한 뒤 시스템을 재부팅하고 다시 `msinfo32`를 실행했다.

VBS를 비활성화했기 때문에 Windows Hypervisor의 실행 상태에도 변화가 있을 것으로 예상했지만, 시스템 정보 하단에는 여전히 다음 메시지가 표시되고 있었다.

![VBS 비활성화 후 Hypervisor 상태](/assets/images/gns3-vm/msinfo-vbs-disabled.png)
*VBS 비활성화 및 재부팅 후에도 감지되는 Hypervisor*

`EnableVirtualizationBasedSecurity` 값을 비활성화하는 것만으로는 Windows Hypervisor의 실행을 중지할 수 없었다.

다른 설정에 의해 Windows Hypervisor가 계속 실행되고 있을 가능성을 확인할 필요가 있었다.


### 4.3. Windows Hypervisor 자동 실행 비활성화

VBS를 비활성화한 이후에도 Windows Hypervisor가 계속 감지되었기 때문에, 이번에는 부팅 과정에서 Windows Hypervisor가 실행되지 않도록 직접 설정해 보기로 했다.

```cmd #Windows Hypervisor 부팅 시 자동 실행 비활성화
bcdedit /set hypervisorlaunchtype off
```

`hypervisorlaunchtype`은 Windows 부팅 시 Hypervisor의 실행 여부를 제어하는 BCD(Boot Configuration Data) 설정이다.


```cmd #확인
bcdedit /enum {current} | findstr /I "hypervisorlaunchtype"
```

확인 결과 다음과 같이 `Off`로 설정된 것을 확인했다.

```text
hypervisorlaunchtype    Off
```

![Windows Hypervisor 자동 실행 비활성화](/assets/images/gns3-vm/hypervisor-launch-off.png)
*hypervisorlaunchtype을 Off로 변경한 결과*

BCD 설정은 다음 부팅부터 적용되므로 시스템을 재부팅한 뒤 Windows Hypervisor의 실행 여부를 다시 확인했다.


### 4.4. PowerShell을 통한 VBS 상태 확인

`hypervisorlaunchtype`을 `Off`로 설정한 이후에도 가상화 보안 기능의 상태를 보다 구체적으로 확인하기 위해 PowerShell에서 `Win32_DeviceGuard` 클래스를 조회했다.

```powershell
Get-CimInstance -ClassName Win32_DeviceGuard -Namespace root\Microsoft\Windows\DeviceGuard | Format-List *
```

확인 결과 주요 값은 다음과 같았다.

```text
SecurityServicesConfigured          : {0}
SecurityServicesRunning             : {0}
VirtualizationBasedSecurityStatus   : 2
```

![PowerShell을 이용한 Device Guard 상태 확인](/assets/images/gns3-vm/deviceguard-status.png)
* Win32_DeviceGuard 클래스를 이용한 VBS 상태 확인*

`SecurityServicesConfigured`와 `SecurityServicesRunning`은 모두 `0`으로 나타났지만, `VirtualizationBasedSecurityStatus`는 `2`로 확인되었다.

`VirtualizationBasedSecurityStatus` 값에서 `2`는 VBS가 실행 중인 상태를 의미하며, Windows에서 Hypervisor가 실제로 감지되고 있는지 확인했다.

```powershell
Get-CimInstance Win32_ComputerSystem | Select-Object HypervisorPresent
```

확인 결과 `HypervisorPresent` 값은 `True`로 나타났다.

```text
HypervisorPresent
-----------------
             True
```

![Windows Hypervisor 감지 상태 확인](/assets/images/gns3-vm/hypervisor-present.png)
*PowerShell에서 확인한 Windows Hypervisor 감지 상태*

앞서 `hypervisorlaunchtype`을 `Off`로 설정했음에도 Hypervisor가 계속 감지되고 있었기 때문에 BCD 설정을 다시 확인했다.

```cmd
bcdedit /enum {current} | findstr /I "isolatedcontext hypervisorlaunchtype"
```

확인 결과는 다음과 같았다.

```text
isolatedcontext        Yes
hypervisorlaunchtype   Off
```

![Windows Hypervisor 감지 상태 확인](/assets/images/gns3-vm/hypervisor-hypervisorlaunchtype.png)
*BCD 설정 재확인*

즉 `hypervisorlaunchtype` 설정 자체는 정상적으로 `Off`로 적용되어 있었지만, Windows에서는 여전히 Hypervisor가 감지되고 있었다.

따라서 단순히 Hypervisor의 자동 실행 설정만으로는 해결되지 않는 다른 원인이 존재한다고 판단하고 추가적인 보안 및 가상화 설정을 확인했다.


### 4.5. 이벤트 로그를 통한 Hypervisor 실행 확인

PowerShell에서 `HypervisorPresent` 값이 `True`로 확인되었기 때문에,
이벤트 로그에서도 Windows Hypervisor의 실제 실행 기록을 확인했다.

이벤트 뷰어의 `Windows 로그 → 시스템`에서 이벤트 원본을 `Hyper-V-Hypervisor`로 필터링한 결과,
Hypervisor와 관련된 여러 이벤트가 기록되어 있었다.

특히 이벤트 ID `1`에서는 다음 내용을 확인할 수 있었다.

```text
하이퍼바이저를 시작했습니다.
```

![Hyper-V Hypervisor 시작 이벤트](/assets/images/gns3-vm/hyperv-hypervisor-event.png)
*이벤트 뷰어에서 확인한 Hyper-V-Hypervisor 시작 기록*

이를 통해 `hypervisorlaunchtype`이 `Off`로 설정되어 있음에도, 실제 시스템에서는 Hypervisor가 시작된 기록이 존재한다는 것을 다시 확인했다.

따라서 단순한 상태 표시 문제가 아니라 실제 부팅 과정에서 Hypervisor가 활성화되고 있는 것으로 판단하고, 이를 발생시키는 다른 Windows 보안 설정을 추가로 확인했다.


### 4.6. Windows Hello 관련 Device Guard 설정 확인

`hypervisorlaunchtype`을 `Off`로 변경한 이후에도 시스템의 가상화 관련 상태를 추가로 확인하였다.

```cmd
bcdedit /enum all | findstr /I "hypervisor vsmlaunch isolatedcontext"
```

확인 결과 `hypervisorlaunchtype`은 `Off`로 설정되어 있었지만 `isolatedcontext`는 `Yes`로 표시되었다.

Device Guard 하위 설정을 추가로 확인하는 과정에서 Windows Hello 관련 레지스트리 값이 활성화되어 있는 것을 확인하였다.

```cmd #Windows Hello 활성화 여부 확인
reg query "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard\Scenarios\WindowsHello" /v Enabled
```

확인 결과 `Enabled` 값이 `0x1`로 설정되어 있었다.

```text
Enabled    REG_DWORD    0x1
```

```cmd #Windows Hello 비활성화
reg add "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard\Scenarios\WindowsHello" /v Enabled /t REG_DWORD /d 0 /f
```

변경 후 다시 확인한 결과 `Enabled` 값이 `0x0`으로 변경되었다.

```text
Enabled    REG_DWORD    0x0
```

![Windows Hello Device Guard 설정 변경](/assets/images/gns3-vm/windowshello.png)
*Device Guard의 Windows Hello 관련 설정을 비활성화한 결과*

설정 변경 사항을 적용하기 위해 시스템을 재부팅한 뒤 Hypervisor 상태를 다시 확인하였다.


### 4.7. 재부팅 후 Hypervisor 상태 확인

Windows Hello 관련 Device Guard 설정을 변경한 뒤 시스템을 재부팅하였다.

재부팅 후 Windows Hypervisor의 동작 여부를 다시 확인하였다.

```powershell
Get-CimInstance Win32_ComputerSystem | Select-Object HypervisorPresent
```

확인 결과 이전에는 `True`로 표시되었던 `HypervisorPresent` 값이 `False`로 변경됐다.

```text
HypervisorPresent
-----------------
False
```

![Hypervisor 비활성화 확인](/assets/images/gns3-vm/hypervisor-disabled.png)
*재부팅 후 HypervisorPresent 값이 False로 변경된 것을 확인*

이를 통해 Windows Hypervisor가 더 이상 실행되고 있지 않은 것을 확인했다.


### 4.8. GNS3 VM 실행 확인

Hypervisor가 비활성화된 상태에서 VMware Workstation을 통해 GNS3 VM을 다시 실행했다.

기존에 발생하던 `Virtualized AMD-V/RVI is not supported on this platform` 오류가 더 이상 발생하지 않았으며, GNS3 VM이 정상적으로 부팅됐다.

또한 GNS3 VM 화면에서 다음과 같이 KVM 지원 상태가 `True`로 표시되는 것을 확인했다.

```text
Virtualization: vmware
KVM support available: True
```

![GNS3 VM 정상 실행](/assets/images/gns3-vm/gns3-vm-success.png)
*VMware Workstation에서 GNS3 VM이 정상적으로 실행된 모습*

이를 통해 VMware의 중첩 가상화 기능을 정상적으로 사용할 수 있는 상태가 되었음을 확인했다.

![GNS3 VM 정상 실행](/assets/images/gns3-vm/gns3-vm-success-msinfo.png)
*Hypervisor 오류 해결 후 msinfo*


### 4.9. Windows Hello 관련 설정 확인

추가적인 원인을 확인하는 과정에서 Device Guard 하위의 Windows Hello 관련 설정이 활성화되어 있는 것을 확인했다.

해당 설정을 비활성화한 뒤 재부팅하자 Hypervisor 관련 문제가 해결되었으나, 기존에 Microsoft 계정에 설정한 PIN을 이용한 Windows 로그인 방식이 비활성화되는 현상이 발생했다.

이에 설정 변경이 Windows Hello 로그인 기능에 미친 영향을 확인하기 위해 Windows의 로그인 옵션을 확인했다.

![Windows Hello 로그인 옵션](/assets/images/gns3-vm/windows-hello.png)
*Windows 11의 Windows Hello 로그인 옵션 확인*


## 5. 원인 역검증

GNS3 VM이 정상적으로 실행된 이후, 문제 해결 과정에서 변경했던 설정들을 하나씩 원래 상태로 되돌리면서 어떤 설정이 Hypervisor 활성화에 영향을 주는지 추가로 확인했다.

여러 설정을 동시에 비활성화한 상태에서 문제가 해결되었기 때문에, 이 상태만으로는 정확히 어떤 설정이 원인이었는지 판단하기 어려웠다.

따라서 GNS3 VM이 정상적으로 실행되는 상태를 기준으로 설정을 하나씩 다시 활성화하고, 각 단계에서 `HypervisorPresent` 값을 확인하는 방식으로 역검증을 진행했다.


### 5.1. 설정별 역검증

설정을 하나씩 원복하면서 Hypervisor 동작 여부를 확인한 결과, VBS와 Windows Hello 관련 설정 등이 Hypervisor 활성화에 영향을 주는 것을 확인했다.

특히 모든 관련 설정을 비활성화했을 때는 `HypervisorPresent`가 `False`로 나타났지만, 일부 설정을 다시 활성화하면 `True`로 변경됐다.

이를 통해 단순히 Hyper-V 기능의 설치 여부만으로 Hypervisor의 동작 상태가 결정되는 것은 아니며, Windows의 가상화 기반 보안 기능 역시 Hypervisor 활성화에 관여할 수 있다는 것을 확인했다.

설정을 하나씩 원복하면서 Hypervisor 동작 여부를 확인한 결과는 다음과 같다.

| VBS | Windows Hello | 메모리 무결성 | HypervisorPresent |
| --- | --- | --- | --- |
| OFF | OFF | OFF | `False` |
| **ON** | OFF | OFF | `True` |
| OFF | **ON** | OFF | `True` |
| OFF | OFF | **ON** | `True` |

각 설정을 개별적으로 활성화했을 때 `HypervisorPresent`가 다시 `True`로 변경되는 것을 확인했다.


## 6. 설정 변경에 따른 보안 영향

이번 문제를 해결하기 위해 VBS, Windows Hello 관련 설정, 메모리 무결성 등 Windows의 가상화 기반 보안과 관련된 기능을 비활성화다.

이러한 설정은 단순히 가상화 기능만 제어하는 것이 아니라 Windows의 보안 기능과도 연관되어 있기 때문에, 실사용 환경에서 동일한 방법을 적용할 경우 보안상 영향을 고려할 필요가 있다.

### 6.1 VBS (Virtualization Based Security)

VBS는 하드웨어 가상화 기능을 이용하여 일반적인 Windows 운영 영역과 격리된 보안 영역을 구성하는 기술이다.

Windows의 여러 보안 기능이 이러한 가상화 기반 환경을 이용하기 때문에 VBS를 비활성화하면 VBS에 의존하는 보안 기능 역시 영향을 받을 수 있다.


### 6.2 메모리 무결성

메모리 무결성은 Windows 보안의 코어 격리에 포함된 기능으로, 가상화 기반 보안을 이용하여 커널 영역을 보호한다.

악의적인 코드나 신뢰할 수 없는 드라이버가 Windows 커널에 접근하는 것을 어렵게 만드는 역할을 하므로, 해당 기능을 비활성화하면 이러한 보호 기능을 사용할 수 없게 된다.


### 6.3 Windows Hello 관련 설정

원인 분석 과정에서는 Device Guard 하위의 Windows Hello 관련 설정 역시 Hypervisor 활성화에 영향을 주는 것을 확인하였다.

해당 설정을 변경하면 Windows Hello와 관련된 보안 기능에 영향을 줄 가능성이 있으므로, 단순히 Hypervisor를 비활성화하기 위한 목적으로 무조건 적용하는 것은 적절하지 않다.


### 6.4 실습 환경에서의 적용

이번 설정 변경은 GNS3 VM에서 중첩 가상화를 사용하기 위한 개인 실습 환경을 기준으로 진행하였다.

VBS와 메모리 무결성 등의 기능을 비활성화하면 VMware의 중첩 가상화를 사용할 수 있는 대신 Windows에서 제공하는 일부 보안 기능을 사용할 수 없게 될 수 있다.

따라서 모든 Windows 11 환경에 동일한 설정을 적용하기보다는 현재 Hypervisor 동작 상태와 필요한 보안 기능을 먼저 확인하고, 실습 목적과 보안상의 영향을 고려하여 필요한 설정만 변경하는 것이 적절하다.


## 7. 마치며

이번 GNS3 VM 구축 과정에서는 VMware Workstation 설치 전 발생한 `vmrun` 관련 오류와 Windows Hypervisor로 인한 중첩 가상화 문제가 연이어 발생했다.

단순히 Hyper-V 기능을 비활성화하는 것만으로 해결되지 않았으며, VBS, 메모리 무결성, Device Guard 등 Windows의 가상화 기반 보안 기능까지 함께 확인해야 했다.

문제 해결 이후 변경했던 설정을 하나씩 원복하면서 Hypervisor가 다시 활성화되는지 확인했고, 이를 통해 각 설정이 VMware의 중첩 가상화 환경에 미치는 영향도 확인할 수 있었다.

다만 GNS3 VM을 사용하기 위해 Windows의 여러 보안 기능을 비활성화해야 한다는 점은 아쉬웠다.

이전에 실무에서 Hyper-V 기반으로 가상화 환경 구축했던 경험을 생각하면, 실습 목적에 따라 여러 보안 설정을 변경하면서 VMware를 사용하는 것보다 **처음부터 Hyper-V 기반으로 환경을 구성하는 편이 더 나을 수도 있겠다는 생각이 들었다.**

최종적으로 VMware의 중첩 가상화와 GNS3 VM의 **KVM 지원이 정상적으로 동작하는 것**을 확인하며 구축을 마무리하였다.
