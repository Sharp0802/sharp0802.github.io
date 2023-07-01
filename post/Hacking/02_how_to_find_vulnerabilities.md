---
LCID: ko-kr
Layout: default
Title: "02: 어떻게 취약점을 찾는가"
Timestamps:
- 2023-07-01
Topic: [ "Hacking" ]
---

# Code Auditing

- 소스코드를 직접 눈으로 보고 정적으로 취약점을 찾는 것    
- Twice the Bits, Twice the Trouble: Vulnerabilities Induced by Migrating to 64-bit platforms
    - https://intellisec.de/pubs/2016-ccs.pdf
    - 32bit 플랫폼에서 잘 돌아가는 코드가 64bit플랫폼에서 취약해지는 것에 대한 연구
    - 플랫폼 사이에서 변화하는 data model의 영향을 시스템적으로 분석
    - Contributions
        - 64bit migration으로 인한 취약점
            - C/C++ 코드를 64bit data model로 이식할 때의 취약점을 시스템적으로 연구하고 정의
        - Empirical study (경험적 연구)
            - Debian stable linux와 같이 잘 알려진 소프트웨어를 대상으로 64bit 문제를 확장하여 연구
        - Case-studies
            - 새롭게 발견한 6개의 취약점과, 기존의 2개의 취약점에 대해서 설명
    - 해볼 점
        - 나만 알고 있는 듯한 것들(e.g. 컴퍼런스에서 발표되지 않은 사실이나, 학회에 발표되자 않은, 논문으로 발표되지 않은 것들)을 논문으로 써보자
    - 배경지식
        - 32bit, 64bit 플랫폼의 차이
        - LLP64(windows), LP64(linux)모델의 차이
        - Integer Truncations 관련 취약점
            - Integer overflow
            ```c
            size_t x = attacker_controlled();
            unsigned int y = x;
            char *buf = malloc(y);
            memcpy(buf, src, x);
            ```
            - Integer signess issue
            ```c
            int x = attacker_controlled();
            unsigned short BUF_SIZE = 10
            if (x >= BUF_SIZE)
                return;
            memcpy(buff, src, x);
            ```
    - 따라서, Integer Trncations에 따라 64bit로의 migration이 프로그램을 취약하게 만들 수 있다!
        - 다음의 예제에서, len이 음수일 경우, size_t가 unsigned int와 크기가 다른 64bit환경에서 buffer-overflow가 발생한다.
        ```c
        int len = attacker_controlled();
        char *buf = malloc((unsigned) len);
        memcpy(buf, src, len);
        ```
    - 이러한 결함은 마이그레이션을 예상하지 않고 개발한 개발자가 원인이다.
    - 또한, 이러한 truncation의 대상이 pointer인 경우 또다른 취약점을 발생시킬 수 있다.
    
# Fuzzing

취약점을 자동으로 찾는 도구 또는 테스팅.

자동화 또는 반자동화된 소프트웨어 테스트 기법으로서, 컴퓨터 프로그램에 유효한, 예상치 않은 또는 무작위 데이터를 입력하는 것이다. 이후 프로그램은 충돌이나 빌트인 코드 검증의 실패, 잠재적인 메모리 누수 발견등 같은 예외에 대한 감시가 이루어진다. 

1. 무작의 입력을 대입한다.
2. Crash가 발생하는지 모니터링한다.
3. Crash가 발생한 경우, 해당 입력값을 저장한다.
4. 새로운 입력값을 생성한다.
5. 1번으로 돌아간다.
