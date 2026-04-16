# Product Server API Spec (Frontend)

- 대상 서버: `productserver`
- 기준 경로: `/api/v1/product/**` (+ 이미지 리다이렉트 `/product/main/**`)
- 기준일: 2026-04-16

## 1. 공통 응답 형식

대부분 API는 아래 래퍼를 사용합니다.

```json
{
  "message": "성공 메시지",
  "status": 200,
  "data": {}
}
```

예외:
- 카테고리 API(`GET/POST/PUT/DELETE /api/v1/product/categories...`)는 `Result` 래퍼 없이 본문을 직접 반환합니다.

## 2. 상품 목록 조회

### `GET /api/v1/product/products`

Query:
- `categoryId` (optional, number): 카테고리 필터
- `isSubscribable` (optional, boolean): 정기배송 가능 상품 필터
- `page` (optional, number, default `0`)
- `size` (optional, number, default `10`)
- `sort` (optional, string, 예: `createdDate,desc`)

Response: `Result<Page<ResProductListDto>>`

`data.content[]` 필드:
- `productId`
- `productName`
- `productUrl`
- `price`
- `discountPrice`
- `priceDisplay`
- `mainImageUrl`
- `tags`
- `isSubscribable`
- `stockQuantity`
- `stockStatus`

## 3. 상품 상세 조회

### `GET /api/v1/product/{productId}`

Path:
- `productId` (required, number)

Response: `Result<ResProductDetail>`

주요 `data` 필드:
- 기본: `productId`, `productName`, `categoryName`, `brandName`, `brandId`, `content`
- 가격: `price`, `discountPrice`, `priceDisplay`, `rewardRate`
- 할인정책: `tier1Quantity`, `tier1Rate`, `tier2Quantity`, `tier2Rate`
- 노출/검색: `status`, `tags`, `keywords`, `salesCount`
- 배송/구독/재고: `isSubscribable`, `deliveryFee`, `deliveryMethod`, `stockQuantity`, `stockStatus`
- 미디어/옵션: `imageUrls[]`, `options[]`

## 4. 상품 옵션 조회

### `GET /api/v1/product/{productId}/options`

Path:
- `productId` (required, number)

Response: `Result<List<ResProductOptionDto>>`

`data[]` 필드:
- `optionId`
- `optionName`
- `extraPrice`
- `stockQuantity`
- `stockStatus`

## 5. 상품 등록 (판매자)

### `POST /api/v1/product/seller`

Content-Type:
- `multipart/form-data`

Parts:
- `data` (required, string): `ProductSaveRequest` JSON 문자열
- `files` (optional, file[]): 이미지 파일 배열

`data` JSON 구조:
- `categoryId`: number
- `productSaveDto`: object
- `imageFileSaveDtoList`: array (선택)

`productSaveDto` 주요 필드:
- `productName`, `content`, `productUrl`, `brandName`, `brandId`
- `price`, `discountPrice`, `rewardRate`
- `tier1Quantity`, `tier1Rate`, `tier2Quantity`, `tier2Rate`
- `stockQuantity`, `status`, `tags`, `keywords`
- `isSubscribable`, `deliveryFee`, `deliveryMethod`
- `options[]` (`optionName`, `extraPrice`, `initialStock`)

Response:
- HTTP `201`
- `Result<ResProductSaveDto>`

## 6. 상품 수정 (판매자)

### `PUT /api/v1/product/seller/{productId}`

Content-Type:
- `multipart/form-data`

Path:
- `productId` (required, number)

Parts:
- `data` (required, string): `ProductUpdateRequest` JSON 문자열
- `files` (optional, file[]): 새 이미지 파일 배열

`data` JSON 구조:
- `categoryId`: number
- `productUpdateDto`: object
- `imageFileSaveDtoList`: array (선택)

`productUpdateDto` 주요 필드:
- `productId` (서버에서 path 값으로 세팅)
- `categoryId`, `productName`, `content`
- `productUrl`, `brandName`, `brandId`
- `price`, `discountPrice`, `rewardRate`
- `tier1Quantity`, `tier1Rate`, `tier2Quantity`, `tier2Rate`
- `status`, `tags`, `keywords`
- `isSubscribable`, `deliveryFee`, `deliveryMethod`
- `options[]` (`optionId`, `optionName`, `extraPrice`)

Response:
- HTTP `200`
- `Result<ResProductUpdateDto>`

## 7. 카테고리 API

### `GET /api/v1/product/categories`
- Response: `List<ResCategoryListDto>`

`ResCategoryListDto`:
- `categoryId`, `name`, `displayOrder`, `children[]`

### `POST /api/v1/product/categories`
- Body: `CategorySaveDto`
- Response: HTTP `201`, `ResCategoryDto`

`CategorySaveDto`:
- `categoryName` (required)
- `parentId` (optional, 대분류면 `null`)
- `displayOrder` (optional, default `0`)

### `PUT /api/v1/product/categories/{categoryId}`
- Body: `CategoryUpdateDto`
- Response: `ResCategoryDto`

### `DELETE /api/v1/product/categories/{categoryId}`
- Response: HTTP `204 No Content`

## 8. 홈 콘텐츠 API

### `GET /api/v1/product/home/banners`
- Response: `Result<List<BannerResponse>>`

`BannerResponse`:
- `bannerId`, `imageUrl`, `displayOrder`, `originalFilename`

### `GET /api/v1/product/home/brand-stories`
- Response: `Result<List<BrandStoryResponse>>`

`BrandStoryResponse`:
- `storyId`, `imageUrl`, `displayOrder`, `originalFilename`

### `GET /api/v1/product/home/section-titles`
- Response: `Result<List<HomeSectionTitleResponse>>`

`HomeSectionTitleResponse`:
- `sectionKey`, `title`, `displayOrder`

## 9. 메인 이미지 리다이렉트 API

### `GET /product/main/banner/{bannerId}`
### `GET /product/main/brand-story/{storyId}`

- Response: HTTP `302 Found`
- `Location` 헤더로 원본 이미지 URL 리다이렉트

## 10. 체크아웃 검증 API

### `POST /api/v1/product/checkout/validate`

Body: `CheckoutValidationRequest`

```json
{
  "items": [
    { "productId": 185, "optionId": 10, "quantity": 2 }
  ]
}
```

호환 alias:
- `productId` 대신 `product_id`, `itemId`, `item_id` 허용
- `optionId` 대신 `option_id` 허용
- `quantity` 대신 `amount` 허용

Response: `Result<CheckoutValidationResponse>`

`data`:
- `items[]` (`product_id`, `option_id`, `product_name`, `option_name`, `price`, `discountPrice`, `extraPrice`, `quantity`, `lineTotalPrice`, `capturedAt`)
- `totalPrice`
- `capturedAt`

## 11. 할인 정책 조회 API

### `GET /api/v1/product/config/discount-policy`

Response: `Result<Map<String,Object>>`

`data` 예:
- `bulk`: `tier1Quantity`, `tier1Rate`, `tier2Quantity`, `tier2Rate`
- `subscription`: `discountType`, `currency`, `oneWeekDiscountAmount`, `twoWeekDiscountAmount`, `oneMonthDiscountAmount`, `twoMonthDiscountAmount`
- `bargain`: `enabled`, `discountRate`
- `status`

## 12. 프론트 참고

- 등록/수정 API의 `data`는 multipart 파트 내 "문자열 JSON"입니다.
- 내부 API(`/internal/v1/products/**`)는 프론트 명세에서 제외했습니다.
