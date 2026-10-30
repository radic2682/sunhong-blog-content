---
id: "01HHG7J800QE02933THHWY5WDS"
slug: "github-pages-jekyll-blog"
title: "GitHub Pages + Jekyll 블로그 만들기: 저장소 생성부터 자동 배포까지"
category: "01M651PC00H447GE8N3ADKJMGZ"
tags: ["HTML","CSS","JS","GIT","Jekyll","SASS"]
summary: "GitHub Pages + Jekyll로 첫 블로그를 만들었던 기록입니다. 개발자라면 한 번쯤 거쳐가는 관문인 GitHub Blog 구축 여정을 정리합니다. 왜 GitHub Blog인가 개발자 포트폴리오로서 블로그를 운영하려면 다음 조건이 중요합니다: - 버전 관리로..."
thumbnail: "/img/01HHG7J800QE02933THHWY5WDS/73e9f0186a81.webp"
note: false
created: "2023-12-13T09:00:00+09:00"
updated: "2026-09-15T02:10:29+09:00"
---
![GitHub Blog|large](/img/01HHG7J800QE02933THHWY5WDS/73e9f0186a81.webp)

GitHub Pages + Jekyll로 첫 블로그를 만들었던 기록입니다. 개발자라면 한 번쯤 거쳐가는 관문인 GitHub Blog 구축 여정을 정리합니다.

## 왜 GitHub Blog인가

개발자 포트폴리오로서 블로그를 운영하려면 다음 조건이 중요합니다:
- 버전 관리로 글 히스토리 추적
- Markdown으로 빠른 작성
- 코드 하이라이팅 기본 지원
- 무료 호스팅
- 커스터마이징 가능

GitHub Pages는 이 모든 조건을 만족합니다. 개인 저장소에 정적 HTML을 push하면 자동으로 배포되고, Jekyll을 연동하면 Markdown을 HTML로 자동 변환해줍니다.

## 구축 과정

### 1. 저장소 생성
`username.github.io` 형식의 저장소를 만듭니다. 이 특별한 이름의 저장소는 GitHub Pages에서 루트 도메인으로 자동 서빙됩니다.

### 2. Jekyll 테마 선택
처음부터 만들지 않고 오픈소스 테마를 포크해서 시작하는 게 실용적입니다. 저는 `minimal-mistakes` 테마를 기반으로 커스터마이징했습니다. 기능이 풍부하고 문서화가 잘 되어 있습니다.

### 3. `_config.yml` 설정
사이트 전역 설정을 작성합니다:
- 사이트 제목, 설명, 저자 정보
- 네비게이션 메뉴
- 소셜 링크 (GitHub, LinkedIn 등)
- 플러그인 활성화

### 4. `_posts/` 디렉토리에 글 작성
파일명 형식은 `YYYY-MM-DD-title.md` 필수입니다. 각 글 상단에는 front matter로 메타데이터 지정:

```yaml
---
title: "글 제목"
date: 2023-12-27
categories: [Category1]
tags: [tag1, tag2]
---
```

### 5. 로컬 프리뷰
`bundle exec jekyll serve`로 로컬에서 실시간 프리뷰 가능. 수정하면 자동 리빌드됩니다.

### 6. Push → 자동 배포
`git push` 하면 GitHub Actions가 빌드해서 배포합니다. 몇 분 안에 전세계에서 접근 가능.

## 겪었던 문제들

### 카테고리/태그 페이지가 안 생김
Jekyll의 `jekyll-archives` 플러그인은 GitHub Pages 기본 빌드에서 지원되지 않습니다. 해결 방법은 직접 HTML 파일을 만들거나 GitHub Actions로 커스텀 빌드를 하는 것.

### 이미지 로딩 느림
GitHub Pages의 이미지 서빙이 CDN 경유지만 느린 편. 외부 CDN(jsDelivr)으로 우회하면 체감상 훨씬 빨라집니다.

### 댓글 시스템
정적 사이트라 DB가 없어서 댓글을 달려면 외부 서비스가 필요. `utterances`(GitHub Issues 기반)를 도입해서 해결했습니다.

### SEO
Jekyll이 기본 제공하는 sitemap/robots.txt가 있지만 Google Search Console에 직접 등록 필수. `jekyll-seo-tag` 플러그인으로 메타 태그 자동 생성.

## 장단점

**장점:**
- 완전 무료
- 마크다운으로 빠른 작성
- Git으로 히스토리 관리
- 개발자에게 익숙한 워크플로우

**단점:**
- DB가 없어 동적 기능 제약 (댓글, 조회수 등)
- Jekyll 빌드 시간이 누적되면 느려짐
- 복잡한 플러그인은 사용 불가 (GitHub Pages 화이트리스트)

## 이후 이전

이 블로그로 1년 정도 운영한 후, 더 많은 기능(조회수, 검색, 댓글, 관리 대시보드 등)이 필요해져서 **Django 기반 자체 블로그**(현재 sunhong.me)로 이전했습니다. 정적 사이트의 한계를 느끼면서 동적 서비스의 장점을 깨달은 계기였습니다.

GitHub Blog는 개발자 입문용으로는 정말 훌륭하지만, 진지하게 블로그를 운영하고 싶다면 결국 자체 서버 + 프레임워크로 가게 되는 것 같습니다.
