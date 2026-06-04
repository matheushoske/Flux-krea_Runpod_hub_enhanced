# Flux Krea RunPod Hub — Enhanced (향상판)

[English README](README.md)

[wlsdml1114](https://github.com/wlsdml1114)의 [Flux-krea_Runpod_hub](https://github.com/wlsdml1114/Flux-krea_Runpod_hub) 템플릿을 기반으로 한 **향상된 포크(fork)** 저장소입니다. RunPod Serverless + ComfyUI + Flux Krea 스택은 동일하며, 업스트림 Hub 템플릿에 없는 API 기능—특히 요청에 선택적 `image` 필드를 통한 **이미지-이미지(img2img)**—을 추가했습니다.

| | 업스트림 ([Flux-krea_Runpod_hub](https://github.com/wlsdml1114/Flux-krea_Runpod_hub)) | 이 저장소 (**Flux-krea_Runpod_hub_enhanced**) |
| --- | --- | --- |
| 텍스트-이미지 (txt2img) | 지원 | 지원 (API 동일) |
| 이미지-이미지 (img2img) | 미지원 | **지원** — `image` + 선택 `denoise` |
| 입력 이미지 형식 | — | HTTP(S) URL, raw base64, `data:image/...;base64,...` |
| LoRA (0~3개) | 지원 | 지원 (동일) |
| 네트워크 볼륨 커스텀 UNET | 지원 | 지원 (동일) |
| ComfyUI 워크플로우 | txt2img 전용 JSON 4개 | 동일 4개 + img2img 노드는 `handler.py`에서 런타임 주입 |

[![Runpod Hub (업스트림)](https://api.runpod.io/badge/wlsdml1114/Flux-krea_Runpod_hub)](https://console.runpod.io/hub/wlsdml1114/Flux-krea_Runpod_hub)

**img2img**가 필요하면 **이 저장소**를 배포하세요. txt2img만 필요하고 공식 RunPod Hub 목록을 쓰려면 업스트림 템플릿을 사용하세요.

---

## 이 포크에서 추가된 기능

### 1. 이미지-이미지 (`image` 파라미터)

**원본 이미지**와 **프롬프트**를 함께내면, 빈 잠재 공간에서 시작하는 txt2img 대신 원본을 변형합니다.

- **`image` 없음**: 기존과 동일한 텍스트-이미지.
- **`image` 있음**: img2img 파이프라인이 자동 활성화됩니다.

지원하는 `image` 형식:

| 형식 | 예시 | 비고 |
| --- | --- | --- |
| 공개 URL | `"https://example.com/photo.jpg"` | 워커 내부에서 다운로드 (60초 타임아웃). |
| Raw base64 | `"iVBORw0KGgo..."` | 파일 헤더로 PNG/JPEG/GIF/WebP 감지. |
| Data URI | `"data:image/png;base64,iVBORw0..."` | 접두사 제거 후 디코딩. |
| 로컬 경로 | `"/path/on/worker/input.png"` | 워커에 이미 있는 파일(네트워크 볼륨 등). |

처리 순서:

1. 이미지를 디스크 임시 파일로 변환
2. ComfyUI 준비 대기
3. ComfyUI에 업로드 (`POST /upload/image`)
4. 워크플로우에 `LoadImage` → `ImageScale` → `VAEEncode` 노드 주입
5. `VAEEncode`를 `KSampler`에 연결하고 `denoise` 설정

`width` / `height`는 img2img에서도 적용됩니다. 원본은 해당 크기로 Lanczos 스케일 후 인코딩됩니다.

### 2. 변환 강도 (`denoise`)

| 파라미터 | 필수 | 기본값 | 범위 | 사용 시점 |
| --- | --- | --- | --- | --- |
| `denoise` | 아니오 | `0.75` | `0.0` ~ `1.0` | `image`가 있을 때만 |

- **낮음** (`0.5`~`0.65`): 원본 구도·레이아웃 유지에 가깝게
- **높음** (`0.8`~`1.0`): 프롬프트 반영이 강함, 거의 새로 그리기에 가깝게

txt2img 요청에서는 무시됩니다.

### 3. 이미지 입력 관련 명확한 오류 메시지

잘못된 URL, 손상된 base64, ComfyUI 업로드 실패 시 예:

```json
{ "error": "Invalid input image: image must be a valid URL, base64 string, data URI, or an existing file path" }
```

---

## 생성 모드

- **txt2img**: `image` 없음 → `EmptySD3LatentImage` → KSampler  
- **img2img**: `image` 있음 → LoadImage → ImageScale → VAEEncode → KSampler (`denoise` < 1)

**img2img는 LoRA 및 커스텀 `model`과 함께 사용 가능**합니다.

---

## ✨ 기능 (상속 + 향상)

* **텍스트-이미지**: Flux Krea + 듀얼 CLIP
* **이미지-이미지** *(이 포크)*: `image` + `denoise`로 원본 변환
* **다중 LoRA**: 최대 3개, 개수에 따라 워크플로우 자동 선택
* **커스텀 UNET**: 네트워크 볼륨 경로
* **ComfyUI**: img2img 노드는 `handler.py`에서 동적 추가

---

## 🚀 RunPod Serverless 구성

| 파일 | 역할 |
| --- | --- |
| `Dockerfile` | ComfyUI, Flux Krea 가중치, 의존성 |
| `handler.py` | 요청 처리, img2img 업로드, 워크플로우 패치 |
| `entrypoint.sh` | ComfyUI 및 RunPod 핸들러 시작 |
| `flux_krea_dev_api_*.json` | LoRA 0~3개용 기본 txt2img 워크플로우 |

---

## API 참조

### `input` 필드

`model`, `lora`, `image`, `denoise`를 제외한 모든 필드는 **필수**입니다.

| 파라미터 | 타입 | 필수 | 기본값 | 설명 |
| --- | --- | --- | --- | --- |
| `prompt` | `string` | **예** | — | 생성/변환용 텍스트 |
| `seed` | `integer` | **예** | — | 랜덤 시드 |
| `guidance` | `float` | **예** | — | CFG (KSampler `cfg`). Flux Krea는 보통 `1` 근처 |
| `width` | `integer` | **예** | — | 출력 너비 (img2img 입력 스케일에도 사용) |
| `height` | `integer` | **예** | — | 출력 높이 |
| `model` | `string` | 아니오 | `flux1-krea-dev_fp8_scaled.safetensors` | 커스텀 UNET (네트워크 볼륨) |
| `lora` | `array` | 아니오 | `[]` | `[경로, 가중치]` 배열, 최대 3개 |
| `image` | `string` | 아니오 | — | **향상:** img2img 원본 (URL, base64, data URI) |
| `denoise` | `float` | 아니오 | `0.75` | **향상:** img2img 강도; `image` 없으면 무시 |

---

### 요청 예시

**텍스트-이미지 (업스트림과 동일):**

```json
{
  "input": {
    "prompt": "산과 호수가 있는 아름다운 풍경",
    "seed": 12345,
    "guidance": 1,
    "width": 1024,
    "height": 1024
  }
}
```

**이미지-이미지 — URL:**

```json
{
  "input": {
    "prompt": "이 사진을 수채화 스타일로, 부드러운 파스텔 색감",
    "seed": 12345,
    "guidance": 1,
    "width": 1024,
    "height": 1024,
    "image": "https://example.com/photo.jpg",
    "denoise": 0.65
  }
}
```

**이미지-이미지 — base64 data URI:**

```json
{
  "input": {
    "prompt": "비 오는 밤의 사이버펑크 도시, 네온",
    "seed": 12345,
    "guidance": 1,
    "width": 1024,
    "height": 1024,
    "image": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mP8z8BQDwAEhQGAhKmMIQAAAABJRU5ErkJggg=="
  }
}
```

**img2img + LoRA + 커스텀 모델:**

```json
{
  "input": {
    "prompt": "애니메 스타일 초상화, 디테일한 눈",
    "seed": 42,
    "guidance": 1,
    "width": 1024,
    "height": 1024,
    "image": "https://example.com/portrait.jpg",
    "denoise": 0.7,
    "model": "/my_volume/models/custom_model.safetensors",
    "lora": [
      ["/my_volume/loras/style_lora.safetensors", 0.8]
    ]
  }
}
```

---

### `denoise` 가이드 (img2img)

| `denoise` | 용도 |
| --- | --- |
| `0.4` – `0.55` | 색감·가벼운 스타일, 구도 유지 |
| `0.6` – `0.75` | 스타일 전환, 중간 강도 (기본 `0.75`) |
| `0.8` – `0.95` | 강한 재해석 |
| `1.0` | 잠재 공간에서 거의 전면 재생성에 가깝게 |

---

### 출력

**성공:** `{ "image": "<base64 PNG>" }`  
**실패:** `{ "error": "<메시지>" }`

---

## 🛠️ 배포

1. **이** Git 저장소로 RunPod Serverless 엔드포인트 생성  
2. 빌드 후 `input` JSON으로 작업 제출  
3. `handler.py` 변경 후에는 이미지 재빌드/재배포 필요  

### 📁 네트워크 볼륨

`model` / `lora`는 볼륨 마운트 필요. 대용량 입력은 base64보다 URL 또는 볼륨 경로 권장.

---

## 🔧 워크플로우 파일

| 파일 | LoRA 수 |
| --- | --- |
| `flux_krea_dev_api_nolora.json` | 0 |
| `flux_krea_dev_api_1lora.json` | 1 |
| `flux_krea_dev_api_2lora.json` | 2 |
| `flux_krea_dev_api_3lora.json` | 3 |

`image` 사용 시 `handler.py`가 노드 **60** (LoadImage), **62** (ImageScale), **61** (VAEEncode)을 추가하고 **31** (KSampler)을 재연결합니다.

---

## 🙏 크레딧

| 프로젝트 | 링크 |
| --- | --- |
| **업스트림 (포크 기반)** | [wlsdml1114/Flux-krea_Runpod_hub](https://github.com/wlsdml1114/Flux-krea_Runpod_hub) |
| **Flux Krea** | [bfl.ai](https://bfl.ai/blog/flux-1-krea-dev) |
| **ComfyUI** | [GitHub](https://github.com/comfyanonymous/ComfyUI) |

가중치, Docker 베이스, 핵심 워크플로우는 업스트림에 있습니다. 이 저장소는 **핸들러**와 **문서**만 확장합니다.

---

## 📄 라이선스

업스트림 Flux Krea / ComfyUI 템플릿과 동일한 라이선스를 따릅니다.
