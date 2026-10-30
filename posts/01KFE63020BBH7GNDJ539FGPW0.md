---
id: "01KFE63020BBH7GNDJ539FGPW0"
slug: "django-tech-blog"
title: "개인 기술 블로그 개발: Django 기반 풀스택 웹 애플리케이션"
category: "01M651PC00H447GE8N3ADKJMGZ"
tags: ["Django","Blog","Python","PostgreSQL","Full-Stack"]
summary: "Django와 순수 JavaScript/CSS로 개인 기술 블로그를 개발했다. 마크다운 기반 글 작성, 계층적 카테고리, AI 기능 연동(Gemini), SEO 최적화 등의 기능을 직접 구현했다."
thumbnail: "/img/01KFE63020BBH7GNDJ539FGPW0/a9d6ea3295d8.webp"
note: false
created: "2026-01-21T02:08:56+09:00"
updated: "2026-03-28T02:23:03+09:00"
---
![개인 기술 블로그 메인|large](/img/01KFE63020BBH7GNDJ539FGPW0/a9d6ea3295d8.webp)


Django와 순수 JavaScript/CSS로 개발한 개인 기술 블로그이다. 마크다운 기반 글 작성, 카테고리 관리, AI 기능 연동 등을 직접 구현했다.

## 1. 프로젝트 개요

### 1.1 개발 동기

기존 블로그 플랫폼(Tistory, Velog 등)의 제약에서 벗어나 완전히 커스터마이징 가능한 개인 블로그를 만들고자 했다. 직접 개발함으로써 원하는 기능을 자유롭게 추가하고, 웹 개발 전반에 대한 이해도를 높일 수 있었다.

### 1.2 주요 목표

- 마크다운 기반 기술 블로그 글 작성
- 계층적 카테고리 시스템
- 코드 하이라이팅 및 수식 렌더링
- SEO 최적화
- AI 기능 연동 (요약, 번역 등)
- 관리자 대시보드

## 2. 기술 스택

### 2.1 백엔드

| 구분 | 기술 |
|-----|-----|
| 프레임워크 | Django 4.2+ |
| 데이터베이스 | PostgreSQL |
| 인증 | django-allauth |
| 보안 | django-recaptcha, django-turnstile |
| 이미지 처리 | Pillow |
| 마크다운 | python-markdown, mdx-truly-sane-lists |
| AI | Google Generative AI (Gemini) |

### 2.2 프론트엔드

- 순수 JavaScript (프레임워크 미사용)
- CSS3 (SCSS 미사용, 순수 CSS)
- 코드 하이라이팅: Prism.js
- 수식 렌더링: KaTeX
- 아이콘: Font Awesome

### 2.3 인프라

- 서버: Oracle Cloud (ARM 인스턴스)
- 웹서버: Nginx
- 컨테이너: Docker, Docker Compose
- SSL: Let's Encrypt

## 3. 주요 기능

### 3.1 글 관리 시스템

**마크다운 에디터**
- 실시간 미리보기
- 이미지 드래그 앤 드롭 업로드
- 코드 블록 언어 자동 감지
- 수식 입력 지원 (KaTeX)

**카테고리 시스템**
- 2단계 계층 구조 (대분류 > 소분류)
- 카테고리별 글 목록
- 카테고리 아이콘 지정

**글 상태 관리**
- Draft(임시저장) / Published(발행) 상태
- 예약 발행 기능
- 조회수 카운팅

### 3.2 검색 및 필터링

- 제목/내용 전문 검색
- 카테고리 필터
- 태그 기반 필터링
- 최신순/인기순 정렬

### 3.3 AI 기능 연동

Google Gemini API를 활용한 AI 기능:
- 글 요약 자동 생성
- 영어 번역
- 관련 글 추천
- 오타/문법 검사

### 3.4 SEO 최적화

- 동적 메타 태그 생성
- Open Graph / Twitter Card
- 구조화된 데이터 (JSON-LD)
- XML 사이트맵 자동 생성
- robots.txt 관리
- Google Search Console 연동

### 3.5 기타 페이지

**포트폴리오**: 프로젝트 목록 및 상세 설명
**방명록**: 방문자 메시지 기능
**About**: 자기소개 페이지
**Contact**: 연락처 및 문의 폼

## 4. 시스템 구조

### 4.1 Django 앱 구조

```
system-blog/
├── config/           # 프로젝트 설정
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── blog/             # 메인 블로그 앱
│   ├── models.py     # Post, Category, Comment 모델
│   ├── views/
│   │   ├── public.py   # 공개 페이지 뷰
│   │   ├── edit.py     # 글 작성/수정 뷰
│   │   ├── ai.py       # AI 기능 뷰
│   │   └── category.py # 카테고리 관리
│   ├── templates/
│   └── static/
├── templates/        # 전역 템플릿
├── static/           # 정적 파일
└── media/            # 업로드 파일
```

### 4.2 데이터베이스 모델

**Post 모델 주요 필드**
- title, slug, content
- category (FK)
- tags (ManyToMany)
- is_draft, view_count
- created_at, updated_at
- summary, summary_en (AI 생성)

**Category 모델**
- id (문자열 PK, 예: "010100")
- name, parent
- icon, description

### 4.3 URL 구조

```
/                     # 홈
/post/<slug>/         # 글 상세
/category/<id>/       # 카테고리별 목록
/search/              # 검색
/about/               # 소개
/portfolio/           # 포트폴리오
/guestbook/           # 방명록
/edit/                # 글 작성 (관리자)
```

## 5. 구현 세부사항

### 5.1 마크다운 렌더링

python-markdown 라이브러리에 확장 기능 추가:
- 테이블 지원
- 코드 펜스 (언어 지정)
- 각주
- 목차 자동 생성
- 이미지 크기 조절 문법

### 5.2 이미지 처리

- 업로드 시 WebP 자동 변환
- 썸네일 자동 생성
- 지연 로딩 (Lazy Loading)
- CDN 연동 가능 구조

### 5.3 보안

- CSRF 보호
- XSS 방지 (마크다운 필터링)
- SQL Injection 방지 (Django ORM)
- Rate Limiting
- Cloudflare Turnstile (봇 방지)

### 5.4 성능 최적화

- 데이터베이스 쿼리 최적화 (select_related, prefetch_related)
- 정적 파일 캐싱
- Gzip 압축
- 이미지 최적화

## 6. 배포 환경

### 6.1 Docker 구성

```yaml
services:
  web:
    build: .
    volumes:
      - ./:/app
    environment:
      - DJANGO_SETTINGS_MODULE=config.settings
  
  nginx:
    image: nginx:alpine
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./static:/static
      - ./media:/media
    ports:
      - "80:80"
      - "443:443"
  
  db:
    image: postgres:15
    volumes:
      - pgdata:/var/lib/postgresql/data
```

### 6.2 CI/CD

- GitHub 저장소 연동
- Push 시 자동 배포 스크립트
- 데이터베이스 백업 자동화

## 7. 향후 계획

- 댓글 시스템 개선 (대댓글 지원)
- 다크 모드 지원
- PWA 적용
- 검색 엔진 개선 (Elasticsearch)
- 뉴스레터 구독 기능
- 통계 대시보드 고도화

## 8. 결론

Django와 순수 JavaScript로 블로그를 직접 개발함으로써, 웹 개발의 전체 과정을 경험할 수 있었다. 프레임워크에 의존하지 않고 기본기를 다지는 것이 목표였으며, 필요한 기능을 직접 구현하면서 깊은 이해를 얻을 수 있었다. 현재 300개 이상의 기술 글을 작성하며 운영 중이다.
