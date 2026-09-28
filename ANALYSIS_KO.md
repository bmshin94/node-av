# node-av 분석 & 대화 정리 📚

- **이 저장소(포크):** https://github.com/bmshin94/node-av
- **원본 저장소:** https://github.com/seydx/node-av
- **공식 문서:** https://seydx.github.io/node-av
- **npm:** https://www.npmjs.com/package/node-av
- 분석 기준 버전: `6.0.0-beta.5` (FFmpeg 8.1 Jellyfin 빌드, MIT 라이선스)

---

## 1. 전수조사 분석 — 뭐하는 건지 / 언제 쓰는지 / 나한테 무슨 도움이 되는지

### 한 줄 요약
**node-av = Node.js(JavaScript/TypeScript)에서 FFmpeg를 "라이브러리처럼" 직접 호출하게 해주는 네이티브 바인딩.**
FFmpeg 명령어를 문자열로 조립해서 `spawn` 하는 방식이 아니라, FFmpeg의 C 함수(libavformat, libavcodec, libavfilter 등)를 N-API(C++ 애드온)로 연결해서 JS 객체/함수로 다룬다.

### 폴더 구조 분석

| 폴더/파일 | 역할 | 비고 |
|---|---|---|
| `src/bindings/` | **C++ 네이티브 코드** (약 100개 파일, ~2.6만 줄). FFmpeg C API ↔ Node.js를 잇는 다리 | 각 기능마다 `_sync.cc` / `_async.cc` 짝으로 동기·비동기 둘 다 제공 |
| `src/lib/` | **로우레벨 API** (~2.2만 줄). FFmpeg 구조체를 거의 1:1로 옮긴 TS 클래스 (`FormatContext`, `CodecContext`, `Frame`, `Packet`, `FilterGraph` …) | FFmpeg C 예제를 그대로 TS로 옮길 수 있는 수준 |
| `src/api/` | **하이레벨 API**. 쉽게 쓰는 래퍼 | `Demuxer`, `Decoder`, `Encoder`, `Muxer`, `pipeline`, `HardwareContext`, `FilterComplexAPI`, `FilterPreset`, `DeviceAPI`, `WhisperTranscriber`, `FMP4Stream`, `WebRTCStream`, `RTPStream`, `probe` 등 |
| `src/constants/` | FFmpeg 상수·옵션 타입 (~2.5만 줄, **자동 생성**) | 코덱/포맷/필터(~580개) 옵션에 자동완성 + 오타 컴파일 에러 |
| `src/ffmpeg/` | FFmpeg **실행파일(CLI)** 자동 다운로드·경로 제공 | `ffmpegPath()`, `isFfmpegAvailable()` |
| `src/webrtc/` | `node-av/webrtc` 서브패스 (werift 기반 WebRTC/RTP) | 선택 의존성 |
| `src/utils/` | Electron 지원 유틸 | |
| `examples/` | 예제 57개 + 브라우저(fMP4, WebRTC) + Electron(Builder, Forge) | FFmpeg 공식 C 예제의 TS 버전 포함 |
| `test/` | 테스트 48개 (`node --test` + tsx) | |
| `testdata/` | 테스트용 영상/음성 샘플 | |
| `benchmarks/`, `BENCHMARK.md` | FFmpeg CLI 대비 성능 비교 | 트랜스코딩 속도 CLI와 거의 동일 |
| `scripts/` | 상수/옵션 타입 자동 생성, 릴리스 스크립트 | |
| `install/` | 설치 시 프리빌트 바이너리 확인 → 없으면 소스 빌드 | |
| `packages/` | 플랫폼별 npm 패키지 템플릿 (`@seydx/node-av-<os>-<arch>`) | |
| `externals/` | git 서브모듈 (jellyfin-ffmpeg, node-gyp, binary-data) | 클론만 하면 비어 있음, 소스 빌드용 |
| `binding*.gyp` | 네이티브 빌드 설정 (일반 / Jellyfin / MSVC) | |
| `.github/workflows/` | 프리빌트 빌드·배포, 문서 배포 CI | |
| `CLAUDE.md` | 이 포크에서 추가한 AI 페르소나 가이드 | 원본에는 없음 |

### 3개의 API 층
1. **Low-Level (`node-av/lib`)** — FFmpeg C API 그대로. 최대 자유도, 대신 어렵고 메모리 실수 주의.
2. **High-Level (`node-av/api`)** — `Demuxer.open()` → `Decoder` → `Encoder` → `Muxer`. `for await`로 패킷/프레임 흘려보냄.
3. **Pipeline** — `pipeline(input, decoder, encoder, output)` 한 줄로 변환 파이프라인.

### 주요 기능
- 영상/음성 **변환(트랜스코딩)**, 리먹싱, 자르기, 리사이즈, 프레임(썸네일) 추출, 메타데이터 조회(probe)
- **하드웨어 가속** 자동 감지 (`HardwareContext.auto()` — CUDA, VAAPI, QSV, VideoToolbox 등)
- **필터**: 워터마크, PIP, 그리드 합성 등 FFmpeg 필터 전부 (타입 안전)
- **장치 캡처**: 웹캠, 마이크, 화면 녹화 (macOS/Windows/Linux)
- **스트리밍**: RTSP 카메라 입력 → 브라우저로 fMP4(MSE) 또는 WebRTC 저지연 송출, RTSP 백채널(카메라에 말하기)
- **Whisper 음성인식**: whisper.cpp 내장, 모델 자동 다운로드, 자막(SRT) 생성
- **Electron** 지원 (재빌드 불필요, GPU 공유 텍스처 제로카피)
- `using` / `await using`으로 **자동 리소스 해제**, 모든 비동기 메서드에 `Sync` 버전 존재

### 언제 쓰나?
- Node.js 백엔드에서 업로드 영상을 변환/썸네일 생성할 때
- CCTV·IP카메라(RTSP)를 웹에서 실시간으로 보여줄 때
- 영상 편집·녹화 데스크톱 앱(Electron)을 만들 때
- 음성/영상 → 자막·텍스트 자동 생성할 때
- `fluent-ffmpeg` + CLI 호출로는 부족한 **프레임 단위 제어**가 필요할 때

### 나한테 도움 되는 점
- JS/TS만 알아도 FFmpeg급 미디어 처리 가능 (C 몰라도 됨)
- FFmpeg 별도 설치 불필요 — `npm install` 하면 OS별 바이너리 자동
- 옵션 자동완성으로 FFmpeg 문서 뒤질 일이 줄어듦
- 로컬 Whisper로 **API 비용 없이** 음성인식
- 영상 SaaS, 자막 서비스, CCTV 대시보드 같은 제품의 엔진으로 바로 활용

---

## 2. 더 쉽게 설명하기 🍳

**비유: FFmpeg = 세계 최고 성능의 "영상 요리 공장"**, 근데 공장 직원은 C언어만 알아들어.
**node-av = 그 공장에 JavaScript로 주문할 수 있는 "통역사 + 리모컨".**

- 예전 방식: 공장에 쪽지(명령어 문자열)를 던지고 결과만 기다림 → 중간에 뭘 하는지 모름
- node-av 방식: 리모컨으로 공장 라인을 직접 조종 → 영상 한 장면(프레임)마다 손댈 수 있음

**영상 처리 5단계로 이해하기**
1. `Demuxer` = 택배 상자 뜯기 (mp4에서 영상/소리 꺼내기)
2. `Decoder` = 압축 풀기 (사진 한 장 한 장으로)
3. `Filter` = 꾸미기 (크기 조절, 로고, 자막)
4. `Encoder` = 다시 압축 (용량 줄이기)
5. `Muxer` = 다시 포장 (새 mp4 파일로)

**3가지 모드** = 수동 운전(Low-Level) / 자동 변속(High-Level) / 자율주행(Pipeline)

**실생활 예시**
- 유튜브처럼 업로드 영상을 여러 화질로 자동 변환
- 집 CCTV를 웹 브라우저에서 바로 보기
- 강의 녹음 → 자동 자막 생성
- 화면 녹화 프로그램 만들기

---

## 3. Q&A

### 설치 및 사용법
```bash
npm install node-av            # Node.js 필요 (ESM, TypeScript 권장)
```
- 설치 시 OS별 프리빌트(`@seydx/node-av-<os>-<arch>`)가 자동 선택, FFmpeg CLI도 자동 다운로드 (`SKIP_FFMPEG=true`로 생략 가능)
- 지원: macOS(x64/arm64), Linux(x64/arm64), Windows(x64/arm64, MSVC/MinGW)
- Linux 하드웨어 가속(Intel VAAPI)은 Ubuntu 24.04+/Debian 13+ 필요

```ts
import { Decoder, Demuxer, Encoder, HardwareContext, Muxer, pipeline } from 'node-av/api';
import { FF_ENCODER_LIBX264 } from 'node-av/constants';

await using input = await Demuxer.open('input.mp4');
const video = input.video()!;
using hw = HardwareContext.auto();
using decoder = await Decoder.create(video, { hardware: hw });
using encoder = await Encoder.create(FF_ENCODER_LIBX264, { decoder });
await using output = await Muxer.open('output.mp4', { input });

await pipeline(input, decoder, encoder, output).completion;
```
- 예제 실행(이 저장소 클론 시): `npm install` → `npx tsx examples/api-sw-transcode.ts ...`

### 플러그인? 스킬? MCP?
**셋 다 아님. 일반 npm 라이브러리(네이티브 애드온)**이다. Claude 플러그인·스킬·MCP 서버가 아니다.
단, 이걸 엔진으로 써서 **MCP 서버나 스킬을 직접 만들 수는 있다** (예: "영상 변환 MCP", "자막 생성 MCP").

### API 토큰 필요해?
**필요 없음.** 전부 로컬에서 동작한다.
- Whisper 모델은 처음 한 번 HuggingFace에서 무료 다운로드 (토큰 불필요)
- 설치 시 GitHub Releases에서 FFmpeg 바이너리 다운로드 (토큰 불필요)
- 영상 처리 자체는 인터넷도 필요 없음

### 왜 GitHub에서 유명할까?
- Node.js + FFmpeg 조합은 수요가 큰데, 대표 격이던 `fluent-ffmpeg`는 CLI 래퍼이고 유지보수가 중단됨 → 빈자리를 **진짜 네이티브 바인딩**이 채움
- **FFmpeg 별도 설치 없이** `npm install` 한 번이면 끝 (프리빌트)
- **완전한 TypeScript 타입** — 모든 코덱/필터 옵션 자동완성
- CLI와 **동등한 성능**(벤치마크 공개), 하드웨어 가속
- Whisper, WebRTC, RTSP 백채널, Electron 등 실전 기능 + 57개 예제 + 문서 사이트
- 활발한 릴리스(CHANGELOG), 홈 IoT/카메라(Homebridge 계열) 커뮤니티 수요

### 로컬 에이전트 구축에 도움 될까?
**된다 — 특히 "귀와 눈" 역할.**
- 🎤 **귀**: 마이크 캡처(`DeviceAPI.openMicrophone`) + Whisper 음성인식 → 텍스트를 로컬 LLM(Ollama 등)에 전달
- 👀 **눈**: 웹캠/화면 캡처 → 프레임 추출 → 비전 모델에 이미지 입력
- 🛠️ **손(툴)**: "영상 자르기/변환/자막 달기"를 에이전트 툴이나 MCP 서버로 노출
- 전부 오프라인·무료. 단 LLM 자체와 TTS(말하기)는 포함되지 않음

### 수익화 아이디어 (요약 — 상세는 4번)
자막 자동 생성 SaaS, 영상 변환/압축 API, CCTV 웹 뷰어·AI 알림, 쇼츠 자동 편집기, 회의 녹음 요약 앱, 미디어 MCP 서버 판매, 강의 영상 자동화 등.

### React나 PHP로 만들 수 있어?
- **React**: 화면(프론트엔드)은 React로 OK. 단 node-av는 **서버(Node.js)나 Electron 메인 프로세스에서만** 동작 (브라우저 불가). 구조: React(UI) ↔ Node.js API 서버(node-av) — Next.js API Route, Express 등.
- **PHP**: PHP에서 직접 import는 **불가** (Node.js 전용). 방법: ① PHP → Node.js 마이크로서비스(HTTP) 호출, ② PHP에서 `ffmpeg` CLI 직접 실행 (`PHP-FFMpeg` 라이브러리), ③ node-av가 받아준 ffmpeg 바이너리 경로를 PHP에서 `exec`.
- **추천 조합**: React(프론트) + Node.js(node-av) 백엔드, 기존 PHP 서비스가 있으면 Node 서비스를 옆에 붙이는 형태.

---

## 4. 수익화 아이디어 상세 💰

| # | 아이디어 | 핵심 node-av 기능 | 수익 모델 | 난이도 |
|---|---|---|---|---|
| 1 | **한국어 자동 자막 SaaS** (영상 업로드 → SRT/번역 자막) | Whisper, Demuxer, 자막 필터 | 분당 과금, 월 구독 | ⭐⭐ |
| 2 | **쇼츠/릴스 자동 제작기** (긴 영상 → 세로 9:16 크롭 + 자막 번인 + 하이라이트) | FilterComplex(crop/scale/overlay/drawtext), Whisper | 월 구독(크리에이터) | ⭐⭐⭐ |
| 3 | **영상 변환/압축 API** (개발자용) | Encoder, HW 가속, Pipeline | 사용량 과금 API | ⭐⭐ |
| 4 | **CCTV·IP카메라 웹 대시보드 + AI 알림** (소상공인·매장) | RTSP 입력, fMP4/WebRTC 송출, 프레임 추출 → 비전 AI | 카메라당 월정액, 설치비 | ⭐⭐⭐⭐ |
| 5 | **회의·강의 녹음 요약 데스크톱 앱** (Electron, 완전 오프라인) | 마이크/화면 캡처, Whisper | 1회 구매 or 구독 (보안 중시 기업) | ⭐⭐⭐ |
| 6 | **미디어 처리 MCP 서버 / 에이전트 툴 판매** | 전체 API를 툴로 노출 | 유료 템플릿, 기업 커스터마이징 | ⭐⭐ |
| 7 | **이커머스 상품 영상 자동 생성** (사진+BGM → 슬라이드 영상, 워터마크) | Encoder, overlay/fade 필터 | 쇼핑몰 셀러 구독 | ⭐⭐⭐ |
| 8 | **썸네일·미리보기 자동 생성** (GIF 프리뷰, 스프라이트 시트) | 프레임 추출, scale | API 과금, 플랫폼 B2B | ⭐ |
| 9 | **라이브 방송 보조 툴** (화면+캠 합성, 저지연 송출) | DeviceAPI, FilterComplex, WebRTC | 스트리머 구독 | ⭐⭐⭐⭐ |
| 10 | **외주·컨설팅** (영상 파이프라인 구축 대행) | 전체 | 프로젝트 단가 | ⭐⭐ |

**시작 추천 순서**
1. MVP: **#1 자동 자막** 또는 **#8 썸네일 API** — 구현 쉬움, 수요 확실, API 비용 0원 (로컬 Whisper)
2. 확장: 자막 + 크롭 + 하이라이트 = **#2 쇼츠 자동 제작기**로 발전
3. B2B: **#4 CCTV 대시보드**, **#5 오프라인 회의록** — 단가 높고 반복 매출

**비용 구조 장점**: 외부 AI API 토큰 비용 없음 → 서버(GPU/CPU) 비용만 → 마진 확보 유리.

**주의사항**
- **라이선스**: node-av는 MIT지만 FFmpeg는 LGPL/GPL (x264/x265 등 GPL 코덱 포함 빌드 주의). 상용 배포(특히 데스크톱 앱) 전 라이선스 검토 필수
- 코덱 특허(H.264/HEVC) 이슈 — 필요 시 AV1/VP9 등 로열티 프리 코덱 고려
- 현재 `6.0.0-beta` — 운영 서비스는 안정 버전 고정 권장
- 영상 처리는 CPU/GPU 비용이 크므로 큐(작업 대기열)와 과금 설계 필요

---

## 5. 정리 작업
- 이 문서(`ANALYSIS_KO.md`)에 1~4번 대화 내용을 정리하여 저장하고, 브랜치에서 커밋 후 `main`에 머지함.
