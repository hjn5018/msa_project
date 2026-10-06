# 📖 [Phase 1] 구현 가이드: 멀티 모듈 & 상품 서비스 (Product Service)

> 본 가이드는 사용자가 코드를 직접 작성하면서 Spring Boot 3.x 기반 마이크로서비스의 기초를 체득할 수 있도록 핵심 개념과 파일별 전체 코드, 테스트 방법을 상세히 제공합니다.

---

## 📌 Phase 1 전체 디렉토리 구조 미리보기

이번 단계에서 직접 생성하고 작성할 디렉토리와 파일 구조입니다.

```text
msa_project/
├── docker-compose.yml                  # MySQL 컨테이너 정의
├── infra/
│   └── mysql/
│       └── init.sql                    # 서비스별 DB 스키마 자동 생성 스크립트
├── settings.gradle                     # 루트 Gradle 멀티 모듈 정의
├── build.gradle                        # 루트 Gradle 공통 빌드 스크립트
├── common/                             # 공통 라이브러리 모듈
│   ├── build.gradle
│   └── src/main/java/com/flashcommerce/common/
│       ├── dto/
│       │   └── ApiResponse.java        # 공통 JSON 응답 래퍼
│       └── exception/
│           ├── BusinessException.java   # 비즈니스 공통 예외
│           ├── ErrorCode.java           # 에러 코드 Enum
│           ├── ErrorResponse.java       # 에러 응답 DTO
│           └── GlobalExceptionHandler.java # 전역 @RestControllerAdvice
└── product-service/                    # 상품 마이크로서비스
    ├── build.gradle
    └── src/
        └── main/
            ├── resources/
            │   └── application.yml     # DB 연결 및 포트(8082) 설정
            └── java/com/flashcommerce/product/
                ├── ProductServiceApplication.java # 메인 실행 클래스
                ├── domain/
                │   └── Product.java    # JPA 엔티티 (DB 테이블 매핑)
                ├── repository/
                │   └── ProductRepository.java # Spring Data JPA 레포지토리
                ├── dto/
                │   ├── ProductCreateRequest.java # 상품 등록 DTO (Java Record)
                │   └── ProductResponse.java      # 상품 조회 응답 DTO
                ├── service/
                │   └── ProductService.java       # 비즈니스 로직 계층
                └── controller/
                    └── ProductController.java    # REST API 웹 계층
```

---

## Task 1.1: 루트 Gradle 멀티 모듈 구성

### 💡 핵심 개념 (Python vs Java Gradle)
* **Python**: Poetry나 UV의 workspaces, 혹은 가상환경 아래 여러 패키지를 모노레포로 관리합니다.
* **Gradle**: 루트의 `settings.gradle`에 `include 'common', 'product-service'`와 같이 서브 프로젝트를 선언하고, 루트 `build.gradle`의 `subprojects { ... }` 블록에서 모든 자식 모듈에 자바 버전(Java 21), 의존성 관리 플러그인, Lombok 등을 상속시킵니다.

### 📝 1. `settings.gradle` (프로젝트 루트)
```groovy
rootProject.name = 'flash-commerce'

// 이번 단계에서 활성화할 서브 모듈
include 'common'
include 'product-service'

// Phase 2~5에서 순차적으로 주석을 해제할 모듈들
// include 'order-service'
// include 'payment-service'
// include 'api-gateway'
```

### 📝 2. `build.gradle` (프로젝트 루트)
```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.3.4' apply false
    id 'io.spring.dependency-management' version '1.1.6' apply false
}

allprojects {
    group = 'com.flashcommerce'
    version = '0.0.1-SNAPSHOT'

    repositories {
        mavenCentral()
    }
}

subprojects {
    apply plugin: 'java'
    apply plugin: 'io.spring.dependency-management'

    java {
        toolchain {
            languageVersion = JavaLanguageVersion.of(21)
        }
    }

    configurations {
        compileOnly {
            extendsFrom annotationProcessor
        }
    }

    dependencies {
        // Lombok: Getter, Builder, 생성자 등을 어노테이션으로 자동 생성
        compileOnly 'org.projectlombok:lombok'
        annotationProcessor 'org.projectlombok:lombok'

        // 단위 테스트 라이브러리 (JUnit 5)
        testImplementation 'org.springframework.boot:spring-boot-starter-test'
        testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
    }

    test {
        useJUnitPlatform()
    }
}
```

---

## Task 1.2: 로컬 DB 인프라 구성 (Database-per-Service)

### 💡 핵심 개념
MSA의 제1원칙은 **"서비스마다 독립된 데이터베이스를 갖는다(Database per Service)"** 입니다.  
서비스끼리 DB를 직접 JOIN하거나 테이블을 공유하지 않습니다. 로컬 개발의 리소스 절약을 위해 MySQL 인스턴스는 1대만 띄우되, 내부적으로 `product_db`, `order_db`, `payment_db`로 물리적 데이터베이스를 엄격히 분리합니다.

### 📝 1. `docker-compose.yml` (프로젝트 루트)
```yaml
services:
  mysql:
    image: mysql:8.0
    container_name: flash-commerce-mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
    ports:
      - "3306:3306"
    volumes:
      - ./infra/mysql/init.sql:/docker-entrypoint-initdb.d/init.sql
      - mysql-data:/var/lib/mysql

volumes:
  mysql-data:
```

### 📝 2. `infra/mysql/init.sql`
MySQL 컨테이너가 최초 생성될 때 실행되어 3개 마이크로서비스용 DB를 자동 생성합니다.
```sql
CREATE DATABASE IF NOT EXISTS product_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE IF NOT EXISTS order_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE IF NOT EXISTS payment_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

---

## Task 1.3: `common` 모듈 생성

### 💡 핵심 개념
* `common` 모듈은 자체적으로 실행되는 웹 서버가 아니라, 다른 마이크로서비스들이 의존성(`implementation project(':common')`)으로 끌어다 쓰는 **순수 라이브러리(Jar)** 모듈입니다.
* 모든 서비스가 통일된 JSON 포맷(`ApiResponse<T>`)과 표준 에러 구조를 반환하도록 도와줍니다.

### 📝 1. `common/build.gradle`
```groovy
plugins {
    id 'java-library'
}

dependencies {
    // Spring Web 어노테이션(@RestControllerAdvice 등) 및 Jackson(JSON 변환)
    api 'org.springframework.boot:spring-boot-starter-web:3.3.4'
    // DTO Validation 어노테이션(@NotBlank, @Min 등)
    api 'org.springframework.boot:spring-boot-starter-validation:3.3.4'
}

// 라이브러리 모듈이므로 실행 가능한 bootJar가 아닌 일반 jar로 빌드
bootJar { enabled = false }
jar { enabled = true }
```

### 📝 2. `common/src/main/java/com/flashcommerce/common/dto/ApiResponse.java`
Python FastAPI의 응답 딕셔너리처럼, 성공/실패 여부와 데이터를 감싸는 표준 응답 래퍼입니다.
```java
package com.flashcommerce.common.dto;

import lombok.Getter;

@Getter
public class ApiResponse<T> {
    private final boolean success;
    private final T data;
    private final String message;

    private ApiResponse(boolean success, T data, String message) {
        this.success = success;
        this.data = data;
        this.message = message;
    }

    public static <T> ApiResponse<T> success(T data) {
        return new ApiResponse<>(true, data, "SUCCESS");
    }

    public static <T> ApiResponse<T> success(T data, String message) {
        return new ApiResponse<>(true, data, message);
    }

    public static <T> ApiResponse<T> error(String message) {
        return new ApiResponse<>(false, null, message);
    }
}
```

### 📝 3. `common/src/main/java/com/flashcommerce/common/exception/ErrorCode.java`
```java
package com.flashcommerce.common.exception;

import lombok.Getter;
import org.springframework.http.HttpStatus;

@Getter
public enum ErrorCode {
    INVALID_INPUT_VALUE(HttpStatus.BAD_REQUEST, "잘못된 입력값입니다."),
    ENTITY_NOT_FOUND(HttpStatus.NOT_FOUND, "대상을 찾을 수 없습니다."),
    INSUFFICIENT_STOCK(HttpStatus.BAD_REQUEST, "재고가 부족합니다."),
    INTERNAL_SERVER_ERROR(HttpStatus.INTERNAL_SERVER_ERROR, "서버 내부 오류가 발생했습니다.");

    private final HttpStatus status;
    private final String message;

    ErrorCode(HttpStatus status, String message) {
        this.status = status;
        this.message = message;
    }
}
```

### 📝 4. `common/src/main/java/com/flashcommerce/common/exception/BusinessException.java`
```java
package com.flashcommerce.common.exception;

import lombok.Getter;

@Getter
public class BusinessException extends RuntimeException {
    private final ErrorCode errorCode;

    public BusinessException(ErrorCode errorCode) {
        super(errorCode.getMessage());
        this.errorCode = errorCode;
    }

    public BusinessException(ErrorCode errorCode, String customMessage) {
        super(customMessage);
        this.errorCode = errorCode;
    }
}
```

### 📝 5. `common/src/main/java/com/flashcommerce/common/exception/ErrorResponse.java`
```java
package com.flashcommerce.common.exception;

import lombok.Getter;
import java.time.LocalDateTime;

@Getter
public class ErrorResponse {
    private final boolean success = false;
    private final String errorCode;
    private final String message;
    private final LocalDateTime timestamp;

    public ErrorResponse(ErrorCode errorCode, String message) {
        this.errorCode = errorCode.name();
        this.message = message;
        this.timestamp = LocalDateTime.now();
    }
}
```

### 📝 6. `common/src/main/java/com/flashcommerce/common/exception/GlobalExceptionHandler.java`
Python의 `@app.exception_handler`와 동일한 역할을 합니다.
```java
package com.flashcommerce.common.exception;

import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusinessException(BusinessException e) {
        log.warn("BusinessException: {}", e.getMessage());
        ErrorCode errorCode = e.getErrorCode();
        ErrorResponse response = new ErrorResponse(errorCode, e.getMessage());
        return new ResponseEntity<>(response, errorCode.getStatus());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidException(MethodArgumentNotValidException e) {
        String errorMessage = e.getBindingResult().getAllErrors().get(0).getDefaultMessage();
        log.warn("Validation failed: {}", errorMessage);
        ErrorResponse response = new ErrorResponse(ErrorCode.INVALID_INPUT_VALUE, errorMessage);
        return new ResponseEntity<>(response, ErrorCode.INVALID_INPUT_VALUE.getStatus());
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneralException(Exception e) {
        log.error("Unhandled Exception: ", e);
        ErrorResponse response = new ErrorResponse(ErrorCode.INTERNAL_SERVER_ERROR, "서버 처리 중 오류가 발생했습니다.");
        return new ResponseEntity<>(response, ErrorCode.INTERNAL_SERVER_ERROR.getStatus());
    }
}
```

---

## Task 1.4: `product-service` 생성 (상품 마이크로서비스)

### 💡 핵심 개념 (Spring 계층 구조)
1. **Controller (`@RestController`)**: HTTP 요청 매핑, DTO 유효성 검사 (`@Valid`), 응답 반환 (FastAPI Router)
2. **Service (`@Service`)**: 비즈니스 로직 및 트랜잭션 경계 (`@Transactional`) 관리
3. **Repository (`@Repository`)**: DB 접근 추상화 인터페이스 (Spring Data JPA)
4. **Entity (`@Entity`)**: DB 테이블과 1:1 매핑되는 도메인 모델 (SQLAlchemy Model)
5. **DTO (Java Record)**: 계층 간 데이터를 전달하는 불변 객체 (Pydantic BaseModel)

### 📝 1. `product-service/build.gradle`
```groovy
plugins {
    id 'org.springframework.boot'
}

dependencies {
    // 공통 모듈 참조
    implementation project(':common')

    // 웹 및 JPA, MySQL 드라이버
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    runtimeOnly 'com.mysql:mysql-connector-j'
}
```

### 📝 2. `product-service/src/main/resources/application.yml`
포트 번호 `8082`, `product_db` 접속 정보, Hibernate DDL 자동 생성을 설정합니다.
```yaml
server:
  port: 8082

spring:
  application:
    name: product-service

  datasource:
    url: jdbc:mysql://localhost:3306/product_db?useSSL=false&allowPublicKeyRetrieval=true&characterEncoding=UTF-8&serverTimezone=Asia/Seoul
    username: root
    password: rootpassword
    driver-class-name: com.mysql.cj.jdbc.Driver

  jpa:
    hibernate:
      ddl-auto: update # 개발 환경: 엔티티 변경 시 테이블 자동 갱신
    show-sql: true
    properties:
      hibernate:
        format_sql: true
```

### 📝 3. `product-service/src/main/java/com/flashcommerce/product/ProductServiceApplication.java`
```java
package com.flashcommerce.product;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.ComponentScan;

@SpringBootApplication
@ComponentScan(basePackages = {"com.flashcommerce.product", "com.flashcommerce.common"})
public class ProductServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(ProductServiceApplication.class, args);
    }
}
```
> `@ComponentScan`에 `"com.flashcommerce.common"`을 포함해야 공통 예외 핸들러(`GlobalExceptionHandler`)가 자동 등록됩니다.

### 📝 4. `product-service/src/main/java/com/flashcommerce/product/domain/Product.java`
```java
package com.flashcommerce.product.domain;

import com.flashcommerce.common.exception.BusinessException;
import com.flashcommerce.common.exception.ErrorCode;
import jakarta.persistence.*;
import lombok.AccessLevel;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "products")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    private String name;

    @Column(nullable = false)
    private BigDecimal price;

    @Column(nullable = false)
    private Integer stockQuantity;

    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;

    @Builder
    public Product(String name, BigDecimal price, Integer stockQuantity) {
        this.name = name;
        this.price = price;
        this.stockQuantity = stockQuantity;
        this.createdAt = LocalDateTime.now();
        this.updatedAt = LocalDateTime.now();
    }

    // 도메인 비즈니스 로직: 재고 차감
    public void deductStock(int quantity) {
        if (this.stockQuantity < quantity) {
            throw new BusinessException(ErrorCode.INSUFFICIENT_STOCK, 
                "재고가 부족합니다. (현재 재고: " + this.stockQuantity + ", 요청 수량: " + quantity + ")");
        }
        this.stockQuantity -= quantity;
        this.updatedAt = LocalDateTime.now();
    }

    // 재고 복구 (보상 트랜잭션용)
    public void restoreStock(int quantity) {
        this.stockQuantity += quantity;
        this.updatedAt = LocalDateTime.now();
    }
}
```

### 📝 5. `product-service/src/main/java/com/flashcommerce/product/repository/ProductRepository.java`
```java
package com.flashcommerce.product.repository;

import com.flashcommerce.product.domain.Product;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {
    // 기본 CRUD(findById, save, findAll 등)는 Spring Data JPA가 자동 제공합니다.
}
```

### 📝 6. `product-service/src/main/java/com/flashcommerce/product/dto/ProductCreateRequest.java`
Java 21의 불변 데이터 클래스인 `record`를 사용합니다.
```java
package com.flashcommerce.product.dto;

import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import java.math.BigDecimal;

public record ProductCreateRequest(
    @NotBlank(message = "상품명은 필수입니다.")
    String name,

    @NotNull(message = "가격은 필수입니다.")
    @Min(value = 100, message = "가격은 최소 100원 이상이어야 합니다.")
    BigDecimal price,

    @NotNull(message = "재고 수량은 필수입니다.")
    @Min(value = 0, message = "재고 수량은 0개 이상이어야 합니다.")
    Integer stockQuantity
) {}
```

### 📝 7. `product-service/src/main/java/com/flashcommerce/product/dto/ProductResponse.java`
```java
package com.flashcommerce.product.dto;

import com.flashcommerce.product.domain.Product;
import java.math.BigDecimal;

public record ProductResponse(
    Long id,
    String name,
    BigDecimal price,
    Integer stockQuantity
) {
    public static ProductResponse from(Product product) {
        return new ProductResponse(
            product.getId(),
            product.getName(),
            product.getPrice(),
            product.getStockQuantity()
        );
    }
}
```

### 📝 8. `product-service/src/main/java/com/flashcommerce/product/service/ProductService.java`
```java
package com.flashcommerce.product.service;

import com.flashcommerce.common.exception.BusinessException;
import com.flashcommerce.common.exception.ErrorCode;
import com.flashcommerce.product.domain.Product;
import com.flashcommerce.product.dto.ProductCreateRequest;
import com.flashcommerce.product.dto.ProductResponse;
import com.flashcommerce.product.repository.ProductRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class ProductService {

    private final ProductRepository productRepository;

    @Transactional
    public ProductResponse createProduct(ProductCreateRequest request) {
        Product product = Product.builder()
                .name(request.name())
                .price(request.price())
                .stockQuantity(request.stockQuantity())
                .build();

        Product savedProduct = productRepository.save(product);
        return ProductResponse.from(savedProduct);
    }

    public ProductResponse getProduct(Long id) {
        Product product = productRepository.findById(id)
                .orElseThrow(() -> new BusinessException(ErrorCode.ENTITY_NOT_FOUND, "상품 ID: " + id + "를 찾을 수 없습니다."));
        return ProductResponse.from(product);
    }

    public List<ProductResponse> getAllProducts() {
        return productRepository.findAll().stream()
                .map(ProductResponse::from)
                .toList();
    }

    @Transactional
    public void deductStock(Long id, int quantity) {
        Product product = productRepository.findById(id)
                .orElseThrow(() -> new BusinessException(ErrorCode.ENTITY_NOT_FOUND, "상품 ID: " + id + "를 찾을 수 없습니다."));
        product.deductStock(quantity);
    }
}
```

### 📝 9. `product-service/src/main/java/com/flashcommerce/product/controller/ProductController.java`
```java
package com.flashcommerce.product.controller;

import com.flashcommerce.common.dto.ApiResponse;
import com.flashcommerce.product.dto.ProductCreateRequest;
import com.flashcommerce.product.dto.ProductResponse;
import com.flashcommerce.product.service.ProductService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
public class ProductController {

    private final ProductService productService;

    @PostMapping
    public ResponseEntity<ApiResponse<ProductResponse>> createProduct(@Valid @RequestBody ProductCreateRequest request) {
        ProductResponse response = productService.createProduct(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(ApiResponse.success(response, "상품이 등록되었습니다."));
    }

    @GetMapping("/{id}")
    public ResponseEntity<ApiResponse<ProductResponse>> getProduct(@PathVariable Long id) {
        ProductResponse response = productService.getProduct(id);
        return ResponseEntity.ok(ApiResponse.success(response));
    }

    @GetMapping
    public ResponseEntity<ApiResponse<List<ProductResponse>>> getAllProducts() {
        List<ProductResponse> response = productService.getAllProducts();
        return ResponseEntity.ok(ApiResponse.success(response));
    }

    @PostMapping("/{id}/deduct")
    public ResponseEntity<ApiResponse<Void>> deductStock(@PathVariable Long id, @RequestParam int quantity) {
        productService.deductStock(id, quantity);
        return ResponseEntity.ok(ApiResponse.success(null, "재고가 정상적으로 차감되었습니다."));
    }
}
```

---

## Task 1.5: 실행 및 동작 검증 (Checklist)

모든 코드 작성이 끝나면 아래 명령어로 시스템을 띄우고 테스트합니다.

### 1. Docker MySQL 실행
프로젝트 루트에서 실행:
```bash
docker compose up -d
```
> `docker ps`로 `flash-commerce-mysql` 컨테이너가 `healthy/running` 상태인지 확인합니다.

### 2. Gradle 빌드 및 서비스 실행
```bash
# 전체 프로젝트 빌드 테스트
./gradlew build -x test

# product-service 실행
./gradlew :product-service:bootRun
```
*(터미널에 `Started ProductServiceApplication in ... seconds` 로그가 뜨면 성공입니다)*

### 3. API 동작 테스트 (다른 터미널 창에서 실행)

#### ① 상품 등록 (POST)
```bash
curl -X POST http://localhost:8082/api/v1/products \
  -H "Content-Type: application/json" \
  -d '{"name": "애플 맥북 프로 16인치", "price": 3500000, "stockQuantity": 10}'
```
*기대 응답 (`201 Created`):*
```json
{
  "success": true,
  "data": {
    "id": 1,
    "name": "애플 맥북 프로 16인치",
    "price": 3500000,
    "stockQuantity": 10
  },
  "message": "상품이 등록되었습니다."
}
```

#### ② 상품 단건 조회 (GET)
```bash
curl http://localhost:8082/api/v1/products/1
```

#### ③ 재고 차감 테스트 (POST)
```bash
curl -X POST "http://localhost:8082/api/v1/products/1/deduct?quantity=3"
```
*다시 상품 조회를 했을 때 재고가 10개에서 7개로 줄어들었는지 확인합니다.*

#### ④ 예외 처리 검증 (재고 부족 유도)
```bash
curl -X POST "http://localhost:8082/api/v1/products/1/deduct?quantity=100"
```
*기대 응답 (`400 Bad Request`):*
```json
{
  "success": false,
  "errorCode": "INSUFFICIENT_STOCK",
  "message": "재고가 부족합니다. (현재 재고: 7, 요청 수량: 100)",
  "timestamp": "2026-10-06T..."
}
```
