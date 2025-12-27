# 기술 조사: 부모님 결혼식 타임슬립 서비스

> **서비스 컨셉**: 부모님 결혼식 사진 1-2장 + 내 사진 → 결혼식에 참석한 다양한 장면 생성

## 핵심 기술 스택 요약

```
┌─────────────────────────────────────────────────────────────┐
│                    서비스 아키텍처                           │
├─────────────────────────────────────────────────────────────┤
│  입력: 부모님 결혼식 사진 + 사용자 사진                        │
│                         ↓                                   │
│  1. 얼굴 인식/추출 (InsightFace)                             │
│                         ↓                                   │
│  2. 장면 분석 & 시대 감지 (CLIP + Custom Model)              │
│                         ↓                                   │
│  3. 새 장면 생성 (FLUX / SDXL + ControlNet)                  │
│                         ↓                                   │
│  4. 얼굴 합성 (InstantID / PuLID / IP-Adapter FaceID)        │
│                         ↓                                   │
│  5. 스타일 통일 (빈티지 필터 + 후처리)                         │
│                         ↓                                   │
│  출력: 5-10장의 결혼식 참석 장면                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 1. 얼굴 인식 및 추출 기술

### InsightFace (추천)
- **GitHub**: https://github.com/deepinsight/insightface
- **특징**:
  - 오픈소스 2D/3D 얼굴 분석 라이브러리
  - PyTorch/MXNet 기반
  - 얼굴 감지, 정렬, 인식, 속성 분석 지원
- **최신 업데이트 (2025.11)**:
  - INSwapper_Dax 기술로 고품질 페이스 스왑 지원
  - iOS/macOS 앱 출시
  - InspireFace SDK: 엣지 디바이스에서 2ms 미만 추론 가능

### 얼굴 임베딩 모델
| 모델 | 특징 | 용도 |
|------|------|------|
| buffalo_l | 표준 분석 모델 | 일반적인 얼굴 인식 |
| antelopev2 | 고정밀 모델 | 정밀한 특징 추출 |

---

## 2. 얼굴 합성/스왑 기술

### 방법 비교 (2025년 기준)

| 기술 | 품질 | 속도 | 설치 난이도 | 추천 용도 |
|------|------|------|-------------|-----------|
| **InstantID** | ⭐⭐⭐⭐⭐ | 보통 | 중간 | 고품질 결과물 |
| **PuLID** | ⭐⭐⭐⭐⭐ | 느림 | 어려움 | 최고 품질 필요시 |
| **IP-Adapter FaceID V2** | ⭐⭐⭐⭐ | 빠름 | 쉬움 | 대량 생산 |
| **ReActor** | ⭐⭐⭐ | 매우 빠름 | 쉬움 | 빠른 프로토타입 |

### InstantID
- 단일 참조 이미지로 얼굴 아이덴티티 전송
- InsightFace + IP-Adapter 조합
- 얼굴 임베딩을 추출하여 이미지 생성 제어

### PuLID (Pure and Lightning ID)
- **FLUX와 통합 시 최고 품질**
- 저장 공간 많이 필요하지만 결과물 최상
- 얼굴 특징 보존에 탁월

### IP-Adapter FaceID Plus V2
- IP-Adapter 생태계의 일부로 설치 간편
- 커스텀/머지 체크포인트와 호환
- 월 500장 이하 생산 시 충분한 품질

---

## 3. 이미지 생성 모델

### FLUX (Black Forest Labs) - 2025년 최신
- **FLUX.1 Kontext** (2025.05 출시)
  - 텍스트 + 이미지 동시 처리
  - 캐릭터 일관성 유지에 탁월
  - 다른 환경에서도 얼굴 특징, 표정 완벽 보존

- **FLUX.2**
  - 프로덕션급 일관성
  - 멀티 레퍼런스 컨트롤: 동일 캐릭터로 수백 장 생성 가능
  - 1개 이상의 참조 이미지로 아이덴티티 고정

### Stable Diffusion XL (SDXL)
- 고해상도 이미지 생성
- 인페인팅 모델 별도 제공
- ControlNet과 조합하여 정밀 제어

### InfiniteYou Framework (연구)
- 자유로운 텍스트 설명으로 사진 재구성
- 얼굴 아이덴티티 보존하면서 새로운 장면 생성
- FLUX 기반 DiT 모델 활용

---

## 4. 장면 제어 기술 (ControlNet)

### 핵심 컨트롤 타입
| 타입 | 정확도 | 용도 |
|------|--------|------|
| Canny Edge | 94.2% | 환경 엣지 보존 |
| Depth Map | 91.8% | 3D 공간 관계 분석 |
| OpenPose | 88.5% | 인물 포즈 감지/재현 |

### 장면 배치 전략
```
결혼식 장면 생성 워크플로우:
1. Canny → 원본 결혼식 사진의 환경 엣지 추출
2. OpenPose → 사용자 포즈 감지 및 배치
3. Depth → 전경/배경 분리로 자연스러운 합성
```

### Qwen-Edit 2509 (2025.09)
- 200억 파라미터 모델
- 멀티 이미지 편집: 1-3장 입력 지원
- **person-to-scene 조합에 최적화**

---

## 5. 빈티지 스타일 변환

### 자동 스타일 감지 & 적용
80-90년대 결혼식 사진 특징:
- 필름 그레인
- 색 바램 (faded colors)
- 세피아/웜톤
- 라이트 리크 효과

### 스타일 변환 도구

| 도구 | 특징 | API |
|------|------|-----|
| Style AI | 90년대 사진 특화 | O |
| Pixelcut | 시대별 프리셋 | O |
| Fotor | Gemini 기반 편집 | O |
| Stable Diffusion | 로컬 처리 가능 | - |

### Stable Diffusion 로컬 처리
```python
# 빈티지 스타일 프롬프트 예시
prompt = """
1980s wedding photo style,
film grain, slightly faded colors,
warm sepia tone, soft focus,
Kodak film aesthetic
"""
```

---

## 6. 인페인팅 기술

### 개념
- 마스크 영역만 선택적으로 수정
- 원본 사진의 나머지 부분 보존
- 텍스트 프롬프트로 새 요소 추가

### 주요 모델
- Stable Diffusion Inpainting
- SDXL Inpainting (고해상도)
- Kandinsky 2.2 Inpainting

### 핵심 파라미터
- **Denoising Strength**: 마스크 영역 변화 정도 제어
- 값이 너무 높으면 주변과 불일치 발생
- 0.4-0.7 범위 권장

---

## 7. 구현 옵션

### Option A: API 기반 (빠른 MVP)

```
[프론트엔드] → [백엔드 서버] → [Replicate API]
                              ↓
                         - Face Swap: $0.013/회
                         - FLUX: 별도 요금
                         - 스타일 변환: 별도 요금
```

**Replicate 주요 모델**
| 모델 | 비용 | 특징 |
|------|------|------|
| cdingram/face-swap | ~$0.013/회 | 범용 |
| codeplugtech/face-swap | 저렴 | 빠름, CPU 가능 |
| easel/advanced-face-swap | 높음 | 상업용 품질 |

**예상 비용 (1회 서비스)**
- 5장 생성 기준: $0.065 ~ $0.15
- 마진 포함 서비스 가격: $1-3 가능

### Option B: ComfyUI 워크플로우 (자체 서버)

```
[사용자] → [웹 프론트엔드] → [FastAPI 백엔드] → [ComfyUI 서버]
                                              ↓
                                         GPU 서버
                                         (RTX 4090 등)
```

**ComfyUI 파이프라인**
```
1. Load Images →
2. InsightFace 얼굴 추출 →
3. ReActor/InstantID 페이스 스왑 →
4. 고해상도 업스케일 →
5. 빈티지 스타일 적용 →
6. Save
```

**필요 모델**
- inswapper_128.onnx
- retinaface_resnet50
- codeformer.pth (얼굴 복원)

**서버 비용**
- GPU 클라우드: $1-3/시간 (RTX 4090 기준)
- 자체 서버: 초기 투자 후 운영비만

### Option C: 하이브리드

- 기본 처리: 자체 ComfyUI 서버
- 피크 타임: Replicate API로 오버플로우 처리
- 비용 최적화 가능

---

## 8. 추천 기술 스택

### MVP 단계
```
프론트엔드: Next.js / React
백엔드: FastAPI (Python)
AI 처리: Replicate API
  - Face Swap: cdingram/face-swap
  - 이미지 생성: FLUX.1-schnell
  - 스타일 변환: API 또는 후처리
스토리지: AWS S3 / Cloudflare R2
DB: PostgreSQL / Supabase
```

### 스케일업 단계
```
AI 처리: 자체 ComfyUI 서버
  - GPU: RTX 4090 또는 A100
  - 워크플로우 자동화
  - 배치 처리 지원
캐싱: Redis
CDN: Cloudflare
```

---

## 9. 기술적 챌린지 & 해결책

| 챌린지 | 해결책 |
|--------|--------|
| 오래된 저화질 사진 | 업스케일러 (Real-ESRGAN) 전처리 |
| 조명 불일치 | ControlNet Depth + 후처리 |
| 얼굴이 작거나 흐림 | GFPGAN/CodeFormer 얼굴 복원 |
| 시대별 의상 | LoRA 파인튜닝 또는 인페인팅 |
| 자연스러운 배치 | OpenPose + Depth 조합 |

---

## 10. 참고 자료

### 공식 문서 & GitHub
- [InsightFace](https://github.com/deepinsight/insightface)
- [ComfyUI ReActor](https://github.com/Gourieff/ComfyUI-ReActor)
- [ComfyUI InstantID Faceswap](https://github.com/nosiu/comfyui-instantId-faceswap)
- [FLUX on HuggingFace](https://huggingface.co/black-forest-labs/FLUX.1-schnell)

### 튜토리얼 & 가이드
- [Stable Diffusion Art - ControlNet Guide](https://stable-diffusion-art.com/controlnet/)
- [Stable Diffusion Art - Inpainting Guide](https://stable-diffusion-art.com/inpainting/)
- [Stable Diffusion Art - InstantID Guide](https://stable-diffusion-art.com/instantid/)
- [Comflowy - Face Swap Tutorial](https://www.comflowy.com/blog/face-swap)

### API 서비스
- [Replicate Face Swap Collection](https://replicate.com/collections/face-swap)
- [Replicate Pricing](https://replicate.com/pricing)

### 연구 논문
- [InfiniteYou: Flexible Photo Recrafting](https://arxiv.org/html/2503.16418v2)
- [FLUXSynID Framework](https://arxiv.org/html/2505.07530v3)

---

## 다음 단계

1. [ ] MVP 기술 스택 확정
2. [ ] Replicate API 테스트
3. [ ] ComfyUI 워크플로우 프로토타입
4. [ ] 빈티지 스타일 프리셋 개발
5. [ ] 프론트엔드 디자인
6. [ ] 베타 테스트

---

*문서 작성일: 2025-12-27*
