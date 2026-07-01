# mall — E-Commerce System Analysis Report

## 1. Project Overview

| Attribute | Value |
|---|---|
| **Name** | mall (E-Commerce System) |
| **Author** | macrozheng |
| **License** | Apache License 2.0 |
| **Group** | com.macro.mall |
| **Version** | 1.0-SNAPSHOT |
| **Branch** | `master`: Spring Boot 3.5 / JDK 17; `dev-v2`: Spring Boot 2.7 / JDK 8 |
| **Source** | https://github.com/macrozheng/mall |
| **Docs** | https://www.macrozheng.com |

**One-liner**: A full-featured e-commerce system based on Spring Boot 3.5 + MyBatis, including a back-office management system and a front-end storefront, deployed via Docker.

### Companion Frontend Projects

| Project | Description | URL |
|---|---|---|
| mall-admin-web | Admin UI (Vue + Element) | https://github.com/macrozheng/mall-admin-web |
| mall-app-web | Mobile storefront (uni-app) | https://github.com/macrozheng/mall-app-web |
| mall-swarm | Microservice version (Spring Cloud Alibaba) | https://github.com/macrozheng/mall-swarm |

---

## 2. Module Architecture

```
mall (Parent POM)
├── mall-common    — Shared utilities and common code
├── mall-mbg       — MyBatis Generator generated database code
├── mall-security  — Spring Security + JWT shared module
├── mall-demo      — Framework test code
├── mall-admin     — Back-office admin API (31 Controllers)
├── mall-search    — Elasticsearch-based product search
└── mall-portal    — Front-end storefront API (13 Controllers)
```

### Dependency Graph

```
        mall-common (foundation)
            │
         mall-mbg (data layer, depends on mall-common)
            │
      mall-security (auth/authz, depends on mall-common)
            │
    ┌───────┼───────────┐
    │       │           │
 mall-demo mall-admin mall-search mall-portal
(test)    (admin API) (search svc) (storefront)
          │           │
      aliyun-oss    elasticsearch
      minio         spring-data-es
```

---

## 3. Technology Stack

### Backend

| Technology | Purpose | Version |
|---|---|---|
| **Spring Boot** | Web framework | 3.5.14 |
| **Spring Security** | Auth & authorization | bundled |
| **MyBatis** | ORM framework | 3.5.19 |
| **MyBatis Generator** | Data layer code generation | 1.4.2 |
| **PageHelper** | MyBatis pagination | 2.1.1 |
| **MySQL** | Relational database | 5.7+ / 8.0 |
| **Druid** | Connection pool | 1.2.24 |
| **Elasticsearch** | Search engine | 7.17.3 |
| **MongoDB** | NoSQL document DB | 5.0 |
| **Redis** | Cache, session, cart | 7.0 |
| **RabbitMQ** | Message queue (order timeout) | 3.10.5 |
| **SpringDoc OpenAPI** | API documentation | 2.8.17 |
| **JWT** | Stateless auth | jjwt |
| **Aliyun OSS** | Cloud object storage | 3.18.5 |
| **MinIO** | Self-hosted object storage | 8.6.0 |
| **Alipay SDK** | Payment integration | 4.40.630.ALL |
| **Logstash** | Log collection | 8.0 |
| **Hutool** | Java utility library | 5.8.40 |
| **Lombok** | Language enhancement | - |
| **Bouncy Castle** | Cryptography library | 1.84 |
| **OkHttp** | HTTP client | 5.1.0 |
| **Docker** | Container deployment | fabric8 plugin |

### Frontend

| Technology | Purpose |
|---|---|
| Vue | Frontend framework |
| Vue Router | Routing |
| Vuex | State management |
| Element | UI component library |
| Axios | HTTP requests |
| uni-app | Cross-platform mobile framework |

---

## 4. Module Deep Dive

### 4.1 mall-common (Shared Utilities)

**Package structure**:
```
com.macro.mall.common
├── api/           CommonResult<T>, CommonPage<T>, ResultCode, IErrorCode
├── config/        BaseRedisConfig (Redis serialization config)
├── domain/        SwaggerProperties, WebLog
├── exception/     ApiException, Asserts, GlobalExceptionHandler
├── log/           WebLogAspect (AOP request logging)
├── service/       RedisService + RedisServiceImpl
└── util/          RequestUtil
```

**Responsibilities**:
- Unified API response format `CommonResult<T>`: `{code, message, data}`
- Unified pagination wrapper `CommonPage<T>` (based on PageHelper)
- Global exception handling (`@RestControllerAdvice`)
- AOP request logging
- Redis operations wrapper (`set/get/remove` etc.)
- Shared dependency base for all modules

---

### 4.2 mall-mbg (Data Layer Generator)

**Package structure**:
```
com.macro.mall
├── CommentGenerator.java    Custom MBG comment generator
├── Generator.java           MBG entry point
├── mapper/                  74 Mapper interfaces
└── model/                   Entity + Example classes
resources:
├── generator.properties     DB connection config
├── generatorConfig.xml      MBG config (all tables)
└── com/macro/mall/mapper/   74 XML mapping files
```

**Responsibilities**:
- Auto-generates all data layer code via MyBatis Generator
- Full Mapper + Model + Example generation for 76 tables
- Custom CommentGenerator adds Chinese annotations

---

### 4.3 mall-security (Auth Module)

**Package structure**:
```
com.macro.mall.security
├── annotation/      @CacheException
├── aspect/          RedisCacheAspect
├── component/       JwtAuthenticationTokenFilter, DynamicSecurity*,
│                    RestAuthenticationEntryPoint, RestfulAccessDeniedHandler
├── config/          CommonSecurityConfig, SecurityConfig, IgnoreUrlsConfig, RedisConfig
└── util/            JwtTokenUtil, SpringUtil
```

**Responsibilities**:
- JWT token generation, parsing, validation
- Spring Security filter chain configuration
- **Dynamic URL authorization**: Loads resource-role mappings from DB at runtime via `DynamicSecurityService`, dynamically matches request paths
- Whitelist URL configuration (`IgnoreUrlsConfig`)
- Redis caching for auth data
- Unified entry point (401) and access-denied handling (403)

**Auth flow**:
```
Request → JwtAuthenticationTokenFilter → Parse JWT → Load user permissions
→ DynamicAuthorizationManager match resource → permit/deny
```

---

### 4.4 mall-admin (Back-office Admin API)

**Package structure**:
```
com.macro.mall
├── MallAdminApplication.java
├── controller/     31 REST Controllers
│   ├── Pms*        — Product mgmt (product, brand, category, attribute, SKU)
│   ├── Oms*        — Order mgmt (order, returns, return reasons, settings)
│   ├── Ums*        — User mgmt (admin, role, menu, resource)
│   ├── Sms*        — Promotion mgmt (coupon, flash sales, home recommendations)
│   └── Cms*        — Content mgmt (subjects, preference areas)
├── service/ + impl/
├── dao/            Custom complex query DAOs
├── dto/            24 DTOs
├── bo/
├── config/         App config
└── validator/      FlagValidator custom validation
```

**Full Controller List**:

| Domain | Controller | Description |
|---|---|---|
| **Product** | PmsProductController | Product CRUD, publish/unpublish |
| | PmsBrandController | Brand management |
| | PmsProductCategoryController | Product categories |
| | PmsProductAttributeController | Product attributes |
| | PmsProductAttributeCategoryController | Attribute categories |
| | PmsSkuStockController | SKU stock management |
| **Order** | OmsOrderController | Order query, delivery, notes |
| | OmsOrderReturnApplyController | Return request handling |
| | OmsOrderReturnReasonController | Return reason management |
| | OmsOrderSettingController | Order settings |
| | OmsCompanyAddressController | Company address management |
| **Promotion** | SmsCouponController | Coupon CRUD |
| | SmsCouponHistoryController | Coupon claim history |
| | SmsFlashPromotionController | Flash sale activities |
| | SmsFlashPromotionSessionController | Flash sale sessions |
| | SmsFlashPromotionProductRelationController | Flash product relations |
| | SmsHomeAdvertiseController | Home page ads |
| | SmsHomeBrandController | Home brand recommendations |
| | SmsHomeNewProductController | Home new product recommendations |
| | SmsHomeRecommendProductController | Home popular recommendations |
| | SmsHomeRecommendSubjectController | Home topic recommendations |
| **Content** | CmsSubjectController | Subject management |
| | CmsPrefrenceAreaController | Preference area management |
| **Auth** | UmsAdminController | Admin login/register/permissions |
| | UmsRoleController | Role management |
| | UmsMenuController | Menu management |
| | UmsResourceController | Resource management |
| | UmsResourceCategoryController | Resource category management |
| | UmsMemberLevelController | Member level management |
| **Storage** | MinioController | MinIO file upload |
| | OssController | Aliyun OSS file upload |

---

### 4.5 mall-portal (Storefront API)

**Package structure**:
```
com.macro.mall.portal
├── MallPortalApplication.java
├── controller/     13 REST Controllers
│   ├── HomeController              Home page aggregation
│   ├── PmsPortalProductController  Product browsing/search/detail
│   ├── PmsPortalBrandController    Brand page
│   ├── OmsCartItemController       Shopping cart
│   ├── OmsPortalOrderController    Place/cancel/payment confirm
│   ├── OmsPortalOrderReturnApplyController  Return requests
│   ├── UmsMemberController         Register/login/info
│   ├── UmsMemberReceiveAddressController   Shipping addresses
│   ├── UmsMemberCouponController   Coupons
│   ├── MemberReadHistoryController Browse history (MongoDB)
│   ├── MemberProductCollectionController   Product favorites (MongoDB)
│   ├── MemberAttentionController   Brand follows (MongoDB)
│   └── AlipayController            Alipay payment
├── service/ + impl/
├── dao/            5 custom DAOs
├── domain/         15 domain objects
├── repository/     3 MongoDB Repositories
├── component/      CancelOrderReceiver/Sender, OrderTimeOutCancelTask
├── config/         10 config classes
└── util/           DateUtil
```

**Core Features**:

| Feature | Implementation |
|---|---|
| **Home page** | Aggregates ads, new products, popular items, brand recs, subjects, flash sales |
| **Product search** | Elasticsearch full-text search + filter/sort |
| **Shopping cart** | Redis-backed storage |
| **Order placement** | MySQL transaction + RabbitMQ delayed messages for timeout cancellation |
| **Payment** | Alipay SDK (sandbox) |
| **Member** | JWT login + Redis cache |
| **Browse/favorites** | MongoDB document storage |

**Order timeout cancellation**:
```
Place order → Send delayed message (TTL 30min) to RabbitMQ
→ Message expires → Dead-letter exchange forwards → Cancel order queue
→ CancelOrderReceiver consumes → Check & cancel unpaid order
```

---

### 4.6 mall-search (Product Search Service)

**Package structure**:
```
com.macro.mall.search
├── MallSearchApplication.java
├── controller/     EsProductController
├── service/        EsProductService + impl
├── dao/            EsProductDao
├── domain/         EsProduct, EsProductAttributeValue, EsProductRelatedInfo
├── repository/     EsProductRepository (Spring Data ES)
└── config/         SpringDocConfig, MyBatisConfig
```

**Core APIs**:

| Endpoint | Description |
|---|---|
| `POST /esProduct/importAll` | Full import products from MySQL to ES |
| `GET /esProduct/search` | Keyword search + brand/category/attribute aggregation + sort |
| `GET /esProduct/recommend/{id}` | Product recommendation |
| `GET /esProduct/{id}` | Query single product |

**Search features**:
- IK Chinese tokenizer (`ik_max_word`)
- Faceted filtering by brand, category, attributes
- Sorting by relevance, newest, sales, price

---

### 4.7 mall-demo (Test Module)

**Purpose**: Framework test/validation module integrating Thymeleaf template engine, RestTemplate call examples, and Spring Security configuration validation.

---

## 5. Database Design

### 5.1 MySQL — 76 Tables

| Prefix | Domain | Count | Description |
|---|---|---|---|
| `cms_` | Content Management | 12 | help, member_report, subject, subject_category, topic, topic_category, prefrence_area, prefrence_area_product_relation, subject_product_relation, topic_comment, topic_category, help_category |
| `oms_` | Order Management | 7 | order, order_item, order_operate_history, order_return_apply, order_return_reason, order_setting, cart_item, company_address |
| `pms_` | Product Management | 18 | product, brand, product_category, product_attribute, product_attribute_category, product_attribute_value, product_full_reduction, product_ladder, product_vertify_record, sku_stock, product_operate_log, comment, comment_replay, album, album_pic, member_price, product_category_attribute_relation, feight_template |
| `sms_` | Promotion | 13 | coupon, coupon_history, coupon_product_category_relation, coupon_product_relation, flash_promotion, flash_promotion_session, flash_promotion_product_relation, home_advertise, home_brand, home_new_product, home_recommend_product, home_recommend_subject, home_recommend_subject_product_relation |
| `ums_` | Users | 24 | admin, admin_login_log, admin_permission_relation, admin_role_relation, role, role_menu_relation, role_permission_relation, role_resource_relation, permission, resource, resource_category, menu, member, member_level, member_login_log, member_rule_setting, member_tag, member_product_category_relation, member_member_tag_relation, member_statistics_info, member_receive_address, member_brand_attention, member_collection, member_collection_spu, member_collection_subject, growth_change_history, integration_change_history, integration_consume_setting |

### 5.2 Redis — Cache & Temporary Data

- Session / Token cache (admin + member)
- Shopping cart (Hash structure)
- Verification codes
- Order ID generation

### 5.3 MongoDB — User Activity Data (DB: `mall-port`)

- Member browse history
- Product favorites
- Brand follows

### 5.4 Elasticsearch — Product Search Index (Index: `pms`)

- Product documents (with nested brand, category, attributes, SKU)
- IK Chinese tokenizer

---

## 6. Key Architecture Decisions

### 6.1 Unified API Response

```json
{
  "code": 200,
  "message": "Success",
  "data": { ... }
}
```

All controllers return `CommonResult<T>`; exceptions are handled uniformly by `GlobalExceptionHandler`.

### 6.2 RBAC Permission Model

```
Admin (ums_admin)
  └── many-to-many → Role (ums_role)
        └── many-to-many → Resource (ums_resource)
```

- Resource defines URL + HTTP method
- `DynamicSecurityMetadataSource` loads all resources and their required roles at runtime
- `DynamicAuthorizationManager` matches request paths in real-time

### 6.3 JWT Auth Flow

```
Login → Verify credentials → Generate JWT (userID + roles + resources)
→ Subsequent requests carry Header: Authorization: Bearer <token>
→ JwtAuthenticationTokenFilter parses token → Sets SecurityContext
```

### 6.4 Order Timeout Cancel (RabbitMQ Delayed Messages)

```
Place order → Send message to order.direct (routing key: order.create)
 → order.create.queue bound to order.create.xdelayed exchange (TTL: 30min)
 → Message expires → Dead-letter exchange order.dead.exchange
 → order.cancel.queue bound → CancelOrderReceiver
 → Check order status, cancel if unpaid & restore stock
```

### 6.5 Dual Storage Strategy

| Storage | Use Case | Config |
|---|---|---|
| Aliyun OSS | Cloud storage | oss.endpoint / accessKey / bucket |
| MinIO | Self-hosted private storage | minio.endpoint / accessKey / bucket |

### 6.6 Docker Containerization

```
maven package → fabric8 docker-maven-plugin → build image
→ Docker Compose deploy (docker-compose-app.yml + docker-compose-env.yml)
```

- Base image: `openjdk:17`
- Image name: `mall/${project.name}:${project.version}`
- Startup: `-Dspring.profiles.active=prod`

---

## 7. API Routes Overview

### Admin API (mall-admin)

| Path Prefix | Module | Examples |
|---|---|---|
| `/product/**` | Product | `/product/list`, `/product/create`, `/product/update/{id}` |
| `/brand/**` | Brand | `/brand/list`, `/brand/create` |
| `/productCategory/**` | Category | `/productCategory/list/{parentId}` |
| `/order/**` | Order | `/order/list`, `/order/delivery` |
| `/returnApply/**` | Returns | `/returnApply/list`, `/returnApply/update/{id}` |
| `/coupon/**` | Coupon | `/coupon/list`, `/coupon/create` |
| `/flashPromotion/**` | Flash sales | `/flashPromotion/list` |
| `/admin/**` | Admin | `/admin/login`, `/admin/info` |
| `/role/**` | Role | `/role/list`, `/role/allocMenu` |
| `/resource/**` | Resource | `/resource/list` |
| `/minio/**` | File upload | `/minio/upload` |
| `/oss/**` | Aliyun OSS | `/oss/policy` |

### Storefront API (mall-portal)

| Path | Description |
|---|---|
| `/home/**` | Home page aggregation |
| `/product/**` | Product browsing & search |
| `/brand/**` | Brand listings |
| `/cart/**` | Cart operations |
| `/order/**` | Place order, pay, cancel |
| `/returnApply/**` | Return requests |
| `/member/**` | Register, login, profile |
| `/address/**` | Address management |
| `/coupon/**` | Coupons |
| `/member/readHistory/**` | Browse history |
| `/member/productCollection/**` | Favorites |
| `/member/attention/**` | Brand follows |
| `/alipay/**` | Alipay payment |

### Search API (mall-search)

| Path | Description |
|---|---|
| `/esProduct/search` | Product search + faceted filter |
| `/esProduct/recommend/{id}` | Product recommendation |
| `/esProduct/{id}` | Product detail |
| `/esProduct/importAll` | Full import to ES |

---

## 8. Project File Structure

```
mall/
├── pom.xml
├── README.md
├── LICENSE
├── .gitignore
│
├── mall-common/          # Shared utilities + common code
│   └── src/main/java/com/macro/mall/common/
├── mall-mbg/             # MyBatis Generator output
│   └── src/main/java/com/macro/mall/{mapper,model}/
├── mall-security/        # Spring Security + JWT
│   └── src/main/java/com/macro/mall/security/
├── mall-admin/           # Admin API (31 Controllers)
│   └── src/main/java/com/macro/mall/{controller,service,dao,dto,bo,config,validator}/
├── mall-portal/          # Storefront API (13 Controllers)
│   └── src/main/java/com/macro/mall/portal/{controller,service,dao,domain,repository,component,config,util}/
├── mall-search/          # Elasticsearch search service
│   └── src/main/java/com/macro/mall/search/{controller,service,dao,domain,repository,config}/
├── mall-demo/            # Test module
│   └── src/main/java/com/macro/mall/demo/{controller,service,dto,bo,config,validator}/
│
└── document/
    ├── sql/mall.sql              # DB schema (76 tables)
    ├── docker/                   # Docker Compose deployment files
    │   ├── docker-compose-app.yml
    │   ├── docker-compose-env.yml
    │   └── nginx.conf
    ├── elk/logstash.conf         # Logstash config
    ├── sh/                       # Deployment scripts
    ├── pdm/                      # PowerDesigner data models
    ├── resource/                 # Architecture diagrams, screenshots
    ├── postman/                  # API collections
    ├── axure/                    # Axure prototypes
    ├── mind/                     # Mind maps
    ├── reference/                # Reference docs
    └── pos/                      # POS related
```

---

## 9. Design Highlights

1. **Multi-module decoupling**: Common → MBG → Security → Admin/Portal/Search — clear layering
2. **Unified response contract**: `CommonResult<T>` across all endpoints
3. **Dynamic authorization**: Resource-role mappings loaded from DB at runtime, no hardcoding
4. **Delayed messages**: RabbitMQ TTL + dead-letter exchange for order timeout cancel, no polling
5. **Dual storage**: Supports both Aliyun OSS and self-hosted MinIO
6. **Document DB**: MongoDB for user activity data, avoids MySQL bloat
7. **Full-text search**: Elasticsearch + IK analyzer, multi-dimensional faceted search
8. **Containerization**: Maven plugin builds Docker images directly, Compose orchestration
9. **API docs**: SpringDoc OpenAPI auto-generation
10. **Code generation**: MyBatis Generator automates data layer, reducing boilerplate
