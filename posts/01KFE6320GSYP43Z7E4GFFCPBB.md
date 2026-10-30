---
id: "01KFE6320GSYP43Z7E4GFFCPBB"
slug: "lab-website-wordpress"
title: "연구실 홈페이지 개발: WordPress 기반 학술 웹사이트 구축"
category: "01M651PC00H447GE8N3ADKJMGZ"
tags: ["WordPress","Website","Lab Homepage","PHP","CMS"]
summary: "연세대학교 Intelligent Systems Lab 연구실 홈페이지를 WordPress 기반으로 개발했다. 연구 분야 소개, 구성원 관리, 출판물 아카이빙, 뉴스/갤러리 기능을 구현했다."
thumbnail: "/img/01KFE6320GSYP43Z7E4GFFCPBB/3405641dd68d.webp"
note: false
created: "2026-01-21T02:08:58+09:00"
updated: "2026-03-28T02:23:03+09:00"
---
![ISL 연구실 홈페이지 메인|large](/img/01KFE6320GSYP43Z7E4GFFCPBB/3405641dd68d.webp)


연세대학교 Intelligent Systems Lab(ISL) 연구실의 공식 홈페이지를 WordPress 기반으로 개발했다. 연구실 소개, 구성원 정보, 연구 분야, 출판물 관리 등의 기능을 제공한다.

## 1. 프로젝트 개요

### 1.1 개발 배경

연구실 홈페이지는 외부에 연구 성과를 알리고, 연구실 구성원 정보를 관리하며, 잠재적 대학원생들에게 연구실을 소개하는 중요한 역할을 한다. 기존에 관리가 어려웠던 홈페이지를 WordPress 기반으로 새롭게 구축하여 교수님과 학생들이 쉽게 콘텐츠를 관리할 수 있도록 했다.

### 1.2 주요 목표

- 연구실 연구 분야 및 성과 효과적 전달
- 구성원(교수, 학생, 졸업생) 정보 체계적 관리
- 출판물(논문, 특허, 저서) 카테고리별 정리
- 뉴스, 갤러리 등 활동 기록 아카이빙
- 비전공자도 쉽게 콘텐츠 업데이트 가능한 CMS 구조

## 2. 기술 스택

### 2.1 플랫폼 선택

**WordPress 6.8** 를 선택한 이유:
- 비개발자도 콘텐츠 관리 용이
- 풍부한 플러그인 생태계
- SEO 최적화 기능 내장
- 안정적인 보안 업데이트 지원

### 2.2 사용 기술

| 구분 | 기술 |
|-----|-----|
| CMS | WordPress 6.8.3 |
| 테마 | 커스텀 테마 (isl_2025) |
| 프론트엔드 | HTML5, CSS3, JavaScript |
| 캐러셀 | Swiper.js |
| 게시판 | KBoard 플러그인 |
| 분석 | Google Analytics 4 |
| 이미지 | WebP 포맷 최적화 |

## 3. 사이트 구조

### 3.1 메뉴 구성

```
Home
├── Research
│   ├── Automotive Systems
│   ├── DNN Accelerator
│   ├── Processing-in-Memory
│   └── Neuromorphic
├── Family
│   ├── Professor
│   ├── Students
│   └── Alumni
├── Publications
│   ├── Journals
│   ├── Conferences
│   ├── Patents
│   └── Books
├── Awards
├── News
├── Gallery
├── QnA
└── Contact
```

### 3.2 주요 페이지 설명

**Home**: 연구실 소개와 최신 뉴스를 캐러셀 형태로 표시. 주요 연구 분야 4개를 카드 형식으로 요약 제공.

**Research**: 4개 연구 분야별 상세 설명 페이지. 각 분야의 목표, 접근 방식, 대표 성과를 정리.

**Family**: 교수 프로필, 현재 학생 목록, 졸업생 현황을 사진과 함께 제공. 개인 홈페이지나 LinkedIn 링크 연결.

**Publications**: 논문을 Top-Tier, Major 등급으로 분류하여 표시. 연도별, 카테고리별 필터링 지원.

## 4. 주요 기능 구현

### 4.1 뉴스 캐러셀

Swiper.js를 활용한 자동 순환 뉴스 슬라이더:
- 5초 간격 자동 전환
- 좌우 네비게이션 버튼
- 페이지네이션 도트
- 터치/드래그 지원 (모바일)

### 4.2 출판물 관리 시스템

학술 출판물을 체계적으로 관리하기 위한 커스텀 포스트 타입 구현:
- 저널/컨퍼런스/특허/저서 분류
- 등급별 태그 (Top-Tier, Major 등)
- 연도별 아카이빙
- BibTeX 내보내기 기능

### 4.3 구성원 프로필

각 구성원의 정보를 카드 형식으로 표시:
- 프로필 사진 (WebP 최적화)
- 이름, 직위, 연구 분야
- 이메일, 개인 홈페이지 링크
- 졸업생의 경우 현재 소속 표시

### 4.4 반응형 디자인

모바일, 태블릿, 데스크톱 모든 환경에서 최적화된 레이아웃:
- CSS Media Query 활용
- 모바일 햄버거 메뉴
- 이미지 지연 로딩 (Lazy Loading)
- 터치 친화적 UI

## 5. SEO 및 성능 최적화

### 5.1 SEO 최적화

- Schema.org 마크업 적용 (Organization, Person, Article)
- Open Graph 메타 태그
- XML 사이트맵 자동 생성
- 검색엔진 친화적 URL 구조

### 5.2 성능 최적화

- 이미지 WebP 변환으로 용량 60% 감소
- CSS/JS 파일 압축 및 캐싱
- CDN 활용한 정적 리소스 제공
- 데이터베이스 쿼리 최적화

## 6. 관리 기능

### 6.1 콘텐츠 관리

WordPress 관리자 대시보드를 통해:
- 뉴스/공지 작성 및 수정
- 구성원 정보 업데이트
- 출판물 추가/수정
- 갤러리 사진 업로드

### 6.2 접근 권한 관리

역할별 접근 권한 설정:
- 관리자: 전체 기능 접근
- 편집자: 콘텐츠 작성/수정
- 구성원: 본인 프로필 수정

## 7. 결론

WordPress 기반으로 연구실 홈페이지를 구축함으로써, 비개발자인 연구실 구성원들도 쉽게 콘텐츠를 관리할 수 있게 되었다. 특히 출판물 관리 시스템과 구성원 프로필 기능은 학술 연구실의 특성에 맞게 커스터마이징했다. 향후 다국어 지원(영어/한국어)과 논문 인용 통계 연동 기능을 추가할 예정이다.
