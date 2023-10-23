---
LCID: ko-kr
Layout: default
Title: "11: 하드웨어 보안"
Timestamps:
- 2023-07-05
Topic: [ "Development", "Hacking" ]
---

## 컴퓨터의 구조

1. 응용 프로그램(User Space)
    - 모든 프로그램의 실행 환경
    - 일반 사용자와 관리자로 권한 구분(Ring 3)
    - 백신, 실행파일 **무결성 검증** 필요
2. 커널(Kernel Space)
    - 운영체제 핵심 역할 및 시스템 서비스 제공
    - 과거 소프트웨어 플랫폼의 최고 권한(Ring 0)
    - a.k.a VMM(Virtual Machine Monitor)
    - 커널 및 드라이버 **무결성 검증**(신뢰 기반) 필요
3. 하이퍼바이저
    - 가상 머신 생성, 관리, 운영 서비스 제공
    - 현재 소프트웨어 플랫폼의 최고 권한(Ring -1)
        - 커널보다 권한이 높아 커널 무결성 검증 가능
    - 하이퍼바이저 **무결성 검증** 필요
4. 펌웨어
    - 단말 하드웨어 초기화 및 단말 부팅 담당
    - 하드웨어 내부의 소프트웨어
    - 펌웨어 **무결성 검증**(신뢰 기반) 필요
5. 하드웨어
    - 단말의 물리적 구성 담당
        - CPU, RAM, mainboard, HDD/SSD, TPM
    - 광학 조사 및 x-ray 투과 등으로 변조 흔적 탐지

1-3: 소프트웨어 플랫폼
4-5: 하드웨어 플랫폼

## 신뢰 체인

>> 선 검증, 후 실행

1. 하드웨어에서부터 User space로 이르기까지 먼저 무결성을 검증하고
2. User space에서 헤드웨어에 이르기까지 실행한다

이러한 방법으로 신뢰 체인(trust chain)을 형성한다.

## 무결성(integrity) 검증 기법

- 코드 서명(code signing):
    - 실행 파일에 디지털 서명을 추가
    - 서명한 이후로 실행 파일이 수정 또는 변조되는 것을 방지
- 무결성 측정(integrity measurement):
    - 실행파일의 측정 값(hash)을 신뢰할 수 있는 저장소에 누적
        - e.g.) TPM
    - 누적된 측정값을 정상 상태의 측정값과 비교하여 잔말 상태가 정상 혹은 비정상인지 확인

## Inget Boot Guard

- CPU와 PCH(Platform Control Hub)를 이용해 부팅 검증
    - Intel ME(Management Engine)에 저장된 key이용
    - 이후에 실행될 Firmware(BIOS)의 서명 검증 수행
- UEFI활용 및 펌웨어의 서명 검증
    - UEFI firmware의 가장 첫 번째 모듈(Initial Boot Block, IBB)의 무결성 검증

## UEFI SecureBoot

- 펌웨어에 내장된 key를 이용하여 서명 검증
    - 3단계 구조(PK, KEK, DB)로 키 관리
    - BIOS 메뉴를 통해 키 추가 및 삭제 가능
        - 주의) BIOS 관리자 패스워드 설정 필요
- bootloader의 서명 확인 및 실행 담당
    - OS 실행에 필요한 첫 번째 모듈인 부트로더의 무결성 검증
    - 이후, 하이퍼바이저 또는 커널의 무결성은 부트로더가 검증

## Microsoft HVCI

- 하이퍼바이저에 내장된 key를 이용하여 서명 검증
    - Intel VT나 AMD-V를 활용하여 커널 및 머널 모듈의 실행 감시
        - 이벤트 탐지 및 하이퍼콜(hypercall)활용
- 커널 및 머널 모듈의 서명 확인 및 실행 담당
    - OS의 핵심 영역인 커널과 커널 모듈의 무결성 검증
    - 이루 runtime에 지속적으로 커널 및 모듈의 변조를 감시하고 이를 방어

## Syscall vs Hypercall

- syscall
    - `int $0x80`
    - `sysenter`
    - `syscall`
- hypercall
    - `vmcall`
    - User Space에서도 Hypervisor을 호출할 수 있다!

## 코드 서명의 장단점

- 장점
    - 검증 절차가 단순 명료하고, 별도의 하드웨어가 필요 없음
- 단점
    - 유효성이 만료된 서명의 처리가 어려움
        - e.g.) 특정 응용프로그램에 취약점이 있어서 신규버전으로 업데이트 및 실행이 되었는지 확인하고 싶다면?
    - 안전한 저장공간의 부재로 서명 검증 결과를 프로그램의 실행과 연결할 수 밖에 없음

## TPM

- 변조 방지(tamper-resistant) 설계가 된 디바이스
    - 프로세서, RAM, ROM, NVRAM 내장
    - CPU와 별개로 자신의 상태를 유지
- 암호 연산 기능과 측정값 누적 기능 제공
    - 측정값은 PCR(Platform Configuration Register)에 저장

- 저장된 측정값을 이용하여 시스템 신뢰성을 판단하는데 사용 가능
    - PCR에 저장된 값을 로컬에서 확인하거나 원격 검증(remote-attestation)을 통해 확인
- PCR의 특정 값을 이용해서 비밀(secret)에 접근 제어 가능
    - Seal명령은 PCR 값을 이용해서 데이터를 암호화
    - Unseal명령은 PCR값을 이용해서 암호화된 데이터를 복호화
    - Microsoft Bitlocker 및 Windows Hello도 TPM 활용

## RTM (Root of Trust for Measurement)

- RTM은 TPM에게 측정값(measurement) 정보를 전송
    - TPM은 PCR에 저장된 이전 값과 함께 신규 측정값을 저장
    - Extend: $PCR_{new} = Hash(PCR_{old} || Measurement_{new})$

## State-Of-The-Art Boot Process

- Intel BootGuard
- Intel Bios Guard

- Secure Boot

- TPM

- OS
    - Disk Encryption
    - Remote Attenstation

## 전원이 켜지면

1. CPU는 BIOS의 코드서명 검사
2. BIOS/UEFI는 BIOS/UEFI, 부트로더의 해쉬를 TPM에 저장
3. BIOS/UEFI는 부트로더의 코드서명 검사
4. Bootloader는 커널의 해쉬를 TPM에 저장
5. 부트로더는 커널의 코드서명 검사
6. Remote Attestation Server는 TPM에게 요청하여 무결성이 유지되었는지 검사

## OS에 따른 최신 보안기술

응용프로그램: CFI(Control-Flow Integrity)
커널: 응용프로그램 서명 검증, CFI(Control-Flow Integrity)
Hypervisor: VBS(Virtualization-Based Security) (ms-windows-only)

## 경량 가상화 기반 운영체제 보호 기술

- Intel: Light-Box(Lightweight Hypervisor)
    - Host(Ring-1)
        - Shadow-Watcher(Monitor)
            - Guest에 대한 권한을 가진다
        - Shared Kernel (RW)
    - Guest(Ring 0~3)
        - Host를 향한 접근은 차단된다
        - Shared Kernel (R-)

- ARM: TrustZone -> Light-Box(Trusted App & Trusted Kernel)
    - Secure 영역과 다른 영역을 완전히 분리
        1. User Application은 일반 커널과 통신한다
        2. 일반 커널은 Trusted Kernel(OP-TEE)에 SMC call을 보낸다
        3. Trusted Kernel은 Shadow-Watcher를 통해 User Application을 모니터링한다
        4. 일정 기간동안 SMC call이 발생하지 않으면, Trusted Kernel은 일반 커널이 점거되었다고 간주한다

## Kernel mode heap 할당

Kernel모드에서는 메모리가 할당될때 페이지 단위로 할당된다

special cache가 있는 이유는 권한 관련된 오브젝트들이 오버플로우가 나면 권한상승이 되기 때문에, 이러한 취약점을 방지하기 위해 따로 일반 캐쉬로 페이지를 저장한다.

freelist 참조
freelist에 유휴 공간 없다면 page의 공간을 freelist로 이동
page에 유휴 공간 없다면 partial page의 공간을 freelist로 이동

## Scheduler

FCFS: First Come First Serve
SJF: Shortest Job First
    - Starvation Effect
Priority:
    - Every process has a priority
    - Aging: The longer process is pending, the higher priority

## Scheduler Option

- CONFIG_PREEMPT_VOLUNTARY
- It is default configuration for desktop
- This configuration provides preemption point in kernel code
- If the kernel code meets 

- CONFIG_PREEMPT
- It is default configuration for Android
- This option recudes the latency of the kernel by makign all kernel code preemptible
- Except for the case when the kernel code is in the critical section