---
layout: post
title: "마크다운 언어 문법"
date: 2026-01-09 00:59:00 +0900
categories: [카테고리1, 카테고리2]
tags: [태그1, 태그2]
excerpt: "포스트 미리보기 문구"
published: false # 공개 여부
---
# H1 제목
## H2 제목
### H3 제목

**굵게** 또는 __굵게__
*기울게* 또는 _기울게_
~~취소선~~
- 순서 없는 리스트
    - 중첩 리스트
1. 순서 있는 리스트
    1. 중첩 번호

인라인 코드: `code`
코드 블록:
```python
print("Hello, World!")
```


## 커밋 및 배포
작성 후 `git add .`, `git commit -m "첫 포스팅 추가"`, `git push`를 실행하세요. 1-2분 후 `https://mhs442.github.io`에서 확인 가능합니다. 테마가 Tale이므로 포스트 목록과 태그 페이지가 자동 생성됩니다.[1]


![대체 텍스트](https://example.com/image.jpg)

또는 크기 조절:
<img src="https://example.com/image.jpg" width="50%" alt="대체 텍스트">
<img src="https://example.com/image.jpg" width="300" height="200">


![이미지](./assets/images/photo.jpg)


| 기능    | 마크다운 | HTML  |
|---------|:---------:|-------|
| 이미지  | ![ ]( )  | `<img>` |
| 표      | \| 테이블 \| | `<table>` |
| 인용    | > 인용   | `<blockquote>` |


| 왼쪽 | 가운데 | 오른쪽 |
|:-----|:-------:|-------:|
| A    |    B    |    C   |


> 이것은 인용구입니다.
> 여러 줄도 됩니다.

다중 레벨 인용:
> > 중첩 인용구
>
> 기본 인용구로 복귀


