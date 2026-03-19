# Syzagent

`Syzagent`는 `source/syzdirect/Runner/run_hunt.py`를 중심으로, 신규 Linux kernel CVE를 바로 넣어서 SyzDirect hunting pipeline을 돌릴 수 있게 정리한 최소 실행 번들입니다.

핵심 목적은 하나입니다.

- 논문이나 공식 repo의 고정 `dataset.xlsx` 샘플만 재현하는 것이 아니라
- 새로운 CVE를 넣었을 때도
- source 준비, bitcode 빌드, interface 분석, target point 분석, distance instrumentation, fuzzing까지
- 가능한 한 자동으로 이어지게 만드는 것

이 저장소는 원본 SyzDirect 전체를 보존하려는 프로젝트가 아닙니다. `run_hunt.py`가 실제로 동작하는 데 필요한 구성만 남기고, 신규 CVE 자동화에서 자주 깨지는 지점을 runner 쪽에서 방어적으로 흡수하는 데 초점을 둡니다.

## 이 프로젝트가 정확히 무엇인가

원본 SyzDirect는 크게 다음 구성으로 나뉩니다.

- `syzdirect_function_model`
  커널 interface 추출
- `syzdirect_kernel_analysis`
  target point 분석
- `syzdirect_fuzzer`
  Syzkaller 기반 퍼징 실행부

`run_hunt.py`는 이 조각들을 하나의 실행 흐름으로 묶는 래퍼입니다. `Syzagent`는 이 래퍼를 중심으로 다음을 가능하게 하려는 저장소입니다.

- CVE만 주고 필요한 메타데이터를 자동 추론
- 중간 stage부터 재시작
- 퍼징 전에 필요한 입력물 자동 복구
- syscall callfile을 LLM과 heuristic으로 보강
- 퍼징 도중 상태를 보고 멈췄다가 더 좋은 템플릿으로 다시 돌리는 agent loop 운영

즉, `Syzagent`는 다음 성격을 가집니다.

- 공식 SyzDirect 바이너리를 감싸는 오케스트레이션 레이어
- CVE 중심의 실행기
- 재시작 가능한 stage 기반 파이프라인
- LLM 기반 syscall 추론과 agent 기반 템플릿 강화 실험 지점

반대로 다음은 아닙니다.

- 원본 연구 repo 전체
- 완전히 제품화된 범용 퍼징 프레임워크
- Syzkaller 자체를 대체하는 도구

## 어떤 파일이 들어 있는가

이 저장소에는 `run_hunt.py`를 돌리는 데 필요한 핵심 파일만 들어 있습니다.

- `source/syzdirect/Runner/`
  - `run_hunt.py`
  - `Fuzz.py`
  - `Config.py`
  - `cve_resolver.py`
  - `SyscallAnalyze/`
- `source/syzdirect/kcov.diff`
- `source/syzdirect/bigconfig`
- `source/syzdirect/template_config`
- `source/syzdirect/syzdirect_fuzzer/`
  - `bin/`
  - `executor/`
  - `sys/`
  - `.descriptions`
- `source/syzdirect/syzdirect_function_model/build/lib/`
  - `interface_generator`
  - `configs/`
- `source/syzdirect/syzdirect_kernel_analysis/build/lib/`
  - `target_analyzer`
  - `configs/`

의도적으로 제외한 것:

- 기존 dataset
- 예전 실험 결과물
- 대용량 workdir
- 부가 메모나 임시 디버깅 파일

## 전체 구조 한눈에 보기

`run_hunt.py`의 신규 CVE 파이프라인은 다음 6단계입니다.

```text
source -> bitcode -> analyze -> target -> distance -> fuzz
```

조금 더 운영 관점에서 보면 다음 그림으로 이해하면 됩니다.

```text
            +-----------------------+
            |   CVE / commit /      |
            |  function / file      |
            +-----------+-----------+
                        |
                        v
            +-----------------------+
            |  source 준비          |
            |  - linux checkout     |
            |  - commit 검증/fetch  |
            |  - kcov patch         |
            +-----------+-----------+
                        |
                        v
            +-----------------------+
            |  bitcode build        |
            |  - emit-llvm wrapper  |
            |  - kernel build       |
            |  - .llbc 생성         |
            +-----------+-----------+
                        |
                        v
            +-----------------------+
            |  interface analyze    |
            |  - interface_generator|
            |  - k2s mapping        |
            |  - bad llbc 격리      |
            +-----------+-----------+
                        |
                        v
            +-----------------------+
            |  target analyze       |
            |  - target point       |
            |  - tfinfo 생성/보정   |
            |  - callfile 생성      |
            +-----------+-----------+
                        |
                        v
            +-----------------------+
            | distance instrument   |
            | - instrumented kernel |
            | - bzImage / vmlinux   |
            +-----------+-----------+
                        |
                        v
            +-----------------------+
            | fuzz / agent loop     |
            | - syz-manager         |
            | - metrics/log 분석    |
            | - callfile 강화       |
            +-----------------------+
```

## 왜 이런 래퍼가 필요한가

신규 CVE를 넣어 자동화하려고 하면 보통 아래 같은 지점에서 자주 깨집니다.

- 로컬 repo에 commit object가 없는데 바로 `git checkout`을 시도
- 최신 kernel bitcode에서 `interface_generator`가 특정 `.llbc` 때문에 죽음
- `template_config`가 빠져 있어 fuzz 단계 직전에 깨짐
- `target_functions_info.txt`가 비어서 fuzz가 사실상 no-op가 됨
- callfile이 비거나 syzkaller syscall 이름과 맞지 않아 target call이 disable됨
- 퍼징은 돌지만 coverage가 늘지 않아 사실상 잘못된 syscall chain으로 시간을 낭비

이 저장소는 이런 문제를 원본 도구 자체를 크게 바꾸기보다 runner 쪽에서 흡수하는 방향을 택합니다.

## 주요 실행 모드

`run_hunt.py`는 크게 세 가지 방식으로 사용할 수 있습니다.

### 1. `new`

신규 CVE를 위한 메인 모드입니다.

예시:

```bash
cd source/syzdirect/Runner

python3 run_hunt.py -j 8 -uptime 24 new \
  --cve CVE-2026-23126 \
  --commit b97d5eedf4976cc94321243be83b39efe81a0e15 \
  --function nsim_bpf_destroy_prog \
  --file drivers/net/netdevsim/netdevsim.c \
  --agent-rounds 3 \
  --agent-uptime 6
```

이 모드는 다음 두 형태로 쓸 수 있습니다.

- `--commit`, `--function`, `--file`까지 직접 주는 수동 모드
- `--cve`만 주고 일부를 자동 추론하는 반자동 모드

### 2. `dataset`

원본 SyzDirect의 dataset 기반 흐름을 호환용으로 남겨 둔 모드입니다.

예시:

```bash
python3 run_hunt.py dataset -dataset dataset.xlsx -j 8 \
  prepare_for_manual_instrument compile_kernel_bitcode \
  analyze_kernel_syscall extract_syscall_entry \
  instrument_kernel_with_distance fuzz
```

신규 CVE 자동화가 주 목적이라면 보통은 `new` 모드를 쓰면 됩니다.

### 3. `fuzz`

이미 준비된 target에 대해 fuzz만 다시 돌리고 싶을 때 쓰는 모드입니다.

예시:

```bash
python3 run_hunt.py fuzz --targets 0 -uptime 12 --agent-rounds 5
```

이 모드는 source, bitcode, analysis를 다시 하지 않고 퍼징 쪽 반복 실험만 하고 싶을 때 유용합니다.

## 신규 CVE 파이프라인 상세 설명

### 1단계: `source`

목표 commit 기준으로 Linux source tree를 준비합니다.

무엇을 하나:

- 로컬 linux template repo가 있으면 거기서 clone
- 없으면 `torvalds/linux`를 clone
- 요청한 commit이 로컬 object로 존재하는지 먼저 검증
- 없으면 fetch 시도
- 그래도 없으면 stage1에서 명확하게 실패
- checkout 후 `kcov.diff` 적용
- 자동 빌드에 필요한 완화 처리 수행

왜 중요한가:

신규 CVE 자동화에서 자주 보는 오류가 다음입니다.

```text
fatal: reference is not a tree
fatal: pathspec '...' did not match any file(s) known to git
```

원인은 checkout 자체가 아니라 "그 commit object를 로컬이 아직 모른다"는 점입니다. 이 runner는 checkout 전에 commit 존재를 확인하고, 필요하면 fetch를 먼저 시도합니다.

### 2단계: `bitcode`

LLVM bitcode를 뽑을 수 있도록 kernel을 빌드합니다.

무엇을 하나:

- `emit-llvm.sh` wrapper 생성
- kernel config 복사, 일반적으로 `bigconfig`
- 필요한 빌드 옵션 추가
- `olddefconfig`
- 실제 kernel build
- `.llbc` 산출물 생성

핵심 산출물:

- `arch/x86/boot/bzImage`
- 각 object에 대응되는 `.llbc`

### 3단계: `analyze`

`interface_generator`를 통해 kernel interface를 추출합니다.

무엇을 하나:

- syzkaller feature signature 생성
- `interface_generator` 실행
- kernel-to-syzkaller 매핑 파일 생성

이 단계는 최신 kernel에서 특히 잘 깨집니다. 대표적으로:

- `Invalid value reference from metadata`
- 특정 `.llbc` 로딩 실패
- 특정 `.llbc` 처리 중 segfault

그래서 runner는 다음 방어 로직을 추가합니다.

- 실패 로그에서 문제 `.llbc` 후보를 추출
- 해당 파일을 quarantine 디렉터리로 이동
- quarantine 위치는 analysis tree 바깥이라 다음 스캔에 다시 안 잡힘
- skip manifest 저장
- 재실행 시 이전에 격리한 파일을 다시 건드리지 않음

즉, "한 개의 나쁜 bitcode 파일 때문에 전체 분석이 매번 죽는 상황"을 완화합니다.

### 4단계: `target`

취약 함수까지 도달하기 위한 target point 분석과 fuzz 입력 생성 단계입니다.

무엇을 하나:

- target function을 positive point로 기록
- SyzDirect target relation map 생성
- `target_analyzer` 실행
- `target_functions_info.txt` 생성 또는 보정
- `PrepareForFuzzing` 호출
- 결과가 비면 fallback callfile 생성

이 단계에서 흔한 함정:

- 로그상으로는 끝난 것처럼 보이는데
- `target_functions_info.txt`가 비어 있음
- callfile도 비어 있거나 없음
- fuzz/agent loop는 돌지만 실제 target syscall은 한 번도 안 탐

이 저장소의 runner는 이 문제를 막기 위해:

- `tfinfo`가 비면 최소 `0 <function> <file>` 엔트리 강제 생성
- callfile이 없거나 비면 자동 생성 또는 재생성
- xidx 0 callfile을 다른 xidx에 복제하는 fallback 제공
- fuzz 직전 필수 입력물 검증 수행

### 5단계: `distance`

distance instrumentation이 들어간 kernel을 다시 빌드합니다.

무엇을 하나:

- target function index 결정
- distance directory 탐색
- `Makefile.kcov` 작성
- instrumented kernel rebuild
- `bzImage`, `vmlinux` 복사

이 단계 결과물이 실제 fuzz 단계의 kernel 이미지가 됩니다.

### 6단계: `fuzz`

최종적으로 `syz-manager`를 이용해 퍼징을 시작합니다.

무엇을 하나:

- fuzz 입력물 검증
- syscall 이름 정규화
- manager config 생성
- instrumented kernel로 VM 부팅
- `syz-manager` 실행

추가로 현재 runner는 일반 fuzz 모드에서 HTTP 포트를 동적으로 할당합니다. 그래서 여러 CVE를 연속 또는 병렬로 확인할 때 `address already in use` 충돌이 줄어듭니다.

## CVE를 어떻게 자동으로 가져오는가

`new` 모드에서 `--commit`, `--function`, `--file` 중 하나라도 빠지면 `cve_resolver.py`가 동작합니다.

resolver는 fix commit을 다음 순서로 찾습니다.

1. Linux kernel CVE list
   `git.kernel.org/pub/scm/linux/security/vulns.git`
2. NVD API
   `services.nvd.nist.gov`
3. GitHub commit search
   `api.github.com`

그 다음 다음 순서로 메타데이터를 만듭니다.

1. fix commit patch를 가져옴
2. patch에서 가장 유력한 `.c` 파일을 찾음
3. hunk header를 보고 가장 유력한 함수명을 추출
4. 기본적으로 `fix_commit~1`을 vulnerable commit으로 사용

왜 `Fixes:` 태그 대신 `fix~1`을 쓰는가:

- `Fixes:`는 원래 버그를 도입한 아주 오래된 commit을 가리키는 경우가 많음
- shallow fetch나 빠른 실험 환경에서 가져오기 어려울 수 있음
- `fix~1`은 가장 최근의 vulnerable state라 one-day reproduction에 더 실용적임

### CVE만 넣고 자동 추론하는 예시

```bash
python3 run_hunt.py -j 8 -uptime 24 new --cve CVE-2025-99999
```

이 경우 runner는 가능한 범위에서 다음을 자동으로 추론합니다.

- checkout commit
- target function
- target file

## pre-fix 재현과 post-fix 검증

기본 `new --cve ...`는 vulnerable kernel 기준으로 돌립니다.

- checkout 대상은 기본적으로 `fix_commit~1`

반대로 `--verify-patch`를 주면 fixed kernel 기준 검증 모드가 됩니다.

- checkout 대상은 실제 fix commit

예시:

```bash
python3 run_hunt.py new \
  --cve CVE-2025-99999 \
  --verify-patch
```

이 모드는 "패치 후에는 정말로 재현이 사라지는가"를 빠르게 검증하고 싶을 때 유용합니다.

## LLM call을 어떻게 활용하는가

신규 CVE hunting에서 가장 어려운 부분 중 하나는 "어떤 syscall 조합이 이 함수까지 가는가"입니다. target function을 안다고 해서 바로 좋은 callfile이 나오는 것은 아닙니다.

현재 runner는 이 부분에 두 개의 레이어를 둡니다.

### 1. heuristic syscall 추정

`guess_syscalls(file_path)`는 source file path를 보고 대략적인 syscall family를 고릅니다.

예:

- `net/vmw_vsock/...` -> `connect$vsock_stream`
- `kernel/bpf/...` -> `bpf$PROG_LOAD`
- `fs/...` -> `openat`

장점:

- 빠름
- deterministic
- LLM이 없어도 동작

단점:

- coarse-grained
- 세부 variant를 잘못 고를 수 있음

### 2. LLM 기반 syscall 제안

`llm_analyze_cve()`는 다음 정보를 묶어 prompt를 만듭니다.

- CVE ID
- kernel commit
- target function
- file path

그리고 로컬 `claude` CLI에 JSON만 반환하도록 요청합니다. 기대 형식은 다음과 같습니다.

```json
{
  "syscalls": [
    {
      "Target": "name$variant",
      "Relate": ["setup1", "setup2"]
    }
  ]
}
```

의도는 다음과 같습니다.

- 1~3개의 핵심 target syscall 제안
- 각 target에 대해 3~6개의 관련 setup syscall 제안
- syzkaller naming을 정확히 맞춘 결과 생성

만약 LLM 호출이 실패하거나:

- timeout
- JSON 파싱 실패
- CLI 미설치
- 비정상 종료

이 경우 runner는 자동으로 heuristic fallback을 사용합니다.

즉, LLM은 있으면 더 좋고, 없어도 전체 파이프라인이 멈추지는 않게 설계합니다.

## syscall name normalization이 왜 필요한가

LLM이든 heuristic이든 대략 맞는 방향을 잡아도 syzkaller가 실제로 아는 syscall 이름과 조금만 어긋나면 target call은 바로 unusable 상태가 됩니다.

예를 들어 다음과 같은 문제가 생길 수 있습니다.

- `io_uring_register`
- `connect$vsock`
- `bind$vsock`
- `connect$mptcp`
- `setsockopt$mptcp`

이 이름들이 사람이 보기엔 맞아 보여도, syzkaller descriptions 기준으로는 정확한 variant가 아니어서 `unknown input call`로 빠질 수 있습니다.

그래서 runner는 fuzz 직전에 syscall 이름을 실제 syzkaller 정의 기준으로 정규화합니다. 이 단계는 다음 문제를 막기 위해 매우 중요합니다.

- target call이 disable되었는데 로그만 대충 움직이는 상황
- fuzz는 도는 것처럼 보이지만 실제 목표 syscall은 한 번도 호출되지 않는 상황

## 중간에 멈추고 다시 시작하는 방법

이 프로젝트는 한 번에 끝까지 달리는 스크립트가 아니라, stage 기반으로 끊어서 운영하는 것을 전제로 합니다.

### `--from-stage`

아래처럼 특정 stage부터 재시작할 수 있습니다.

```bash
python3 run_hunt.py new \
  --cve CVE-2025-99999 \
  --commit abc123 \
  --function vuln_func \
  --file net/core/sock.c \
  --from-stage distance
```

가능한 stage:

- `source`
- `bitcode`
- `analyze`
- `target`
- `distance`
- `fuzz`

이 기능이 유용한 상황:

- source checkout과 build는 이미 끝났음
- target 분석만 다시 하고 싶음
- distance kernel은 이미 있고 fuzz만 다시 하고 싶음
- callfile을 수동 수정한 뒤 fuzz stage만 반복하고 싶음

## state 파일과 중간 상태 추적

`new` 모드 실행 시 state manifest가 다음 위치에 남습니다.

```text
<workdir>/state/case_0.json
```

여기에는 다음 정보가 들어갑니다.

- CVE
- commit
- function
- file
- config
- last_stage
- phase
- 주요 artifact 경로
- 마지막 error 메시지

대표적인 `phase` 값:

- `initialized`
- `running`
- `completed`
- `failed`

이 파일은 다음 용도로 쓸 수 있습니다.

- 어떤 stage에서 멈췄는지 확인
- 외부 orchestration에서 진행률 추적
- 장시간 fuzz가 실패했을 때 원인 기록 확인
- 사람이 수동介入하기 전에 현재 상태 파악

## 퍼징 중간에 멈춰서 강화하는 방법: Agent Loop

이 저장소에서 중요한 차별점 중 하나가 agent loop입니다.

기본 퍼징은 단순합니다.

```text
fuzz -> 끝
```

agent loop를 켜면 흐름이 다음처럼 바뀝니다.

```text
fuzz -> health 평가 -> triage -> callfile 강화 -> 다시 fuzz
```

사용 예시:

```bash
python3 run_hunt.py new \
  --cve CVE-2025-99999 \
  --commit abc123 \
  --function vuln_func \
  --file net/core/sock.c \
  --agent-rounds 3 \
  --agent-window 300 \
  --agent-uptime 6
```

의미:

- 최대 3라운드 반복
- 각 라운드는 6시간 퍼징
- 라운드 사이에 health 평가와 callfile 강화 수행

### 라운드 내부 흐름

각 라운드는 다음 순서로 진행됩니다.

1. 현재 callfile로 fuzz
2. crash가 났는지 확인
3. metrics와 manager log를 읽어 health 평가
4. 실패 유형을 R1/R2/R3로 분류
5. 분류에 맞는 agent를 호출해 callfile 강화
6. 강화된 callfile로 다음 라운드 진행

### health는 무엇으로 평가하는가

현재 agent loop는 대략 다음 지표를 봅니다.

- execution 증가량
- coverage 증가량
- crash 증가량
- `"all target calls are disabled"` 같은 치명 로그

그 결과 상태를 대략 이렇게 분류합니다.

- `healthy`
- `stagnant`
- `fatal`

### R1 / R2 / R3는 무엇인가

#### R1

의미:

- syscall family 자체를 잘못 골랐을 가능성
- target call이 실제로 runnable하지 않음
- 아무것도 제대로 실행되지 않음

징후:

- 실행 수 증가가 거의 없음
- target call disabled

대응:

- 다른 related syscall 후보를 더 넣거나, target syscall 후보 자체를 재구성

#### R2

의미:

- syscall 방향은 맞을 수 있음
- 하지만 parameter 또는 object generation이 틀림

징후:

- 로그에 `EINVAL`, `EFAULT`가 과도하게 많음

대응:

- 더 적절한 object setup 또는 parameter synthesis 필요

#### R3

의미:

- 실행은 되는데 coverage나 진전이 멈춤
- dependency chain 또는 context syscall이 부족함

징후:

- 실행 수는 올라가는데 coverage가 안 오름

대응:

- 관련 syscall을 더 붙여 call sequence를 풍부하게 만듦

### 어떤 agent를 붙이는가

현재 매핑은 다음과 같습니다.

- `R1` 또는 `R3` -> `RelatedSyscallAgent`
- `R2` -> `ObjectSynthesisAgent`

즉:

- syscall 체인 자체가 부족하거나 틀렸으면 관련 syscall을 늘리는 쪽
- object/parameter가 문제면 object synthesis 쪽

주의:

이 저장소는 `run_hunt.py`에 연결되는 인터페이스를 설명하는 최소 번들이고, 실제 `source/agent/` 구현 전체는 포함하지 않습니다. 더 큰 로컬 환경에서 agent 모듈이 존재할 것을 가정합니다.

## "중간에 멈춰서 퍼징을 강화한다"는 말의 실제 의미

운영적으로는 두 레벨이 있습니다.

### 레벨 1: stage 단위 중단

예:

- `target`까지 돌리고 callfile을 사람이 직접 수정
- 그 뒤 `--from-stage distance` 또는 `--from-stage fuzz`로 재시작

### 레벨 2: fuzz round 단위 중단

예:

- 24시간 한 번에 돌리지 않고
- 4시간 또는 6시간씩 나눠서
- 각 라운드 사이에 결과를 보고 더 나은 syscall template로 교체

이 방식은 신규 CVE hunting에 훨씬 현실적입니다. 처음 만든 callfile이 완벽할 가능성은 낮기 때문입니다.

## 실제로 추가된 방어 로직

이 저장소의 runner는 신규 CVE 자동화를 위해 다음 방어 로직을 추가했습니다.

- checkout 전에 commit 존재 여부 검증
- 없는 commit object는 fetch 후 재검증
- 그래도 없으면 stage1에서 즉시 명확하게 실패
- `interface_generator` 실패 시 문제 `.llbc` quarantine
- quarantine 파일 재스캔 방지 manifest 유지
- `template_config` 자동 보장
- 비어 있는 `target_functions_info.txt` 자동 복구
- callfile이 없거나 비면 재생성
- 필요 시 xidx 0 callfile을 다른 xidx에 복제
- fuzz 전 필수 입력물 검증
- syscall 이름 normalization
- 일반 fuzz 모드에서 HTTP 포트 자동 할당
- `state/case_0.json`에 진행 상태와 오류 저장

## 실제 실행 예시

다음은 실제로 확인한 `fuzz` stage 실행 예시입니다.

명령:

```bash
timeout 120 python3 run_hunt.py -j 1 -uptime 1 -fuzz-rounds 1 -workdir ./workdir new \
  --cve CVE-2026-23126 \
  --commit b97d5eedf4976cc94321243be83b39efe81a0e15 \
  --function nsim_bpf_destroy_prog \
  --file drivers/net/netdevsim/netdevsim.c \
  --from-stage fuzz
```

확인된 핵심 로그:

```text
serving http on http://0.0.0.0:2346
serving rpc on tcp://[::]:35303
booting test machines...
wait for the connection from test machine...
VMs 1, executed 8, cover 293, signal 363/0, crashes 0, repro 0
VMs 1, executed 363, cover 830, signal 957/1493, crashes 0, repro 0
VMs 1, executed 2386, cover 831, signal 960/1849, crashes 0, repro 0
```

의미:

- manager가 실제로 떴음
- VM 부팅과 guest 연결이 됨
- execution과 coverage가 증가했음
- 최소한 "퍼징이 실제로 돌고 있다"는 점은 확인됨

또 다른 실제 확인 포인트:

- `CVE-2025-21655`는 coverage 증가까지 확인
- `CVE-2025-21756`, `CVE-2025-40257`는 unknown syscall 문제는 줄었지만 일부는 `got no coverage`로 멈춤

이것은 runner 인프라가 망가졌다는 뜻이 아니라, 각 CVE별 callfile 품질이나 kernel/runtime 상태가 여전히 병목일 수 있음을 의미합니다.

## 병렬 실행과 포트 문제

퍼징을 여러 번 돌리다 보면 흔히 다음 문제가 납니다.

```text
bind: address already in use
```

원인은 `syz-manager` HTTP 포트를 고정으로 쓰기 때문입니다.

이 runner는 일반 fuzz 모드에서 빈 TCP 포트를 자동으로 잡아 config에 넣습니다. 따라서 여러 CVE를 연속으로 검증하거나 동시에 띄울 때 충돌 가능성이 줄어듭니다.

## 추천 운영 시나리오

### 시나리오 1: CVE만 먼저 넣어보기

```bash
python3 run_hunt.py -j 8 -uptime 24 new --cve CVE-2025-99999
```

추천 상황:

- 우선 자동 추론이 얼마나 되는지 보고 싶을 때
- commit/function/file을 아직 사람이 정리하지 않았을 때

### 시나리오 2: 메타데이터는 수동으로 고정

```bash
python3 run_hunt.py -j 8 -uptime 24 new \
  --cve CVE-2025-99999 \
  --commit abc123 \
  --function vuln_func \
  --file net/core/sock.c
```

추천 상황:

- resolver가 여러 파일 중 잘못 고를 수 있을 때
- function/file을 이미 사람이 더 정확히 알고 있을 때

### 시나리오 3: analysis 이후부터 반복

```bash
python3 run_hunt.py -j 8 new \
  --cve CVE-2025-99999 \
  --commit abc123 \
  --function vuln_func \
  --file net/core/sock.c \
  --from-stage target
```

추천 상황:

- source와 bitcode는 이미 있음
- callfile 생성 전략만 여러 번 바꿔보고 싶음

### 시나리오 4: agent loop로 템플릿 강화

```bash
python3 run_hunt.py -j 8 -uptime 6 new \
  --cve CVE-2025-99999 \
  --commit abc123 \
  --function vuln_func \
  --file net/core/sock.c \
  --agent-rounds 4 \
  --agent-window 300 \
  --agent-uptime 6
```

추천 상황:

- 처음 만든 callfile이 부정확할 가능성이 높음
- 한 번에 24시간 돌리는 것보다 짧은 라운드로 조정하고 싶음

## 필요한 실행 환경

최소한 다음은 필요합니다.

- Python 3
- Clang/LLVM 기반 kernel build 환경
- VM image
- guest 접속용 SSH key
- QEMU/KVM 기반 실행 환경

유용한 환경변수:

```bash
export SYZDIRECT_RUNTIME=/path/to/runtime
export SYZDIRECT_VM_IMAGE=/path/to/stretch.img
export SYZDIRECT_SSH_KEY=/path/to/stretch.id_rsa
```

상황에 따라 추가로 필요할 수 있는 것:

- 빠른 clone을 위한 로컬 Linux template repo
- LLM 기반 syscall 추론을 원할 경우 `claude` CLI
- CVE 자동 해석을 위한 네트워크 접근

## 빠른 시작

```bash
git clone https://github.com/whysocscs/Syzagent.git
cd Syzagent/source/syzdirect/Runner

export SYZDIRECT_RUNTIME=/path/to/runtime
export SYZDIRECT_VM_IMAGE=/path/to/stretch.img
export SYZDIRECT_SSH_KEY=/path/to/stretch.id_rsa

python3 run_hunt.py -j 8 -uptime 24 new \
  --cve CVE-2026-23126 \
  --commit b97d5eedf4976cc94321243be83b39efe81a0e15 \
  --function nsim_bpf_destroy_prog \
  --file drivers/net/netdevsim/netdevsim.c
```

이미 산출물이 있어서 fuzz만 다시 들어가고 싶다면:

```bash
python3 run_hunt.py -j 8 -uptime 12 new \
  --cve CVE-2026-23126 \
  --commit b97d5eedf4976cc94321243be83b39efe81a0e15 \
  --function nsim_bpf_destroy_prog \
  --file drivers/net/netdevsim/netdevsim.c \
  --from-stage fuzz
```

## 현재 한계

미리 알고 있어야 할 제약도 있습니다.

- 이 저장소는 SyzDirect 바이너리를 최소 번들 형태로 싣고 있으며, 모든 것을 source부터 다시 빌드하는 구조는 아님
- CVE auto-resolve는 best-effort이므로 일부 CVE는 `--commit`, `--function`, `--file`을 수동 지정해야 함
- LLM syscall 추론의 품질은 로컬에서 사용하는 모델/CLI에 영향받음
- 일부 kernel은 여전히 low-level bitcode 문제나 runtime coverage 문제를 일으킬 수 있음
- agent loop는 외부 `source/agent/` 구현을 가정함
- runner 인프라가 안정화되어도, 각 CVE의 callfile 품질 자체는 여전히 별도 개선 대상일 수 있음

## 마지막으로

이 저장소를 한 문장으로 요약하면 다음과 같습니다.

`Syzagent`는 신규 Linux kernel CVE를 입력으로 받아, SyzDirect의 공식 구성요소를 이용해 재현 가능한 hunting pipeline을 자동으로 돌리고, 필요하면 LLM과 agent loop를 이용해 퍼징 입력을 중간에 강화할 수 있도록 만든 CVE 지향 래퍼입니다.
