# Search Server API 명세 (Frontend)

- 대상: `/api/v1/search/**`
- 기준일: 2026-04-16
- 변환 소스: `docs/searchserver-api-spec-frontend.html` / `docs/searchserver-api-spec-frontend.pdf`

## 1. 공통 응답 포맷

```json
{
  "status": "success",
  "totalElements": 100,
  "totalPages": 9,
  "currentPage": 0,
  "size": 12,
  "isFirst": true,
  "isLast": false,
  "hasNext": true,
  "hasPrevious": false,
  "extra": {},
  "data": []
}
```

인증은 게이트웨이 정책을 따르며, 보호 경로에서는 `Authorization` 헤더가 필요할 수 있습니다.

## 2. 상품 검색

### `GET /api/v1/search/products`

Query:
- `title`: 상품명 검색어
- `keyword`: 보조 검색어
- `category`: 대카테고리 (기본 `ALL`)
- `subCategory`: 소카테고리
- `sortType`: 최신순 / 판매량순 / 가격 높은순 / 가격 낮은순
- `page`: 페이지 (기본 `0`)

`data[]` 필드:
- `id`
- `imageUrl`
- `productTitle`
- `originalPrice`
- `price`
- `discountRate`
- `discountTag`
- `isNew`
- `productTag`
- `productUrl`
- `category`

## 3. 정기배송 상품

### `GET /api/v1/search/products/subscription`

Query:
- `page`: 페이지

`data[]` 필드:
- `id`
- `imageUrl`
- `productTitle`
- `price`
- `productUrl`

## 4. 베스트셀러

### `GET /api/v1/search/products/bestseller`

설명:
- 현재 구현은 검색 랭킹 기반 (실판매량 집계 아님)

`data[]` 필드:
- `id`
- `imageUrl`
- `productTitle`
- `price`
- `salesRank`
- `rankTag`
- `productUrl`

## 5. 홈 베스트셀러

### `GET /api/v1/search/products/home-bestseller`

Query:
- `size`: 노출 개수 (기본 `3`)

`data[]` 필드:
- `rank`
- `id`
- `imageUrl`
- `productTitle`
- `price`
- `score`
- `salesCount`
- `createdAt`
- `productUrl`

## 6. 유사 상품 추천 (상세 페이지용)

### `GET /api/v1/search/products/{productId}/similar`

Path/Query:
- `productId`: 기준 상품 ID
- `size`: 반환 개수 (기본 `3`)

`data[]` 필드 (슬림 DTO):
- `productId`
- `imageUrl`
- `title`
- `tags`
- `price`

`tags` 규칙:
- 다중 태그 배열
- 우선순위: `[NEW]` 먼저, 그다음 `[판매 1위/2위/3위]`

예시:

```json
{
  "productId": 185,
  "imageUrl": "https://...",
  "title": "오독오독 사료",
  "tags": ["[NEW]", "[판매 2위]"],
  "price": 12900
}
```

## 7. 자동완성

### `GET /api/v1/search/products/autocomplete`

Query:
- `name`: 자동완성 입력 문자열

응답:
- 배열 각 항목 `id`, `title`

## 8. 인기 검색어

### `GET /api/v1/search/products/trending`

응답 배열 각 항목:
- `rank`
- `keyword`
- `score`

## 9. 리뷰 검색

### `GET /api/v1/search/reviews`

Query:
- `productId`: 상품 ID 필터
- `keyword`: 리뷰 내용/작성자 검색어
- `sortType`: `BEST` 또는 최신순
- `reviewType`: `ALL` / `PHOTO` / `IMAGE` / `VIDEO` / `TEXT`
- `page`, `size`: 페이징 (기본 `0`, `5`)

## 10. 공지 검색

### `GET /api/v1/search/notices`

Query:
- `searchRange`: 전체 / 일주일 / 한달 / 세달
- `searchType`: 제목 등
- `keyword`: 검색어
- `page`, `size`: 페이징

## 11. 프론트 적용 메모

- 상세의 "이런 제품은 어때요?"는 `/api/v1/search/products/{productId}/similar?size=3` 고정 사용 권장
- UI 태그는 `tags[]` 순서를 그대로 렌더링하면 우선순위 정책이 자동 반영됨
