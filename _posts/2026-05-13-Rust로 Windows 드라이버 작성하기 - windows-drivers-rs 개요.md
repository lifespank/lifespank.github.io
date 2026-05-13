---
title: "Rust로 Windows 드라이버 작성하기 — windows-drivers-rs 개요"
date: 2026-05-13 15:30 +0900
categories: ['윈도우 드라이버']
tags: 윈도우 드라이버
---

### 1. windows-drivers-rs가 무엇인가

**`windows-drivers-rs`(약칭 WDR)는 Microsoft가 직접 운영하는 Rust 크레이트 모음으로, WDM·KMDF·UMDF 드라이버를 Rust로 작성할 수 있게 해 준다.** 저장소는 [github.com/microsoft/windows-drivers-rs](https://github.com/microsoft/windows-drivers-rs)이고, 핵심 크레이트는 모두 `crates.io`에 게시되어 있다. C로 작성된 WDK 헤더를 `bindgen`으로 자동 변환한 raw FFI 계층 위에, 안전한 idiomatic 래퍼를 얹는 두 층 구조다.

README의 한 줄 정의:

> This repo is a collection of Rust crates that enable developers to develop Windows Drivers in Rust. It is the intention to support both WDM and WDF driver development models.

핵심은 두 단어다. **"crate" + "FFI".** Rust 컴파일러가 직접 WDM/WDF를 이해하지는 않는다. 단지 C 헤더에서 추출한 함수 시그니처와 구조체 정의를 Rust로 옮긴 다음, 그 위에 RAII나 타입 안전성 같은 Rust의 도구를 얹어 놓은 것이다. **빌드 결과물은 결국 같은 `.sys` 파일이고**, OS 입장에서는 C로 쓴 드라이버와 구별이 되지 않는다.

> [!NOTE]
> README는 "still in early stages of development and is not yet recommended for production use"라고 명시하고 있다. **양산 드라이버용이 아니라 실험/평가용**이라는 뜻이다. 다만 Microsoft가 직접 유지보수한다는 점에서 외부 크레이트들과는 무게가 다르다.

### 2. 왜 Rust로 드라이버를 짜는가

커널 모드 드라이버에서 자주 터지는 버그 클래스 중 어느 것이 컴파일 타임에 걸리고, 어느 것이 여전히 안 걸리는지를 정직하게 정리해 두는 게 출발점이다.

#### 2-1. Rust가 컴파일 타임에 잡는 것

**객체 수명과 use-after-free.** WDF 디바이스 컨텍스트, IRP 완료 후의 stale buffer 포인터, 디큐된 큐 객체 — 모두 C에서 흔히 터지는 패턴이다. 안전 래퍼로 감싸진 객체에 대해서는 Rust의 borrow checker가 컴파일 타임에 dangling reference를 잡는다. `Box<T>` 안에 들어간 컨텍스트 구조체는 해제 시점이 정해져 있고, `&mut T`는 동시 접근이 컴파일러에 의해 차단된다.

**데이터 경합(data race).** 동일한 가변 데이터를 두 스레드가 잠금 없이 동시에 만지는 패턴 — `Send`/`Sync` 트레이트가 컴파일 타임에 차단한다. C에서는 코드 리뷰와 약속에만 의존하던 부분이다. 단, 이는 어디까지나 **스레드 간 경합** 한정이다. 같은 스레드가 IRQL을 올렸다 내리는 사이에 같은 데이터를 만지는 패턴(예: PASSIVE에서 갱신 중인 자료구조를 DPC 컨텍스트에서 들여다보기)은 `Send`/`Sync`가 모델링하지 못한다. 이 점은 아래 2-3에서 다시 다룬다.

#### 2-2. 부분적으로만 잡는 것

**잠금 디스코드.** `wdk::wdf::SpinLock` 같은 래퍼가 있지만, 현재 구현은 `acquire()` / `release()`를 별개의 메서드로 노출한다(아래 코드 인용 참조). 즉 **"잠금을 잡지 않고 보호 자원에 접근"하는 패턴은 컴파일러가 막아주지 못한다.** Rust 생태계에서 흔한 `parking_lot`/`std::sync::Mutex<T>` 스타일의 "데이터를 잠금 안에 넣어둔다" 패턴은 아직 WDR에 도입되지 않았다. 이 부분은 RAII 가드를 반환하는 형태로 발전해야 할 여지가 크다.

```rust
// crates/wdk/src/wdf/spinlock.rs — 현재 시점의 SpinLock 인터페이스
pub fn acquire(&self) { ... }
pub fn release(&self) { ... }
```

#### 2-3. 여전히 못 잡는 것

**IRQL 위반.** "이 함수는 PASSIVE_LEVEL에서만 호출 가능"이라는 제약은 본질적으로 **런타임 CPU 상태**이다. Rust 타입 시스템이 IRQL을 modelling하려면 typestate 패턴(예: `ZeroMarker<PassiveLevel>` 같은 phantom type)을 광범위하게 도입해야 하는데, WDR은 현재 그렇게까지는 가지 않는다. `wdk-alloc`의 docstring도 정직하게 다음과 같이 쓰여 있다.

```rust
/// # Safety
/// This allocator is only safe to use for allocations happening at `IRQL`
/// <= `DISPATCH_LEVEL`
```

**즉, IRQL 위반은 C 드라이버와 동등하게 런타임에 BSOD로 나타난다.** Rust를 쓴다고 자동으로 사라지는 클래스가 아니다.

> [!IMPORTANT]
> "Rust = 메모리 안전" 슬로건을 커널 모드에 그대로 옮기면 오해가 생긴다. **WDR이 줄여주는 것은 주로 객체 수명·데이터 경합 계열이고, IRQL·DPC 컨텍스트·페이지 가능 메모리(paged memory) 접근 같은 커널 고유 위험은 여전히 사람이 책임진다.** 안전한 부분은 안전한 함수로 노출하고, 위험한 부분은 `unsafe fn`으로 노출해 호출 측에 책임을 넘기는 게 WDR의 일관된 디자인이다.

### 3. 크레이트 구성

WDR은 일곱 개의 크레이트로 이루어진다. 역할이 잘 분리되어 있어서 각자의 책임만 보면 전체 구조가 명확하다.

| 크레이트 | 역할 |
|---|---|
| `wdk-build` | `build.rs`에서 호출. WDK 경로 탐지, `bindgen` 호출, 링커 옵션 설정. 빌드 의존성. |
| `wdk-sys` | `bindgen`이 WDK 헤더로부터 자동 생성한 raw FFI. 거의 모든 항목이 `unsafe`. C 매크로 중 bindgen이 변환하지 못한 것들은 수동 재구현. |
| `wdk` | `wdk-sys` 위에 얹은 safe idiomatic 래퍼. 현재 안전 래퍼로 노출된 객체는 `wdf::SpinLock`과 `wdf::Timer` **두 종류뿐**이다. 그 외에 `println!` 매크로(`DbgPrint` 기반)와 디버거 브레이크용 `dbg_break()` 정도가 들어 있다. 참고로 `paged_code!`는 안전 래퍼가 아니라 `wdk-sys::PAGED_CODE` 매크로의 단순 `pub use` 재-export이다. |
| `wdk-macros` | `call_unsafe_wdf_function_binding!` 같은 매크로 모음. `wdk-sys`가 재-export하므로 직접 의존할 일은 거의 없다. |
| `wdk-alloc` | `#[global_allocator]`로 등록할 `WdkAllocator`. `ExAllocatePool2(POOL_FLAG_NON_PAGED, ...)`를 호출한다. |
| `wdk-panic` | `#[panic_handler]` 기본 구현. **현재는 `loop {}` 무한 루프이다(아래 NOTE 참조).** |
| `cargo-wdk` | **Preview.** `cargo-make`를 대체할 새로운 cargo 확장. README에서 명시적으로 시도해 보길 권한다. |

> [!NOTE]
> `wdk-panic`의 release 빌드 panic handler는 현재 다음과 같다.
> ```rust
> #[panic_handler]
> const fn panic(_info: &PanicInfo) -> ! {
>     loop {}
>     // FIXME: Should this trigger Bugcheck via KeBugCheckEx?
> }
> ```
> `FIXME` 주석이 시사하듯, **release 빌드에서 panic이 발생하면 그 CPU가 무한 루프에 진입한다.** BSOD가 깔끔하게 떨어지는 것이 아니라 시스템이 응답하지 않는 채로 멈춘다는 뜻이다. 양산용으로 쓰려면 `panic_handler`를 직접 구현해 `KeBugCheckEx`를 호출하도록 바꿔야 한다.

`wdk-alloc`의 핵심 한 단락:

```rust
// crates/wdk-alloc/src/lib.rs (발췌)
const RUST_TAG: ULONG = u32::from_ne_bytes(*b"rust");

unsafe impl GlobalAlloc for WdkAllocator {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        let ptr = unsafe {
            ExAllocatePool2(POOL_FLAG_NON_PAGED, layout.size() as SIZE_T, RUST_TAG)
        };
        // ...
    }
    unsafe fn dealloc(&self, ptr: *mut u8, _layout: Layout) {
        unsafe { ExFreePool(ptr.cast()); }
    }
}
```

**WinDbg에서 `!poolused`로 풀 사용량을 보면 태그가 `rust`로 표시된다.** little-endian 저장 순서를 고려해 일부러 `b"rust"`를 그대로 넘기는 디테일이 있다.

### 4. 빌드 파이프라인

`cargo build` 한 번으로 끝나지 않는 게 드라이버의 특성이다. INF 처리, 카탈로그 서명, 디바이스 노드 설치 정보 등 cargo 표준 빌드 단계 밖의 작업이 여럿 필요하다. WDR은 이를 `cargo-make`에 묶어 둔다.

#### 4-1. 사전 요구사항

* **LLVM 17.0.6** — `bindgen`이 사용하는 `libclang` 때문이다. `winget install -i LLVM.LLVM --version 17.0.6 --force`로 설치. **LLVM 18은 ARM64 바인딩 생성에 버그가 있어 19가 나오기 전까지는 17을 고정해야 한다.**
* **`cargo-make`** — `cargo install --locked cargo-make --no-default-features --features tls-native`.
* **eWDK 개발자 프롬프트** — WDK 환경 변수가 세팅된 셸에서만 동작.

#### 4-2. 빌드 단계

`cargo make`(또는 `cargo make default`)를 호출하면 대략 다음 순서로 진행된다.

```
cargo build (rust → .sys, KMDF/WDM의 경우. UMDF는 .dll)
   ↓
stampinf  (.inx → .inf, 버전·날짜 스탬핑)
   ↓
inf2cat   (.cat 카탈로그 생성)
   ↓
signtool  (테스트 인증서로 .sys와 .cat 서명)
   ↓
infverif  (INF 정적 검증)
   ↓
target/<profile>/package/ 에 결과물 + WDRLocalTestCert.cer 출력
```

결과물 디렉터리에 함께 떨어지는 **`WDRLocalTestCert.cer`** 는 WDR이 그 자리에서 즉석 생성한 self-signed 테스트 인증서이다. 테스트 머신에 설치하고 *test signing* 모드로 부팅하면 그대로 `.sys`를 로드할 수 있다. 양산 머신의 Trusted Root에는 절대 넣지 말 것 — README가 명시적으로 경고한다.

#### 4-3. `Cargo.toml` 보일러플레이트

샘플 KMDF 드라이버의 핵심 설정:

```toml
[package.metadata.wdk.driver-model]
driver-type = "KMDF"
kmdf-version-major = 1
target-kmdf-version-minor = 33

[lib]
crate-type = ["cdylib"]

[profile.dev]
lto = true
panic = "abort"

[profile.release]
lto = true
panic = "abort"

[lints.rust]
unsafe_op_in_unsafe_fn = "forbid"

[lints.clippy]
multiple_unsafe_ops_per_block = "forbid"
undocumented_unsafe_blocks = "forbid"
```

* `crate-type = ["cdylib"]` — `.sys`는 결국 PE 형식의 DLL이므로 cdylib로 빌드.
* `panic = "abort"` — `panic_handler`가 unwinding 없이 즉시 종료하도록 함. 커널에서는 unwinding 자체가 의미가 없다.
* `unsafe_op_in_unsafe_fn = "forbid"` + `multiple_unsafe_ops_per_block = "forbid"` — `unsafe fn` 안에서도 unsafe 연산은 다시 `unsafe { ... }` 블록으로 감싸야 하고, 한 블록 안에 unsafe 연산은 하나만 허용. **"어떤 unsafe 호출이 어떤 invariant에 책임지는지"를 1:1로 명확히 만드는 강한 규약이다.**

> [!TIP]
> `cargo-wdk`(현재 preview)는 위의 cargo-make 체인을 한 단계 더 단순화한다. 새 프로젝트라면 README 권장대로 `cargo-wdk`를 먼저 시도해 보고, 필요한 단계가 빠져 있으면 그때 cargo-make로 떨어지는 방향이 합리적이다.

### 5. 지원 범위와 한계

README의 *Supported Configurations* 섹션이 명확하다.

* **빌드 시스템 차원에서는 WDM, KMDF, UMDF, Win32 Services 모두 지원**한다. WDK 22H2 이상에서 검증됨.
* **하지만 `crates.io`에 게시된 크레이트는 현재 KMDF v1.33만 지원한다.** 다른 구성(UMDF 2.33, WDM 등)은 저장소를 클론한 뒤 `wdk-sys`의 `build.rs`에 있는 config를 직접 수정해야 한다.

> "Currently, the crates available on `crates.io` only support KMDF v1.33, but bindings can be generated for everything else by cloning `windows-drivers-rs` and modifying the config specified in `build.rs` of `wdk-sys`. Crates.io support for other WDK configurations is planned in the near future."

즉, **퍼블리시된 패키지만으로 바로 시작할 수 있는 시나리오는 KMDF 1.33이고**, 그 외는 약간의 fork-and-patch가 필요한 단계이다.

### 6. C 드라이버와 코드 모양 비교

같은 일을 하는 KMDF `DriverEntry`를 C와 Rust로 나란히 본다.

#### 6-1. C

```c
NTSTATUS
DriverEntry(_In_ PDRIVER_OBJECT DriverObject, _In_ PUNICODE_STRING RegistryPath)
{
    WDF_DRIVER_CONFIG config;
    WDF_DRIVER_CONFIG_INIT(&config, EvtDriverDeviceAdd);

    return WdfDriverCreate(DriverObject,
                           RegistryPath,
                           WDF_NO_OBJECT_ATTRIBUTES,
                           &config,
                           WDF_NO_HANDLE);
}
```

#### 6-2. Rust (WDR 샘플 발췌)

```rust
#![no_std]
extern crate alloc;
#[cfg(not(test))]
extern crate wdk_panic;

#[cfg(not(test))]
use wdk_alloc::WdkAllocator;
use wdk_sys::{
    call_unsafe_wdf_function_binding,
    DRIVER_OBJECT, NTSTATUS, PCUNICODE_STRING, PDRIVER_OBJECT,
    ULONG, WDFDRIVER, WDF_DRIVER_CONFIG, WDF_NO_HANDLE, WDF_NO_OBJECT_ATTRIBUTES,
};

#[cfg(not(test))]
#[global_allocator]
static GLOBAL_ALLOCATOR: WdkAllocator = WdkAllocator;

#[unsafe(export_name = "DriverEntry")]
pub unsafe extern "system" fn driver_entry(
    driver: &mut DRIVER_OBJECT,
    registry_path: PCUNICODE_STRING,
) -> NTSTATUS {
    // (1) WDF_DRIVER_CONFIG.Size 채우기 — C의 WDF_DRIVER_CONFIG_INIT 매크로 대체.
    //     단순한 `as ULONG` 캐스팅이 아니라, 컴파일 타임에 절단 가능성을 명시적으로 차단한다.
    let mut driver_config = {
        let wdf_driver_config_size: ULONG;
        #[allow(clippy::cast_possible_truncation)]
        {
            const WDF_DRIVER_CONFIG_SIZE: usize = core::mem::size_of::<WDF_DRIVER_CONFIG>();
            const { assert!(WDF_DRIVER_CONFIG_SIZE <= ULONG::MAX as usize) }
            wdf_driver_config_size = WDF_DRIVER_CONFIG_SIZE as ULONG;
        }
        WDF_DRIVER_CONFIG {
            Size: wdf_driver_config_size,
            EvtDriverDeviceAdd: Some(evt_driver_device_add),
            ..WDF_DRIVER_CONFIG::default()
        }
    };

    let driver_attributes = WDF_NO_OBJECT_ATTRIBUTES;
    let driver_handle_output = WDF_NO_HANDLE.cast::<WDFDRIVER>();

    // (2) 매크로 호출 결과는 반환 표현식이 아니라 변수에 보관한다.
    let wdf_driver_create_ntstatus;
    // SAFETY: 모든 포인터가 DriverEntry 계약에 의해 유효함이 보장된다.
    unsafe {
        wdf_driver_create_ntstatus = call_unsafe_wdf_function_binding!(
            WdfDriverCreate,
            driver as PDRIVER_OBJECT,
            registry_path,
            driver_attributes,
            &mut driver_config,
            driver_handle_output,
        );
    }

    // (3) 이 사이에 registry_path(UTF-16)를 Rust String으로 변환하고
    //     println!으로 찍는 처리가 들어간다(원본 참조).

    wdf_driver_create_ntstatus
}
```

비교에서 눈에 띄는 점.

* **`#[unsafe(export_name = "DriverEntry")]`** — OS 로더가 찾는 심볼 이름은 그대로 `DriverEntry`이지만, Rust 함수명은 snake_case로 둘 수 있다. `unsafe()` 속성 형식은 Rust 2024 edition의 새 문법이다.
* **`call_unsafe_wdf_function_binding!(WdfDriverCreate, ...)`** — `WdfDriverCreate`는 직접 호출할 수 있는 함수가 아니라 WDF가 런타임에 디스패치 테이블로 노출하는 함수이다. 매크로가 그 디스패치 호출을 unsafe FFI로 풀어낸다.
* **`WDF_DRIVER_CONFIG_INIT` 매크로의 부재.** C 매크로가 Rust로 자동 변환되지 않으므로, `Size`와 콜백을 손으로 채운다. 위 (1) 블록을 보면 단순한 `size_of() as ULONG` 캐스팅이 아니라, `const { assert!(... <= ULONG::MAX as usize) }`로 컴파일 타임 절단 검증을 명시적으로 끼워 넣고 있다. C에서는 매크로 한 줄로 끝나는 일이, Rust에서는 절단 가능성·`clippy::cast_possible_truncation` 린트와 명시적으로 싸우는 5줄 블록이 된다. **편의 매크로 한 줄이 사라지면 그 자리에 안전성 근거가 자라난다 — 이게 WDR이 호출 측에 떠넘기는 비용의 형태이다.**
* **반환 표현식이 아니라 변수 보관.** 위 (2)처럼 `call_unsafe_wdf_function_binding!`의 결과는 `let` 변수에 담긴 뒤, (3)에서 `registry_path` 후처리를 거치고 마지막에 함수 반환으로 흘러간다. 매크로 호출이 `unsafe` 블록 안에 갇혀야 하므로 그 블록을 함수 본문의 마지막 표현식으로 만들기 어렵고, 그 결과 자연스럽게 "결과 보관 → 후처리 → 반환" 패턴이 강제된다.
* **`unsafe extern "system" fn`** — 진입 함수 자체가 raw 포인터를 받으므로 `unsafe`이다. 내부에서 안전한 추상으로 변환하는 책임은 작성자에게 있다.

> [!NOTE]
> `call_unsafe_wdf_function_binding!` 자체가 한 줄의 unsafe 호출이고, 위에서 본 `multiple_unsafe_ops_per_block = "forbid"` 린트와 맞물려 **호출 한 건 = `unsafe` 블록 한 개 + 안전성 주석 한 개**가 강제된다. C에서는 어디서 SAL annotation을 빼먹어도 빌드가 되지만, Rust+WDR에서는 호출마다 안전성 근거를 적어야만 컴파일이 통과한다. 이게 코드 리뷰의 부담을 작성자 시점으로 옮긴다.

### 7. 한계와 주의점

양산 채택을 검토하기 전에 알아둬야 할 항목들.

* **`no_std` + `alloc` 환경.** `std::collections::HashMap`이나 `std::sync::Mutex` 같은 `std`만의 타입은 못 쓴다. 대신 `alloc::collections::BTreeMap`, `alloc::vec::Vec`, `alloc::string::String`은 `wdk-alloc`을 등록한 뒤 사용 가능.
* **`panic = "abort"` 강제.** 커널에서 unwinding이 불가능하기 때문이다. 또한 `wdk-panic`의 기본 구현이 `loop {}`이므로, 실제 양산에는 `KeBugCheckEx` 호출 panic_handler를 직접 구현해야 한다.
* **`crates.io` 버전이 항상 GitHub HEAD가 아니다.** README의 *Crates.io Release Policy*에 명시되어 있다: "Releases will only be made when requested by the community, or when the `windows-drivers-rs` team believes there is sufficient value in pushing a release." 새 기능이 필요하다면 path/git dependency로 끌어 써야 할 수 있다.
* **WHQL/HLK 인증의 공개 사례가 적다.** 빌드 산출물 형태(`.sys` + `.inf` + `.cat`)는 동일하므로 *원칙적으로는* HLK 테스트 통과가 가능하지만, **WHQL 인증을 통과한 Rust 드라이버를 외부에서 공개적으로 확인하기는 어렵다.** 단, Microsoft 내부에서는 사용 사례가 분명히 존재한다 — Microsoft Surface 팀이 Tech Community 블로그(2024-01)에서 `windows-drivers-rs`를 Surface 드라이버 개발에 활용하고 있다고 명시적으로 언급했다. 즉 "양산 가능성"과 "외부에서 본 사례 부족"을 분리해서 봐야 한다.
* **bindgen 출력 안정성.** WDK 헤더가 업데이트되면 자동 생성된 FFI 시그니처가 바뀔 수 있다. `wdk-build`가 WDK 22H2+로 검증되어 있다는 말은 그 외 버전에서는 미세한 차이가 생길 수 있다는 뜻이기도 하다.
* **WDF preview 단계.** README가 명시적으로 "not yet recommended for production use"라고 못 박는다. **현 시점(2026 기준)에서는 새 드라이버를 Rust로 짜는 것보다 기존 C 드라이버의 일부 모듈을 Rust로 교체하는 partial-Rust 방식이 위험을 줄이는 길이다.**

### 8. 출처

* [microsoft/windows-drivers-rs (GitHub)](https://github.com/microsoft/windows-drivers-rs) — README, Supported Configurations, Getting Started, Cargo Make 섹션. 본문의 모든 인용은 main 브랜치 HEAD 기준.
* [Windows-rust-driver-samples](https://github.com/microsoft/Windows-rust-driver-samples) — Microsoft가 별도로 운영하는 Rust 드라이버 샘플 모음. 실전 코드 패턴은 이쪽이 더 풍부하다.
* `windows-drivers-rs/examples/sample-kmdf-driver` — 본문 6장의 Rust 코드 인용 원본.
* `windows-drivers-rs/crates/wdk-alloc/src/lib.rs`, `crates/wdk-panic/src/lib.rs`, `crates/wdk/src/wdf/spinlock.rs` — 3장의 코드 인용 원본.
* [cargo-wdk on crates.io](https://crates.io/crates/cargo-wdk) — cargo-make를 대체할 신규 빌드 도구.
* [Using the Enterprise WDK (eWDK)](https://learn.microsoft.com/en-us/windows-hardware/drivers/develop/using-the-enterprise-wdk) — WDR 빌드의 전제가 되는 eWDK 환경 설정.
* Microsoft Tech Community — Surface 팀의 Rust 드라이버 도입 관련 게시물(2024-01). 7장의 "Microsoft 내부 사용 사례" 진술의 1차 출처.

더 깊이 다루는 자료:

* **Rust for Linux** — `kernel.org`의 `Documentation/rust/`. 커널 측 Rust 적용을 다른 관점에서 본 비교 자료. 메모리 안전 슬로건이 어디까지를 커버하고 못 하는지에 대한 토론이 풍부하다.
* **`bindgen` 가이드** — WDK 헤더가 Rust로 어떻게 매핑되는지를 이해하려면 직접 헤더와 생성된 코드를 비교해 보는 게 빠르다.
* **`call_unsafe_wdf_function_binding!` 매크로 구현** — `windows-drivers-rs/crates/wdk-macros`. WDF 함수 디스패치가 런타임에 어떻게 풀리는지가 한 매크로에 압축돼 있어 읽어 둘 가치가 있다.
