============================================
Báo cáo công nghệ sử dụng – Backend DMS.THFC
============================================

- Phạm vi: toàn bộ backend trong workspace: 7 service Java (cms, dms, outlet, salesforce, product,
  sale, integration), auth-service, thư viện common, thư viện CTKM promotionprogram, và bộ tính HTTM
  httm-py.
- Nguồn số liệu: pom.xml của từng module, BOM spring-boot-dependencies 2.7.18 và
  spring-cloud-dependencies 2021.0.3 (để ra version thực của các thư viện không ghi version),
  Dockerfile, application.yml, httm-py/Dockerfile và httm-py/k8s/httm-engine.yaml.
- Ngày lập: 19/11/2026.
- Lưu ý: version lấy theo nhánh đang checkout, nhánh khác có thể lệch:

Repo                                                                     Nhánh
-----------------------------------------------------------------------  -------------------------------------
common, auth-service, cms-service, outlet-service, salesforce-service,   develop
  integration-service, promotionprogram
dms-service                                                              integration/uat-04
product-service                                                          feature/phucvh11/product-group
sale-service                                                             feature/phucvh11/rebateprogram-scheme
httm-py                                                                  không phải git repo

1. Tổng quan
============

Nhóm                   Công nghệ chính                                        Version
---------------------  -----------------------------------------------------  --------------------------------
Ngôn ngữ               Java (OpenJDK, Eclipse Temurin)                        17
Framework              Spring Boot                                            2.7.18
                       Spring Framework                                       5.3.34
                       Spring Cloud                                           2021.0.3
                       Spring Security                                        5.7.12
ORM                    Hibernate ORM                                          5.6.15.Final
CSDL quan hệ           MariaDB (driver mariadb-java-client)                   driver 3.1.4
NoSQL                  MongoDB (Java driver)                                  driver 4.11.1
Cache                  Redis (client Lettuce)                                 Lettuce 6.1.10.RELEASE
Message broker         RabbitMQ (Spring AMQP / amqp-client)                   2.4.17 / 5.33.1
Lưu file               MinIO (Java SDK)                                       8.6.0
Gọi service            Spring Cloud OpenFeign + OkHttp                        3.1.3 (Feign 11.8, OkHttp 4.9.3)
Lập lịch               Quartz (JDBC, clustered)                               2.3.2
API docs               springdoc-openapi-ui                                   1.7.0
Mapping / boilerplate  MapStruct, Lombok                                      1.5.5.Final / 1.18.40
Build                  Maven                                                  3.9.x
Đóng gói / chạy        Docker, Kubernetes                                     image nền eclipse-temurin:17
Giám sát               Micrometer + Prometheus, OpenTelemetry Java agent,     1.9.17, 2.31.1, 1.6.0
                         JMX exporter
Bộ tính HTTM           Python                                                 3.11

    FE (Web / Mobile PDA)
        |
        | HTTP + OAuth2 token
        v
    +-------------------------------------------------+         +-------------------------------+
    | Các service Java 17 / Spring Boot 2.7.18        |  token  | auth-service                  |
    |                                                 |<------->| OAuth2 Authorization Server   |
    |   cms :7000           dms :7001                 |         | (cấp và kiểm tra token)       |
    |   outlet :7002        salesforce :7003          |         +-------------------------------+
    |   product :7004       sale :7005                |
    |   integration :7006                             |
    |                                                 |
    |   Gọi nhau qua OpenFeign + OkHttp               |
    +-------------------------------------------------+
        |                |             |              |             |
        | JPA/Hibernate  |             |              |             |
        v                v             v              v             v
    [MariaDB]        [MongoDB]    [Redis cache]   [RabbitMQ]     [MinIO]
        ^
        | ghi kết quả
    httm-engine (Python 3.11) ---- SELECT ----> [Apache Doris]

2. Ngôn ngữ và build
====================

Thành phần                           Version       Ghi chú
-----------------------------------  ------------  -----------------------------------------------------------
Java                                 17            Mọi service và common. Riêng promotionprogram/core biên
                                                     dịch Java 8, promotionprogram/pp-core Java 11,
                                                     integration-service/shaded-keycloak-admin-client Java 11

Maven                                3.9.x         Không có Maven wrapper

maven-compiler-plugin                3.13.0        release 17, annotation processor: Lombok, MapStruct,
                                                     hibernate-jpamodelgen

maven-surefire-plugin /              3.1.2         Surefire bật testFailureIgnore=true ở các core module; test
  maven-failsafe-plugin                              @Tag("integration") chạy bằng failsafe

maven-dependency-plugin              3.6.0         Copy aspectjweaver và spring-instrument vào target/ để
                                                     Docker image nạp làm javaagent

build-helper-maven-plugin            3.3.0         Thêm target/generated-sources (JPA metamodel, MapStruct)
                                                     vào classpath

jacoco-maven-plugin                  0.8.10        Báo cáo coverage

maven-shade-plugin                   3.5.3         Chỉ ở shaded-keycloak-admin-client

spring-boot-maven-plugin             2.7.18        Đóng gói fat jar <svc>-service.jar

Kho artifact                         Nexus nội bộ  maven-releases / maven-snapshots; artifact của dự án là
                                                     1.1-SNAPSHOT

3. Spring và Spring Cloud
=========================

Tất cả service kế thừa spring-boot-starter-parent 2.7.18 và import spring-cloud-dependencies
2021.0.3. Một số version trong BOM bị ghi đè lên bản cao hơn (xem mục 11).

Thành phần                             Version                     Dùng ở đâu / để làm gì
-------------------------------------  --------------------------  -------------------------------------------
Spring Boot                            2.7.18                      Mọi service

Spring Framework                       5.3.34                      Ghi đè từ 5.3.31 của BOM

Spring Security                        5.7.12                      OAuth2 resource server (kiểm tra token qua
                                                                     user-info-uri), OAuth2 client
                                                                     client_credentials cho Feign

Spring Data (release train 2021.2.18)  JPA 2.7.18, MongoDB         Repository JPA, Mongo, cache Redis
                                         3.4.18, Redis 2.7.18

Spring AMQP                            2.4.17                      RabbitMQ (log KPI, ack thủ công, prefetch
                                                                     20)

Spring Retry                           1.3.4                       Retry khi gọi ngoài

Spring AOP / spring-aspects + AspectJ  AspectJ 1.9.7               TenantFilterAspect qua load-time weaving
                                                                     (bật ở dms, outlet, product, sale,
                                                                     salesforce; integration đã comment
                                                                     @EnableLoadTimeWeaving)

spring-instrument                      5.3.34                      Javaagent bắt buộc cho load-time weaving

Spring WebSocket                       2.7.18 (starter)            Chỉ sale-service, kèm javax.websocket-api
                                                                     1.1

Spring Cloud Config client             3.1.3                       Profile k8s-config lấy cấu hình từ config
                                                                     server; dev/qa/k8s tắt

Spring Cloud OpenFeign                 3.1.3                       Gọi giữa các service; Feign 11.8,
                                                                     feign-okhttp, timeout 5 s kết nối / 300 s
                                                                     đọc

Spring Cloud Sleuth                    3.1.3                       Tracing, Brave 5.13.9

Spring Cloud Circuit Breaker           2.1.3                       Chỉ integration-service:
                                                                     resilience4j-spring-boot2 1.7.0

springdoc-openapi-ui                   1.7.0                       Swagger UI /swagger-ui.html, OpenAPI 3

Embedded Tomcat                        9.0.118                     Ghi đè từ 9.0.83, bật HTTP/2, nén response

4. Dữ liệu và lưu trữ
=====================

Thành phần             Version                         Ghi chú
---------------------  ------------------------------  -------------------------------------------------------
MariaDB JDBC driver    3.1.4                           Mọi service (pp-service mẫu dùng 3.1.2)
Hibernate ORM          5.6.15.Final                    ddl-auto: none, filter tenant tenantFilter
Hibernate Validator    6.2.5.Final                     Bean Validation
hibernate-jpamodelgen  5.6.15.Final                    Sinh JPA metamodel lúc build
HikariCP               4.0.3                           Connection pool, mặc định tối đa 30 kết nối
MongoDB Java driver    4.11.1                          Ghi đè từ 4.6.1
Redis client Lettuce   6.1.10.RELEASE                  spring.cache.type: redis, kèm commons-pool2 2.11.1
MinIO Java SDK         8.6.0                           Lưu ảnh và file, có proxy URL ẩn link MinIO gốc
Quartz                 2.3.2                           JobStore JDBC, clustered, bảng QRTZ_* (dms, outlet,
                                                         product, sale, salesforce)
Migration schema       Script SQL viết tay             resources/db/migration/V<yyyyMMdd>__<mô tả>.sql,
                                                         idempotent; service không nhúng Flyway hay Liquibase.
                                                         Liquibase 4.3.1 chỉ có trong project mẫu
                                                         promotionprogram/service
Apache Doris           (không ghi version trong repo)  Kho dữ liệu httm-engine đọc qua giao thức MySQL, chỉ
                                                         SELECT

5. Bảo mật và xác thực
======================

Thành phần                                Version                    Ghi chú
----------------------------------------  -------------------------  -----------------------------------------
Spring Security OAuth2 resource server /  5.7.12                     Mọi service; token kiểm tra qua
  client                                                               OAUTH2_USER_INFO_URI, mỗi service có
                                                                       client riêng client_credentials

auth-service:                             2.1.2.RELEASE              Lấy từ BOM
  spring-security-oauth2-autoconfigure                                 spring-cloud-openfeign-dependencies;
                                                                       kéo theo spring-security-oauth2
                                                                       2.3.4.RELEASE
                                                                       (@EnableAuthorizationServer,
                                                                       @EnableResourceServer)

integration-service:                      2.6.8 / 2.5.2.RELEASE      Khai version cứng trong pom
  spring-security-oauth2-autoconfigure /
  spring-security-oauth2

Keycloak Admin Client (shaded)            26.0.6                     Module shaded-keycloak-admin-client của
                                                                       integration-service, kèm RESTEasy
                                                                       6.2.9.Final, Jackson 2.17.2,
                                                                       jakarta.ws.rs-api 3.1.0

Nimbus JOSE + JWT                         9.37.2                     Pin trong dependencyManagement

json-smart                                2.4.9                      Pin trong dependencyManagement

Kaptcha (pro.fessional)                   2.3.3                      dms-service, captcha

GoogleAuth (TOTP)                         1.5.0                      dms-service, mã OTP

Google Auth Library / Play Integrity API  1.23.0 /                   dms-service, kiểm tra tính toàn vẹn ứng
                                            v1-rev20250828-2.0.0       dụng Android

Bucket4j                                  8.10.1                     integration-service, giới hạn tần suất
                                                                       gọi API

Viettel File Signer +                     2.3 / 1.0                  integration-service, ký số file; jar cài
  additionalbouncycastle                                               tay vào ~/.m2

BouncyCastle bcprov-jdk15on /             1.63 / 1.49                integration-service
  bcpkix-jdk15on

Apache Santuario xmlsec                   1.5.1                      integration-service, ký XML

6. Thư viện tiện ích, mapping, JSON
===================================

Thành phần                              Version         Ghi chú
--------------------------------------  --------------  ------------------------------------------------------
Lombok                                  1.18.40         Mọi module (pp-service mẫu: 1.18.26)
MapStruct (+ processor)                 1.5.5.Final     7 core service; lombok-mapstruct-binding 0.2.0
ModelMapper                             3.1.1 / 3.2.1   3.1.1 ở product, integration; 3.2.1 ở salesforce
Jackson                                 2.18.8          Ghi đè từ 2.13.5; có jackson-dataformat-xml,
                                                          jackson-datatype-jsr310
Gson                                    2.10.1          Pin ở dms, outlet, sale, integration; cms theo BOM
                                                          2.9.1
org.json                                20231013
Guava                                   31.1-jre        dms-service
Apache Commons Lang3                    3.12.0
Apache Commons IO                       2.14.0          cms, salesforce, sale dùng 2.16.1; dms dùng 2.20.0
Commons BeanUtils / Commons FileUpload  1.11.0 / 1.6.0  Pin trong dependencyManagement
Javassist                               3.30.2-GA
JAXB API / runtime                      2.3.1
SnakeYAML                               1.33            Ghi đè từ 1.30
Netty                                   4.1.137.Final   Ghi đè từ 4.1.101.Final
TERASOLUNA GFW common                   5.7.0.RELEASE   common-base, ResultMessages / BusinessException
ULID (ulid-java)                        0.1.0           sale-service
Java UUID Generator                     5.1.0           integration-service
java-object-diff                        0.95            integration-service
datasource-proxy                        1.10.1          integration-service, log câu SQL

7. Excel, báo cáo, PDF, ảnh
===========================

Thành phần               Version    Ghi chú
-----------------------  ---------  ---------------------------------------------------------------
Apache POI / POI-OOXML   5.2.3      common-util, outlet, product (các service khác nhận qua common)
Poiji                    3.1.1      common-util, map Excel → object
JXLS (jxls-poi)          2.14.0     common-base, xuất Excel theo template
JasperReports (+ fonts)  6.21.0     sale-service, báo cáo
iText PDF                5.5.13.3   sale, integration
ICU4J                    72.1       sale-service
Thumbnailator            0.4.20     common-base, thu nhỏ ảnh
ZXing core / javase      3.5.1      dms-service, QR code
JavaMail (javax.mail)    1.5.0-b01  cms, dms
Snappy Java              1.1.10.4   Pin trong dependencyManagement

8. Log, giám sát, tracing
=========================

Thành phần                      Version         Ghi chú
------------------------------  --------------  --------------------------------------------------------------
Logback                         1.2.13          Ghi đè từ 1.2.12
SLF4J                           1.7.36
Logstash Logback Encoder        7.3             Log JSON, Filebeat đẩy về ELK (label
                                                  collect_logs_with_filebeat trong docker-compose)
Spring Boot Actuator            2.7.18          Expose health, info, prometheus, metrics
Micrometer registry-prometheus  1.9.17          Theo BOM
Micrometer core                 1.15.12         Pin trong dependencyManagement, lệch với registry 1.9.17
Spring Cloud Sleuth / Brave     3.1.3 / 5.13.9
OpenTelemetry Java agent        2.31.1          Nạp qua biến $OTEL_JAVA_OPS trong Dockerfile
JMX Prometheus javaagent        1.6.0           Cổng 9101

9. Kiểm thử
===========

Thành phần                          Version        Ghi chú
----------------------------------  -------------  -----------------------------------------------------------
JUnit Jupiter                       5.8.2          Theo BOM; một số shared module pin 5.10.0
JUnit 4 (qua junit-vintage-engine)  4.13.2         Shared module còn pin junit 4.12
Mockito core / mockito-inline       4.5.1 / 5.2.0  mockito-inline pin 5.2.0 ở dms, sale, salesforce,
                                                     promotionprogram
AssertJ                             3.27.7         Ghi đè từ 3.22.0
H2                                  2.1.214        Theo BOM; outlet pin 2.4.240, sale pin 1.4.200
mongo-java-server                   1.44.0         salesforce-service, Mongo in-memory
spring-rabbit-test                  2.4.17         sale-service
TestNG                              7.8.0          integration-service
JaCoCo                              0.8.10

10. Đóng gói, triển khai, CI/CD
===============================

Thành phần                Version / giá trị                              Ghi chú
------------------------  ---------------------------------------------  -------------------------------------
Docker base image         eclipse-temurin:17-jattach-fix-attt            Registry nội bộ
  (service Java)                                                           10.60.156.72/baseutils, timezone
                                                                           Asia/Ho_Chi_Minh, chạy user app

JVM flags                 G1GC, MaxRAMPercentage=75                      Heap dump khi OOM vào /app/logs

Javaagent trong image     aspectjweaver-1.9.7, spring-instrument-5.3.34  Ở dms, outlet, product, sale,
                                                                           salesforce; không có ở auth, cms,
                                                                           integration

Docker base image         eclipse-temurin:8-jre
  pp-service mẫu

Docker Compose            file format 2.1                                Dùng cho chạy local / dev, network
                                                                           dms4_network

Kubernetes                                                               Service chạy trên K8s (profile k8s,
                                                                           k8s-config); httm-engine ở
                                                                           namespace dms-report

Container registry        Harbor b2b-harbor.thmilk.vn                    Image dms4/httm-engine

Source control            GitLab                                         Mỗi service là 1 repo riêng

CI                        Jenkins                                        Job theo môi trường, mvn clean
                                                                           install rồi docker build

Artifact repository       Nexus                                          Shared jar 1.1-SNAPSHOT

11. Version bị ghi đè so với BOM Spring Boot 2.7.18
===================================================

Các core / shared module khai lại các property và dependencyManagement dưới đây để vá lỗ hổng bảo
mật (ATTT); ví dụ ở product-service là commit 2ac1fe8f5 "fix ATTT" ngày 15/09/2026. Khi nâng Spring
Boot cần rà lại danh sách này.

Thư viện                           BOM 2.7.18       Đang dùng
---------------------------------  ---------------  ----------------------------------------------------------
Spring Framework                   5.3.31           5.3.34
Spring Security                    5.7.11           5.7.12
Tomcat                             9.0.83           9.0.118
Jackson                            2.13.5           2.18.8
Netty                              4.1.101.Final    4.1.137.Final
SnakeYAML                          1.30             1.33
MongoDB driver                     4.6.1            4.11.1
Logback                            1.2.12           1.2.13
RabbitMQ amqp-client               5.14.3           5.33.1
Micrometer core                    1.9.17           1.15.12
AssertJ                            3.22.0           3.27.7
Commons IO                         (không quản lý)  2.14.0 / 2.16.1 / 2.20.0
Thêm mới vào dependencyManagement                   MinIO 8.6.0, iText 5.5.13.3, Snappy 1.1.10.4, Commons
                                                      FileUpload 1.6.0, Commons BeanUtils 1.11.0, Nimbus JOSE
                                                      9.37.2, json-smart 2.4.9

12. Thư viện CTKM promotionprogram
==================================

Thư viện riêng, version 1.0-SNAPSHOT, được sale-service dùng cho các lớp Pp*.

Module                            Java  Phụ thuộc chính
--------------------------------  ----  ----------------------------------------------------------------------
core                              8     Lombok 1.18.40; test: Mockito 5.2.0

pp-core                           11    Lombok 1.18.40, SLF4J API 1.7.26; test: JUnit 5.10.0, Mockito 5.2.0

service (project mẫu pp-service)  8     Spring Boot 2.3.12.RELEASE, Spring Data JPA, Liquibase 4.3.1, MariaDB
                                          driver 3.1.2, Commons Codec 1.15

13. Bộ tính HTTM trả sau httm-py (httm-engine)
==============================================

Thực hiện bước tổng hợp kết quả tích lũy / trưng bày của blueprint 3.2.1.

Thành phần     Version / giá trị                            Ghi chú
-------------  -------------------------------------------  --------------------------------------------------
Python         3.11                                         Build trên python:3.11-slim-bookworm, chạy trên
                                                              gcr.io/distroless/python3-debian12:nonroot
PyMySQL        1.1.1                                        Đóng kèm trong vendor/, không cài qua pip
Thư viện khác  Chỉ thư viện chuẩn Python                    http.server cho API, multiprocessing /
                                                              concurrent.futures để chạy song song
Nguồn đọc      Apache Doris                                 Chỉ SELECT
Nơi ghi        MariaDB (bảng ntmp_*)                        Chỉ ghi phần chênh lệch, không đè dòng đã duyệt
Triển khai     Kubernetes namespace dms-report              CronJob */20 * * * * + Deployment API cổng 8080
Image          b2b-harbor.thmilk.vn/dms4/httm-engine:1.3.0

14. Ghi chú
===========

- Phiên bản đã hết hỗ trợ: Spring Boot 2.7.x hết hỗ trợ bản OSS từ 11/2023, Spring Cloud 2021.0 cũng
  vậy. auth-service còn dùng Spring Security OAuth (@EnableAuthorizationServer), dự án này đã ngừng
  phát triển từ 05/2022. Đây là lý do phải ghi đè nhiều version ở mục 11.
- Version lệch giữa các service: Commons IO (2.14.0 / 2.16.1 / 2.20.0), H2 (1.4.200 / 2.1.214 /
  2.4.240), ModelMapper (3.1.1 / 3.2.1), Gson (2.9.1 / 2.10.1), OAuth2 legacy giữa auth-service
  (2.1.2 / 2.3.4) và integration-service (2.6.8 / 2.5.2). Micrometer core (1.15.12) cũng không cùng
  version với registry Prometheus (1.9.17).
- Hai cách nhìn về migration: CLAUDE.md ở gốc workspace nói schema đổi bằng script SQL trong
  resources/db/migration, còn product-service/CLAUDE.md nói changelog Liquibase nằm ở repo
  liquibase-service riêng (không có trong workspace). Cả hai service đều không có dependency
  Liquibase/Flyway.
- Version hạ tầng chạy thực tế (MariaDB server, MongoDB server, Redis, RabbitMQ, MinIO, Doris,
  Kubernetes) không được ghi trong repo; bảng trên chỉ ghi version client/driver.