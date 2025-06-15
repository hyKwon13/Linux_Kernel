# ARM64 리눅스 커널 빌드 & QEMU + GDB + ftrace 디버깅 가이드

Ubuntu 가상머신에서 ARM64 리눅스 커널을 직접 빌드하고,  
QEMU 및 GDB로 디버깅하며, ftrace로 내부 함수 호출까지 추적하는 전체 과정을 정리하였다.

---

## 1. 리눅스 커널 소스코드 다운로드 및 컴파일

Ubuntu 가상머신에 커널 빌드를 위한 환경을 먼저 구축한다:

```bash
# 필수 패키지 설치
sudo apt update
sudo apt install build-essential libncurses-dev bison flex libssl-dev libelf-dev bc -y
```

리눅스 커널 소스를 클론하고 디렉터리로 이동한다:

```bash
git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
cd linux
```

기본 config 불러오기:

```bash
make defconfig
# 또는 ARM64용이라면
make ARCH=arm64 defconfig
```

커널 컴파일:

```bash
make -j$(nproc)
# 또는 ARM64용이라면
make ARCH=arm64 -j$(nproc)
```

→ `arch/arm64/boot/Image` 파일이 생성된다.  
이 파일이 QEMU에서 사용할 부팅 가능한 커널 이미지다.

---

## 2. QEMU + GDB 설정 (디버깅용)

QEMU 설치
```bash
sudo apt install qemu-system-arm qemu-system-misc qemu-system-aarch64
```

QEMU가 설치되어 있는지 확인한다:

```bash
which qemu-system-aarch64
qemu-system-aarch64 --version
```

ARM64 가상머신을 QEMU로 부팅한다:

```bash
qemu-system-aarch64 \
  -machine virt \
  -cpu cortex-a53 \
  -nographic \
  -m 1024 \
  -kernel ./arch/arm64/boot/Image \
  -append "console=ttyAMA0" \
  -s -S
```

- `-s`: GDB 서버를 열어 포트 1234에서 대기  
- `-S`: QEMU를 pause 상태로 시작하여 GDB 연결을 대기한다

GDB 설치
```bash
sudo apt install gdb-multiarch
```

다른 터미널에서 GDB 실행:

```bash
gdb-multiarch vmlinux
```

GDB 명령어 입력:

```
(gdb) set architecture aarch64
(gdb) target remote :1234
(gdb) b start_kernel
(gdb) c
```

---

## 3. ftrace 활성화

ftrace는 커널 함수 호출을 실시간으로 추적할 수 있는 강력한 도구다.

```bash
sudo -i  # 루트 계정으로 전환
cd /sys/kernel/debug/tracing
echo function > current_tracer
cat trace
```

> `cat trace`는 커널의 거의 모든 함수 호출을 출력하므로, 출력량이 많을 수 있다.

트레이싱 중단:

```bash
echo nop > current_tracer
```

---

## TIP

특정 함수만 추적하려면 다음처럼 한다:

```bash
echo schedule > set_ftrace_filter
echo function > current_tracer
cat trace
```

트레이스를 일시정지하거나 재개하려면:

```bash
echo 0 > tracing_on   # 정지
echo 1 > tracing_on   # 재개
```

---

## 결과

이 과정을 통해 다음을 수행할 수 있다:

- ARM64 커널을 직접 빌드
- QEMU 가상 머신에서 실행
- GDB로 커널 디버깅
- ftrace로 함수 호출 흐름 추적
