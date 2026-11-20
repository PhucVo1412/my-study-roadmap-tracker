Luồng một request end-to-end
============================

Tài liệu mô tả đường đi của một request từ FE tới một service backend: xác thực token, xác định
tenant, controller → service → DB, gọi sang service khác qua Feign, rồi trả response hoặc lỗi về FE.

- Ví dụ xuyên suốt: API tìm kiếm danh sách chương trình HTTM trả sau trên sale-service, vì API này
  đi qua đủ các bước, kể cả gọi sang cms-service và dms-service.
      GET /api/v1/sale-service/rebateProgram?salesOrgIds=X&page=0&size=10
      Authorization: Bearer <token của user>

- Code đọc ngày 13/11/2026: sale-service nhánh feature/phucvh11/httm-fixbug, common nhánh develop.
  Các service khác (cms, dms, outlet, product, …) dựng theo cùng khuôn nên luồng giống nhau, chỉ
  khác tên class.

1. Tổng quan thành phần
=======================

    FE (Web / Mobile)
      |
      | HTTP + Bearer token của user
      v
    [sale-service :7005]
      1. Filter Spring Security: kiểm tra Bearer token ----- "token này của ai?" -----> auth-service
      2. Filter AccessLogFilter, ContentTypeFilter
      3. Interceptor LogRequestInterceptor, TenantInterceptor
      4. Controller (rest)
      5. Service (service/impl)
           +--> Repository (Spring Data JPA) ------------------------------------------> MariaDB
           +--> Feign client (client)
                  +--> auth-service : lấy token client_credentials
                  +--> cms-service  : Bearer token sale-service + tenantId, accessToken
                  +--> dms-service  : Bearer token sale-service + tenantId, accessToken
      *  Exception handler (@ControllerAdvice): RebateExceptionHandler, ErrorHandle
         nhận exception ném ra từ controller trở xuống

    Trả về FE:
      - Thành công: controller trả 200, ResponseEntity được chuyển thành JSON.
      - Lỗi: exception handler trả 4xx/5xx, JSON lỗi.

Đường dẫn trong bảng tính từ thư mục mã nguồn của module:
<svc>-service/<svc>-core-service/src/main/java/vn/viettel/dms4/services/<svc>/ với service,
common/<module>/src/main/java/vn/viettel/dms4/ với common.

Thành phần                        File
--------------------------------  ----------------------------------------------------------------------------
Cấu hình security, interceptor    sale: config/SecurityConfig.java
  gắn token cho Feign

Hỏi auth-service token là của ai  common-base: commons/security/CustomUserInfoTokenIntrospector.java

Đăng ký interceptor               sale: config/WebConfiguration.java

Xác định tenant                   sale: interceptor/TenantInterceptor.java

Bật tenant filter cho Hibernate   common-domain: commons/multitenancy/aspect/TenantFilterAspect.java

Controller ví dụ                  sale: rest/traderebate/RebateProgramController.java
  (getByCondition)

Service ví dụ (getByCondition,    sale: service/impl/traderebate/RebateProgramServiceImpl.java
  searchPrograms)

Kiểm tra quyền chức năng HTTM     sale: service/traderebate/RebateProgramAccess.java

Feign client sang cms / dms       sale: client/CmsServiceClient.java, sale: client/DmsServiceClient.java

OkHttp, ErrorDecoder của Feign    sale: config/FeignClientConfig.java

API phía dms được gọi tới         dms: rest/DistUnitController.java (getChildsOrParentsByOption)

Chuyển exception thành JSON lỗi,  sale: rest/traderebate/RebateExceptionHandler.java
  riêng API HTTM (ưu tiên cao
  nhất)

Chuyển exception thành JSON lỗi,  sale: config/ErrorHandle.java → common-base:
  chung cho mọi API                 commons/control/common/error/ApiGlobalExceptionHandler.java

2. Luồng đầy đủ
===============

    Thành phần:
      FE   = FE
      SEC  = sale: Spring Security
      AUTH = auth-service
      INT  = sale: TenantInterceptor
      CTL  = sale: RebateProgramController
      SVC  = sale: RebateProgramServiceImpl
      CMS  = cms-service
      DMS  = dms-service
      DB   = MariaDB

     1. FE   -> SEC  : GET /api/v1/sale-service/rebateProgram?salesOrgIds=X
                       Authorization: Bearer token user
     2. SEC  -> AUTH : GET userinfo, Bearer token user
     3. AUTH -> SEC  : principal (id, username, tenantId), details.tokenValue, oauth2Request
        Nếu token sai hoặc hết hạn:
     4.   SEC  -> FE   : 401, dừng tại đây, không vào controller
     5. SEC  -> INT  : request đã xác thực (BearerTokenAuthentication)
     6. INT  -> INT  : tenantId lấy từ param, nếu không có thì từ token
                       TenantContext.setTenantId
     7. INT  -> CTL  : getByCondition(searchText, salesOrgIds, ..., Pageable)
     8. CTL  -> SVC  : getByCondition(criteria, pageable, request)
     9. SVC  -> CMS  : Feign getByUserRoleId(accessToken = token user, tenantId)
    10. CMS  -> SVC  : cây chức năng của user
        Nếu user không có chức năng HTTM:
    11.   SVC  -> FE   : AccessDeniedException thành 403 ACCESS_DENIED, xem mục 5
    12. SVC  -> DMS  : Feign getChildsOrParentsByOption(ids, accessToken, hierarchyGetType, tenantId)
    13. DMS  -> SVC  : danh sách id Sales Org (cha, con, chính nó)
    14. SVC  -> DB   : query qua Hibernate, tenantFilter tự thêm tenant_id
    15. DB   -> SVC  : chương trình, ngân sách, kỳ, phạm vi, trạng thái duyệt
    16. SVC  -> SVC  : map sang RebateProgramSearchDTO, new PageImpl
    17. SVC  -> CTL  : Page RebateProgramSearchDTO
    18. CTL  -> FE   : 200, ResponseEntity được Jackson chuyển thành JSON

    Ghi chú (INT): afterCompletion: TenantContext.clear, chạy cả khi có lỗi

Bước 1. Xác thực token (filter)
-------------------------------

- SecurityConfig bắt mọi API phải có token, trừ swagger, actuator, file tĩnh, /**/ws/** và
  /**/unauthorized/**.
- Token là loại opaque (chuỗi không tự giải mã được). CustomUserInfoTokenIntrospector cầm token gọi
  GET <OAUTH2_USER_INFO_URI> sang auth-service, timeout kết nối 3s, đọc 5s.
- auth-service trả Map thông tin user. Map được bọc thành BearerTokenAuthentication và cất vào
  SecurityContext. Các bước sau đọc user, tenant, token gốc (details.tokenValue) từ đây.
- Token sai hoặc hết hạn → Spring Security trả 401 luôn. Lỗi này không đi qua ErrorHandle.
- Sau đó là filter phụ: AccessLogFilter ghi log KPI (chỉ cho request từ FE, bỏ qua request Feign),
  ContentTypeFilter thêm header Vary và Content-Type mặc định.

Bước 2. Xác định tenant (interceptor)
-------------------------------------

WebConfiguration đăng ký LogRequestInterceptor và TenantInterceptor. Chi tiết cách lấy tenant ở
mục 4.

AuthorizationFilter (phân quyền theo URL, cấu hình trong Mongo) đang bị comment trong
WebConfiguration. Hiện tại từng tính năng tự kiểm tra quyền, ví dụ HTTM dùng RebateProgramAccess ở
bước 4a.

Bước 3. Controller
------------------

- BaseController gắn sẵn /api. RebateProgramController ghép V1 + "/sale-service/rebateProgram" →
  /api/v1/sale-service/rebateProgram.
- Spring tự đổi query string thành tham số: @RequestParam List<String> salesOrgIds, Pageable từ
  page, size, sort.
- Controller chỉ gom tham số thành RebateProgramQueryCriteria và gọi service. Logic nằm ở service.

Bước 4. Service
---------------

RebateProgramServiceImpl.getByCondition → searchPrograms:

- a. Kiểm tra quyền: requireFeature("READ") → RebateProgramAccess gọi Feign
  cmsServiceClient.getByUserRoleId(token, tenant) để lấy cây chức năng của user. Kết quả được cache
  vào request.setAttribute, nên trong một request chỉ gọi cms một lần. User không có chức năng HTTM
  → AccessDeniedException → FE nhận 403 ACCESS_DENIED.
- b. Gọi dms: mở rộng các Sales Org được chọn thành cả cha, con và chính nó qua
  dmsServiceClient.getChildrenOrParentsByOption(...). Chi tiết ở mục 3.
- c. Query DB: repository Spring Data JPA → Hibernate → MariaDB. Tenant được lọc tự động, xem mục 4.
- d. Gom dữ liệu: lấy thêm ngân sách, yêu cầu chỉnh sửa đang chờ, trạng thái duyệt, phạm vi, kỳ. Mỗi
  loại query một lần cho cả lô programIds, không query từng dòng. Ghép thành RebateProgramSearchDTO
  và trả new PageImpl<>(content, pageable, total).

Bước 5. Trả response
--------------------

    return new ResponseEntity<>(rebateProgramService.getByCondition(criteria, pageable, request), HttpStatus.OK);

- Jackson chuyển Page thành JSON {content, pageable, totalElements, totalPages, ...}.
- Không có lớp bọc chung kiểu {code, data}. Khi thành công, FE nhận đúng DTO / Page / List mà
  service trả về, HTTP 200.
- Request xong: TenantInterceptor.afterCompletion xóa TenantContext, AccessLogFilter ghi log KPI.

3. Gọi service khác qua Feign
=============================

    Thành phần:
      SVC  = sale: RebateProgramServiceImpl
      PX   = Feign proxy DmsServiceClient
      TI   = oauth2FeignRequestInterceptor
      AUTH = auth-service
      OK   = OkHttp
      DMS  = dms-service
      DEC  = Feign decoder, ErrorDecoder

     1. SVC  -> PX   : getChildrenOrParentsByOption(ids, token user,
                       "GET_CHILDREN_AND_PARENTS_AND_SELF", tenantId)
     2. PX   -> PX   : ghép URL feign.dms-service + /api/v1/dms-service/distUnit/getChildsOrParentsByOption
                       param ids, accessToken, hierarchyGetType, tenantId
     3. PX   -> TI   : apply(requestTemplate)
        Nếu request đã tự mang header Authorization:
     4.   TI   -> PX   : giữ nguyên header đó
        Ngược lại, chưa có (trường hợp thường gặp):
          Nếu token của sale-service trong cache đã hết hạn:
     5.     TI   -> AUTH : client_credentials, client-id sale-service
     6.     AUTH -> TI   : access token mới
     7.   TI   -> PX   : thêm Authorization Bearer token sale-service
     8. PX   -> OK   : gửi HTTP (pool 200 kết nối, connect 5s, read 300s)
     9. OK   -> DMS  : GET .../distUnit/getChildsOrParentsByOption, param ids, tenantId, ...
        Ghi chú (DMS): chạy lại đúng bước 1-5 như sale: hỏi auth-service token của sale-service,
                       TenantInterceptor lấy tenantId từ param, DistUnitController -> service -> DB
    10. DMS  -> OK   : JSON hoặc mã lỗi
    11. OK   -> DEC  : response
        Nếu 2xx:
    12.   DEC  -> SVC  : danh sách id (Jackson decode)
        Nếu 4xx có field code:
    13.   DEC  -> SVC  : throw RemoteServiceException(code, message)
        Nếu lỗi khác, hoặc body là HTML:
    14.   DEC  -> SVC  : throw RuntimeException(message)

Feign client chỉ là interface (@FeignClient(name = "dms-service", url = "${feign.dms-service}")).
Spring tạo proxy, proxy biến lời gọi hàm thành HTTP request. Base URL lấy từ biến môi trường
(DMS_SERVICE, CMS_SERVICE, …).

Thông tin trong lời gọi Feign nằm ở đâu:

Thông tin               Nằm ở                            Vì sao
----------------------  -------------------------------  -----------------------------------------------------
Token của sale-service  Header Authorization             Để service đích cho phép gọi. Lấy bằng
                                                           client_credentials, cache đến khi hết hạn

Token của user          Param accessToken (nếu API cần)  Để service đích biết user là ai khi cần lọc theo
                                                           quyền / phạm vi của user

Tenant                  Param tenantId                   Token client_credentials không gắn với user nên không
                                                           có tenant. Mọi Feign client phải truyền tenantId,
                                                           nếu không service đích không xác định được tenant

ErrorDecoder trong FeignClientConfig đổi response lỗi thành exception phía sale:

- 4xx có code → RemoteServiceException(code, message), để nơi gọi phân biệt được từng loại lỗi
  nghiệp vụ.
- Body là HTML (service đích hỏng, hoặc gateway trả trang lỗi) → RuntimeException("Remote service
  returned non-JSON error: ...").
- Còn lại → RuntimeException(message).
- Riêng DmsServiceClient#resolveSalesOrgAllocationCandidates và #searchSalesOrgCandidates →
  ResponseStatusException giữ nguyên mã 4xx của dms.

Exception này ném ngược lên service của sale như một lỗi bình thường, rồi đi theo mục 5.

4. Tenant đi theo request
=========================

    Request tới
      |
      +-- Có param ?tenantId ?
      |     có    -> tenantId = param
      |     không -> Token có tenant_id, tenantId hoặc principal.tenantId ?
      |                có    -> tenantId = từ token
      |                không -> Có cấu hình multitenancy.tenant.default-tenant ?
      |                           có    -> tenantId = mặc định
      |                           không -> RuntimeException: can't detect tenantId
      v
    TenantContext.setTenantId (ThreadLocal, riêng cho thread đang xử lý request)
      |
      +--> Hibernate mở session -> TenantFilterAspect bật tenantFilter
      |      -> query entity tự thêm tenant_id = :tenantId
      +--> Native query: không qua filter, phải truyền tenantId bằng tay
      +--> Feign: truyền tenantId qua param
      +--> Chạy async: phải qua bean taskExecutor để mang tenant và security sang thread khác
      +--> afterCompletion: TenantContext.clear

- TenantFilterAspect chạy nhờ AspectJ load-time weaving. Đây là lý do khi start service bắt buộc
  phải có -javaagent (spring-instrument, aspectjweaver).
- Entity kế thừa EntityWithTenant nên query sinh từ tên method (findByCode...) không cần thêm tham
  số tenantId.
- Vì TenantContext là ThreadLocal, code chạy trên thread khác (@Async, CompletableFuture, stream
  song song) sẽ không có tenant, trừ khi chạy qua taskExecutor (SecurityTenantTaskDecorator).

5. Lỗi được trả về FE thế nào
=============================

Spring hỏi các @ControllerAdvice theo thứ tự ưu tiên. Advice đầu tiên vừa áp dụng cho controller đó,
vừa có handler khớp loại exception, sẽ xử lý.

    Token sai ở filter Spring Security
      -> 401, không qua handler nào bên dưới

    Exception ném ra trong interceptor, controller, service hoặc Feign
      |
      +-- Controller nằm trong package rest.traderebate (HTTM)?
            |
            +-- Có: RebateExceptionHandler (ưu tiên cao nhất)
            |     AccessDeniedException                              -> 403 ACCESS_DENIED
            |     body JSON sai, @Valid trên body, sai kiểu tham số  -> 400 INVALID_REQUEST
            |     ResponseStatusException                            -> status của exception, 503 đổi thành 500,
            |                                                           code = reason của exception
            |     => trả JSON gồm code, message
            |     Loại khác, không có handler                        -> chuyển xuống ErrorHandle bên dưới
            |
            +-- Không: ErrorHandle (@ControllerAdvice), kế thừa ApiGlobalExceptionHandler
                  BusinessException(ResultMessages)                  -> 409 CONFLICT, message = câu tiếng Việt
                  ResourceNotFoundException                          -> 404
                  commons BusinessException                          -> httpCode của exception (mặc định 409)
                  @Valid, ConstraintViolation, thiếu param bắt buộc  -> 400
                  Còn lại: NPE, RuntimeException từ Feign, ...       -> 500 Xảy ra lỗi hệ thống
                  => trả JSON ApiError

Vì vậy có hai dạng JSON lỗi:

- API HTTM, lỗi do RebateExceptionHandler xử lý: {"code": "ACCESS_DENIED", "message": "..."}.
- Mọi trường hợp còn lại: ApiError của common, như bên dưới.

Cả hai dạng đều có code và message. FE chỉ cần đọc hai field này là dùng được cho mọi API.

Dạng ApiError:

    {
      "code": "...",
      "message": "Câu thông báo",
      "target": null,
      "details": [],
      "traceId": "..."
    }

- details có phần tử khi có nhiều lỗi cùng lúc, ví dụ nhiều vi phạm @Valid. Lỗi nghiệp vụ trong
  workspace chỉ ném một lỗi đầu tiên nên details thường rỗng.
- traceId do Spring Cloud Sleuth 3.1.3 sinh cho mỗi request và ghi vào MDC với khóa traceId. Nếu
  request tới đã mang header B3 thì Sleuth dùng lại id đó. Feign tự gắn header B3 khi gọi sang
  service khác, nên sale và dms dùng chung một traceId (integration-service không có Sleuth).
  Handler đọc khóa cũ X-B3-TraceId trước, nhưng Sleuth 3 không ghi khóa này nên lần nào cũng phải
  dùng khóa traceId.
- Hiện chưa tra được log bằng traceId: theo logback-spring.xml trong repo, log JSON trên k8s lấy
  trace_id từ khóa cũ %mdc{X-B3-TraceId} (luôn rỗng) và loại khóa traceId khỏi phần MDC. Pattern log
  local cũng không in traceId. Lỗi {code, message} của RebateExceptionHandler và response thành công
  không trả traceId.
- Cách ném lỗi nghiệp vụ trong workspace: throw new
  BusinessException(ResultMessages.error().add(ResultMessage.fromText("..."))) → 409.

6. Điểm cần biết
================

- Một request FE sinh thêm nhiều lời gọi mạng. Với ví dụ trên là ít nhất 4: sale → auth-service
  (kiểm tra token user), sale → cms (quyền), sale → dms, dms → auth-service (kiểm tra token của
  sale). auth-service hoặc một service phụ thuộc chậm hay chết thì API của sale chậm hoặc lỗi theo.
- Cùng một exception có thể ra mã HTTP khác nhau tùy controller. Ví dụ AccessDeniedException: ở API
  HTTM (rest/traderebate) ra 403 nhờ RebateExceptionHandler. Ở controller khác, ErrorHandle và
  ApiGlobalExceptionHandler không có handler riêng cho exception này, nên nó rơi vào handler chung
  Exception.class và ra 500.
- Lỗi từ service đích thường thành 500 ở sale. RemoteServiceException / RuntimeException do
  ErrorDecoder tạo ra không có handler riêng, trừ khi code phía sale tự bắt và đổi thành
  BusinessException. Riêng ResponseStatusException (hai method candidate của DmsServiceClient) giữ
  được mã 4xx khi đi qua API HTTM.
- Request từ service khác cũng đi qua bước 1. Mỗi lời gọi Feign tới đều tốn một lần service đích hỏi
  auth-service về token.