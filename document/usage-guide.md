# mall — Usage Guide

## Accessing the Application

| Interface | URL |
|---|---|
| **Admin Panel (frontend UI)** | http://localhost:8083 |
| Admin API (Swagger) | http://localhost:8086/swagger-ui.html |
| Portal API (Swagger) | http://localhost:8082/swagger-ui.html |
| Search API (Swagger) | http://localhost:8087/swagger-ui.html |

**Login credentials:** Username `test`, Password `123456`

---

## 1. Admin Panel Overview

The admin panel (mall-admin-web) is a Vue 3 SPA that manages the entire e-commerce backend. It is organized into the following modules accessible from the sidebar:

| Menu | Description |
|---|---|
| **Home** | Dashboard with statistics |
| **Product** | Product catalog, brands, categories, attributes, SKU |
| **Order** | Orders, returns, settings |
| **Marketing** | Coupons, flash sales, home page recommendations |
| **Content** | Topics, preference areas |
| **User** | Admins, roles, menus, resources, member levels |
| **System** | Company addresses, order settings |

---

## 2. Product Management

### 2.1 Product Categories

**Path:** Product → Category

Before creating products, set up the category hierarchy:

- **Create top-level category:** Click "Add", enter name, set navigation/show status, choose icon and banner images
- **Create sub-category:** Select a parent category, then add child with the same form
- **Drag to reorder:** Use the tree view to rearrange categories
- **Set filters:** For each category, configure which attributes (specs) appear as filters on the storefront

**Tree structure:** The admin panel provides `GET /productCategory/list/{parentId}` for paginated children and `GET /productCategory/list/withChildren` for the complete tree.

### 2.2 Brands

**Path:** Product → Brand

| Action | Endpoint |
|---|---|
| List all | `GET /brand/list?keyword=&pageNum=1&pageSize=10` |
| Create | `POST /brand/create` with `{name, logo, description, showStatus, ...}` |
| Update | `POST /brand/update/{id}` |
| Delete | `GET /brand/delete/{id}` or `POST /brand/delete/batch` |
| Toggle display | `POST /brand/update/showStatus` |

Brands feed the storefront's brand recommendation section and product detail pages.

### 2.3 Product Attributes

**Path:** Product → Product Attributes

Two types of attributes:

- **Specifications (type=0):** Drive SKU variants (e.g., "Color: Red, Blue", "Size: S, M, L")
- **Parameters (type=1):** Searchable product features (e.g., "Screen Size: 6.1 inch", "RAM: 8GB")

**Workflow:**
1. Create an **attribute category** (e.g., "Phone Specs", "Clothing Specs")
2. Add attributes to the category, specifying:
   - Name, type (spec/param), input type (manual select/list)
   - Selectable values list
   - Whether it supports filtering on storefront
3. When creating a product, attributes are auto-assigned based on the product's category

### 2.4 SKU Stock

**Path:** Product → SKU Stock

For each product, manages inventory at the SKU level:

| Endpoint | Usage |
|---|---|
| `GET /sku/{productId}?keyword=` | Search SKUs by product and optional code |
| `POST /sku/update/{productId}` | Batch update stock quantities, prices, codes |

Each SKU has: `price`, `stock`, `lowStock` (warning threshold), `pic`, `sale` (sales count), `promotionPrice`, `lockStock` (reserved for pending orders).

### 2.5 Products (Core)

**Path:** Product → Product List

**Creating a product** (`POST /product/create`):

The product form is a multi-section page:

1. **Basic Info:** Name, brand, category, product SN, album images, description
2. **Promotion:** Choose from no promotion, flash sale, discount, full-reduction, ladder pricing
3. **Attributes:** Spec and param values (filtered by product category)
4. **SKU:** Auto-generated grid from spec combinations — set price, stock, SKU code per variant
5. **Relations:** Link to subjects and preference areas for storefront discovery

**Fields in `PmsProductParam`:**

```
name, brandId, productCategoryId, productSn, price, stock,
publishStatus, newStatus, recommendStatus, verifyStatus, sort,
description, albumPics, detailTitle, detailDesc, productDetailContent,
keywords, note, unit, weight, serviceIds, promotionType,
giftPoint, giftGrowth, usePointLimit,
productLadderList,              -- ladder pricing tiers
productFullReductionList,       -- full-reduction rules
memberPriceList,                -- member tier pricing
skuStockList,                   -- SKU variants
productAttributeValueList,      -- spec/param values
subjectProductRelationList,     -- linked subjects
prefrenceAreaProductRelationList -- linked preference areas
```

**Listing products** (`GET /product/list`):

| Parameter | Filter |
|---|---|
| `keyword` | Search by name or product SN |
| `productCategoryId` | Filter by category |
| `brandId` | Filter by brand |
| `publishStatus` | 0=unpublished, 1=published |
| `verifyStatus` | 0=unverified, 1=verified |

**Batch operations:**

| Button | Endpoint | Effect |
|---|---|---|
| Publish/Unpublish | `POST /product/update/publishStatus` | Toggle storefront visibility |
| Verify | `POST /product/update/verifyStatus` | Approve products with note |
| Recommend | `POST /product/update/recommendStatus` | Mark for recommendation |
| New | `POST /product/update/newStatus` | Mark as new arrival |
| Delete | `POST /product/update/deleteStatus` | Soft delete (move to recycle) |

---

## 3. Order Management

### 3.1 Order Lifecycle

```
Pending Payment (0)
    ├── User pays (Alipay) → Pending Shipment (1)
    │     ├── Admin ships → Shipped (2)
    │     │     └── User confirms → Completed (3)
    │     └── Admin cancels → Closed (4)
    ├── Timeout (30 min) → Closed (4)
    └── User cancels → Closed (4)
```

### 3.2 Viewing Orders

**Path:** Order → Order List (`GET /order/list`)

| Filter | Description |
|---|---|
| `orderSn` | Exact order number |
| `receiverKeyword` | Recipient name or phone |
| `status` | -1=all, 0=unpaid, 1=unshipped, 2=shipped, 3=completed, 4=closed |
| `orderType` | Normal vs flash sale |
| `sourceType` | PC vs mobile |
| `createTime` | Date range |

**Order detail** (`GET /order/{id}`) shows:
- Order items (product name, quantity, price)
- Payment info (total, freight, discount, pay type)
- Recipient info (name, phone, address)
- Operation history (timeline of status changes)
- Invoice info (if applicable)

### 3.3 Order Operations

| Action | Endpoint | Description |
|---|---|---|
| **Ship** | `POST /order/update/delivery` | Batch set delivery company and tracking number |
| **Close** | `POST /order/update/close` | Force close an order (with reason note) |
| **Delete** | `POST /order/delete` | Remove from admin view (soft delete) |
| **Edit recipient** | `POST /order/update/receiverInfo` | Change shipping address |
| **Edit amounts** | `POST /order/update/moneyInfo` | Adjust freight or discount |
| **Add note** | `POST /order/update/note` | Internal note with status override |

### 3.4 Return Requests

**Path:** Order → Return Request

Storefront customers submit return requests which admins process:

| Endpoint | Description |
|---|---|
| `GET /returnApply/list` | List with filters: status, order SN, date range |
| `GET /returnApply/{id}` | Detail including product, reason, images |
| `POST /returnApply/update/status/{id}` | Approve/decline, set return amount, note |

**Return statuses:** 0=pending, 1=approved, 2=rejected

### 3.5 Order Settings

**Path:** Order → Order Settings (`/orderSetting`)

| Setting | Description |
|---|---|
| `normalOrderOvertime` | Minutes before unpaid normal order auto-cancels |
| `flashOrderOvertime` | Minutes before unpaid flash order auto-cancels |
| `confirmOvertime` | Days after shipment before auto-confirmation |
| `finishOvertime` | Days after completion before order closes |
| `commentOvertime` | Days after completion to leave a review |

### 3.6 Return Reasons

**Path:** Order → Return Reasons

Manage the list of reasons customers can select when submitting a return request (e.g., "Wrong item", "Quality issue", "No longer needed").

---

## 4. Marketing & Promotion

### 4.1 Coupons

**Path:** Marketing → Coupons

**Creating a coupon** (`POST /coupon/create`):

| Field | Options |
|---|---|
| Type | Full-reduction, fixed discount, or voucher |
| Name | Display name |
| Total count | Number available |
| Amount | Discount value |
| Min point | Minimum order amount |
| Per limit | Max per customer |
| Use type | All products, specific categories, specific products |
| Platform | All, PC, mobile |
| Start/End | Validity period |

**Listing:** `GET /coupon/list?name=&type=&pageNum=&pageSize=`

Coupon history (claims) is at `GET /couponHistory/list?couponId=&useStatus=`

### 4.2 Flash Sales (限时购)

**Path:** Marketing → Flash Sales

Flash sales are time-limited promotions with steep discounts.

**Setup involves three steps:**

#### Step 1: Flash Promotion (Activity)
`POST /flash/create`

| Field | Description |
|---|---|
| Title | e.g., "Midnight Flash Sale" |
| Start/End date | Activity period |
| Status | Enable/disable |

#### Step 2: Flash Sessions (Time Slots)
`POST /flashSession/create`

Define daily recurring time slots (e.g., 10:00-12:00, 14:00-16:00, 20:00-22:00). Each session shows the product count when displayed on the storefront (`GET /flashSession/selectList?flashPromotionId=`).

#### Step 3: Product Relations
`POST /flashProductRelation/create`

Link specific products with their flash sale price and quantity to a session:

| Field | Description |
|---|---|
| `flashPromotionId` | Activity ID |
| `flashPromotionSessionId` | Time slot ID |
| `productId` | Product to discount |
| `flashPromotionPrice` | Flash sale price |
| `flashPromotionCount` | Total quantity available |
| `flashPromotionLimit` | Max per customer |

### 4.3 Home Page Recommendations

**Path:** Marketing → Home Page

The storefront home page is composed of manually curated sections. Each has its own management interface:

| Section | Controller | What it controls |
|---|---|---|
| **Banner ads** | `SmsHomeAdvertiseController` | Carousel images with click-through URLs, start/end times |
| **Recommended brands** | `SmsHomeBrandController` | Brands with sort order and recommend status |
| **New products** | `SmsHomeNewProductController` | Products tagged as "new arrival" |
| **Popular products** | `SmsHomeRecommendProductController` | Products tagged as "recommended" |
| **Featured topics** | `SmsHomeRecommendSubjectController` | Themed collections of products |

All of these follow the same pattern:
- **Add:** Select items from a picker dialog
- **Sort:** Set numeric sort order per item
- **Remove:** Batch delete items
- **Toggle:** Enable/disable recommendation status

---

## 5. Content Management

### 5.1 Topics (Subjects)

**Path:** Content → Topics

Themed collections of products (e.g., "Summer Essentials", "Tech Deals"). Managed via:

| Endpoint | Description |
|---|---|
| `GET /subject/listAll` | All subjects |
| `GET /subject/list?keyword=&pageNum=&pageSize=` | Paginated search |

### 5.2 Preference Areas

**Path:** Content → Preference Areas

Pre-defined browsing sections on the storefront (e.g., "For You", "Trending Now"). Currently a read-only list (`GET /prefrenceArea/listAll`). Products are linked to preference areas during product creation.

---

## 6. User & Permission Management

### 6.1 Admin Users

**Path:** User → Users

| Endpoint | Description |
|---|---|
| `POST /admin/register` | Create new admin |
| `POST /admin/login` | Login (returns JWT) |
| `GET /admin/info` | Current user profile + menus + roles |
| `GET /admin/list?keyword=&pageNum=&pageSize=` | List admins |
| `POST /admin/update/{id}` | Update admin info |
| `POST /admin/updateStatus/{id}` | Enable/disable account |
| `POST /admin/role/update` | Assign roles to admin |

### 6.2 Roles

**Path:** User → Roles

RBAC roles group permissions. Each role can be assigned:

| Endpoint | Description |
|---|---|
| `POST /role/create` | Add role |
| `POST /role/allocMenu` | Assign visible menu items |
| `POST /role/allocResource` | Assign API resource permissions |

### 6.3 Menus

**Path:** User → Menus

Hierarchical navigation structure (`GET /menu/treeList` returns the full tree). Each menu has:
- `name`, `parentId`, `level` (0=top, 1=child, 2=grandchild)
- `icon`, `hidden` (visibility toggle)
- `sort` (ordering)

### 6.4 Resources

**Path:** User → Resources

API endpoints that require authorization. Each resource has:
- `name`, `url` (path pattern), `description`
- `categoryId` (grouped by resource category)

The `DynamicSecurityService` loads all resources and their required roles at runtime. When a request hits the admin API, `DynamicAuthorizationManager` checks if the current user's role has access to the matching resource URL.

### 6.5 Member Levels

**Path:** User → Member Levels

Pre-defined membership tiers (`GET /memberLevel/list?defaultStatus=`). Each level has:
- `name`, `growthPoint` (threshold), `defaultStatus` (default for new members)
- `freeFreightPoint`, `commentGrowthPoint`, `privilegeFreeFreight`
- `privilegeSignIn`, `privilegeComment`, `privilegePromotion`
- `privilegeMemberPrice`, `privilegeBirthday`, `note`

These enable member-tier pricing on products and special privileges.

---

## 7. File Storage

### 7.1 MinIO (Self-Hosted)

**Console:** http://localhost:9005 (minioadmin / minioadmin)

| Admin API | Description |
|---|---|
| `POST /minio/upload` | Upload file (returns URL) |
| `POST /minio/delete?objectName=` | Delete file |

The default bucket name is `mall`. The MinIO endpoint is configured at `minio.endpoint` in `application-dev.yml` (default: `http://localhost:9000`).

### 7.2 Aliyun OSS (Cloud)

| Admin API | Description |
|---|---|
| `GET /aliyun/oss/policy` | Get upload signature for direct browser-to-OSS upload |
| `POST /aliyun/oss/callback` | OSS callback after upload |

Configure with your Aliyun credentials in `application-dev.yml`: `accessKey`, `secretKey`, `bucket`, `endpoint`.

---

## 8. Storefront API (mall-portal)

These are the REST APIs consumed by the mobile app (mall-app-web) or any custom frontend.

### 8.1 Member Authentication

**Registration flow:**
```
1. GET  /sso/getAuthCode?telephone=138xxxx     → receive SMS code
2. POST /sso/register?username&password&telephone&authCode  → create member
3. POST /sso/login?username&password            → get JWT token
4. Header: Authorization: Bearer <token>        → use for all subsequent requests
```

**Password reset:**
```
POST /sso/updatePassword?telephone&password&authCode
```

### 8.2 Home Page

Gets the full homepage data bundle:
```
GET /home/content
```
Returns: `advertiseList` (banners), `brandList`, `homeFlashPromotion` (current flash), `newProductList`, `hotProductList`, `subjectList`.

Individual sections can also be fetched separately:
```
GET /home/recommendProductList?pageSize=4&pageNum=1
GET /home/hotProductList?pageSize=6&pageNum=1
GET /home/newProductList?pageSize=6&pageNum=1
GET /home/subjectList?cateId=&pageSize=4&pageNum=1
GET /home/productCateList/{parentId}     -- 0 for top-level
```

### 8.3 Product Browsing

**Category tree:**
```
GET /product/categoryTreeList
```

**Product search and filter:**
```
GET /product/search?keyword=手机&brandId=&productCategoryId=&pageNum=1&pageSize=20&sort=0
```
Sort options: 0=relevance, 1=newest, 2=best selling, 3=price low→high, 4=price high→low

**Brand products:**
```
GET /brand/productList?brandId=1&pageNum=1&pageSize=20
```

**Product detail:**
```
GET /product/detail/{id}
```
Returns: product info, brand, attributes, SKU prices/stock, promotions, full-reduction rules, ladder pricing, member pricing.

### 8.4 Shopping Cart

All cart endpoints require authentication (JWT header).

```
POST   /cart/add              → {productId, productSkuId, quantity, ...}
GET    /cart/list              → current member's cart items
GET    /cart/list/promotion    → cart items with calculated discounts
GET    /cart/update/quantity?id=1&quantity=3
POST   /cart/update/attr       → change SKU of an existing cart item
POST   /cart/delete?ids=1,2,3  → batch remove
POST   /cart/clear             → empty cart
```

Cart data is stored in Redis (session-based).

### 8.5 Order Placement

**Step 1 — Preview order:**
```
POST /order/generateConfirmOrder
Body: [cartId1, cartId2, ...]
```
Returns: `ConfirmOrderResult` with cart items (with promotions), shipping addresses, available coupons, integration rules, and the price breakdown (subtotal, freight, discount, total).

**Step 2 — Place order:**
```
POST /order/generateOrder
Body: {
  "memberReceiveAddressId": 1,
  "couponId": null,
  "useIntegration": 0,
  "payType": 0,
  "cartIds": [1, 2, 3]
}
```
Returns: order ID and order SN. The order is created with status=0 (pending payment). A delayed RabbitMQ message is sent for 30-minute timeout auto-cancel.

**Step 3 — Pay (Alipay):**
```
GET /alipay/pay?outTradeNo=<orderSn>&subject=订单名称&totalAmount=199.00
```
Returns an HTML form that auto-submits to Alipay's sandbox gateway.

**Step 4 — Payment callback:**
```
POST /alipay/notify   — Alipay async notify (server-to-server)
```
The service verifies the Alipay signature and calls `order/paySuccess` to update the order status to 1 (paid/pending shipment).

### 8.6 Order Management (Storefront)

```
GET    /order/list?status=-1&pageNum=1&pageSize=5
       status: -1=all, 0=unpaid, 1=unshipped, 2=shipped, 3=completed, 4=closed
GET    /order/detail/{orderId}
POST   /order/cancelUserOrder?orderId=1     → cancel unpaid order
POST   /order/confirmReceiveOrder?orderId=1 → confirm receipt
POST   /order/deleteOrder?orderId=1         → soft delete
```

### 8.7 Returns

```
POST /returnApply/create
Body: {orderId, productId, returnName, returnPhone, returnAmount, reason, description, proofImages, ...}
```

### 8.8 Member Self-Service

**Shipping addresses:**
```
GET    /member/address/list
POST   /member/address/add     → {name, phone, province, city, region, detailAddress, defaultStatus}
POST   /member/address/update/{id}
POST   /member/address/delete/{id}
```

**Coupons:**
```
GET  /member/coupon/list?useStatus=     → available coupons
GET  /member/coupon/listHistory?useStatus= → all history
POST /member/coupon/add/{couponId}       → claim a coupon
GET  /member/coupon/list/cart/{type}     → coupons applicable to cart items (0=unavailable, 1=available)
GET  /member/coupon/listByProduct/{productId}
```

**Brand follows** (MongoDB):
```
POST   /member/attention/add?brandId=1
POST   /member/attention/delete?brandId=1
GET    /member/attention/list?pageNum=1&pageSize=5
POST   /member/attention/clear
```

**Favorites** (MongoDB):
```
POST   /member/productCollection/add?productId=1
POST   /member/productCollection/delete?productId=1
GET    /member/productCollection/list?pageNum=1&pageSize=5
POST   /member/productCollection/clear
```

**Browsing history** (MongoDB):
```
POST   /member/readHistory/create?productId=1
POST   /member/readHistory/delete?ids=1,2
GET    /member/readHistory/list?pageNum=1&pageSize=10
POST   /member/readHistory/clear
```

---

## 9. Search Service (mall-search)

### 9.1 Data Import

Before search works, products must be indexed into Elasticsearch:
```
POST /esProduct/importAll
```
This reads all published products from MySQL via `EsProductDao` and bulk-indexes them into the `pms` index.

### 9.2 Search API

```
GET /esProduct/search?keyword=手机&brandId=&productCategoryId=&pageNum=0&pageSize=5&sort=0
```

| Parameter | Description |
|---|---|
| `keyword` | Search text |
| `brandId` | Filter by brand |
| `productCategoryId` | Filter by category |
| `sort` | 0=relevance, 1=newest, 2=sales, 3=price asc, 4=price desc |
| `pageNum` | 0-based page number |
| `pageSize` | Results per page |

**Search scoring:** Product name (weight 10) > subtitle (weight 5) > keywords (weight 2). Results below score 2.0 are excluded.

### 9.3 Faceted Search Info

```
GET /esProduct/search/relate?keyword=手机
```
Returns the brand names, category names, and attribute values available for the search term — used to build filter chips/facets on the search results page.

### 9.4 Product Recommendations

```
GET /esProduct/recommend/{productId}?pageNum=1&pageSize=5
```
Uses function scoring: name (weight 8) > brandId (weight 5) > productCategoryId (weight 3) > subtitle/keywords (weight 2 each). The reference product is excluded from results.

### 9.5 CRUD

```
POST /esProduct/create/{id}     → index single product
GET  /esProduct/delete/{id}     → remove from index
POST /esProduct/delete/batch    → bulk remove
```

---

## 10. Technology Architecture Notes

### Authentication Flow (JWT)

```
Login → Validate credentials → Generate JWT (jjwt library)
→ Token format: {sub, created, exp} signed with HMAC-SHA256
→ Token cached in Redis for fast validation
→ Every request: JwtAuthenticationTokenFilter extracts token
→ DynamicAuthorizationManager checks resource-role permissions
```

### Order Timeout (RabbitMQ)

```
Order placed → Send message to TTL queue (30 min delay)
→ Message expires → Dead-letter exchange forwards to cancel queue
→ CancelOrderReceiver picks up → Checks order status
→ If still unpaid: cancels order, restores stock, releases coupon
```

### Cart Storage (Redis)

```
Cart items stored as Redis Hash: mall:cart:{memberId}
Each hash field is the cart item ID, value is JSON-serialized OmsCartItem
```

### User Activity (MongoDB)

```
Database: mall-port
Collections: member_read_history, member_product_collection, member_brand_attention
Each document stores memberId + product/brand info + timestamp
```

### Product Search (Elasticsearch)

```
Index: pms
Document: EsProduct (denormalized with brand, category, attributes)
IK Analyzer: ik_max_word for Chinese text tokenization
```
