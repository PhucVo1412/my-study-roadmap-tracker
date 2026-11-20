# Đánh giá API giai đoạn 1

- **Ngày:** 13/11/2026
- **Phạm vi:** các API có trong collection Bruno.
  - Trọng tâm là HTTM trả sau (`sale-service`, `/rebateProgram/**`): 33 controller, 214 endpoint, 85 class service.
  - Đối chiếu thêm một số module tiêu biểu: nhóm sản phẩm (`product-service`, `ProductGroupManagementController`), phân loại KH (`outlet-service`, `CustomerTypeController`), system param (`cms-service`, `SystemParameterEntryController`), phân bổ KPI (`salesforce-service`, `StaffKpiAllocationController`).
- **Cách làm:** đọc controller và service impl trên code đang checkout. Các con số dưới đây được đếm trực tiếp từ code.

## 1. Điểm tốt

1. **Phân tầng rõ ràng:** controller → service → repository. DTO nằm ở module `*-shared-service` để các service khác dùng lại qua Feign. Phần lớn controller HTTM chỉ nhận tham số rồi gọi service.
2. **URL nhất quán ở cấp cao:**
   - Mọi API theo dạng `/api/v1/<service>/<resource>`, nhờ `BaseController` dùng chung.
   - Riêng HTTM có đường dẫn phân cấp theo nghiệp vụ (chương trình → `cycles` / `schemes` → `levels` → `accum-conditions`, `reward-rules`…) và dùng kebab-case.
   - HTTP method đúng nghĩa. Xóa hàng loạt qua `?ids=`.
   - Có các PUT ghi đè cả bộ, với quy tắc rõ ràng: phần tử có `id` thì cập nhật, không có `id` thì thêm mới, phần tử không gửi lên thì bị xóa.
3. **Multi-tenant làm ở tầng hạ tầng:** tenant filter của Hibernate tự thêm `tenant_id` vào query, nên không phải lặp điều kiện này ở từng câu. Native query thì truyền `tenantId` rõ ràng.
4. **Service HTTM làm đúng nhiều việc:**
   - **Transaction có chủ đích:** 174 `@Transactional`, trong đó 40 là `readOnly`.
   - **Query theo lô, tránh N+1:** màn danh sách chương trình lấy ngân sách, kỳ và phạm vi bằng một query cho cả trang, không query từng dòng.
   - **Kiểm tra quyền gom về một chỗ** (`RebateProgramAccess`). Kết quả lời gọi cms được cache trong request.
   - **Lỗi import được đổi thành `BusinessException`** với câu tiếng Việt cho người dùng.
5. **Test và tài liệu API:**
   - Service HTTM có 127 file test với 1.373 test.
   - 123/214 endpoint có `@Operation` cho Swagger.
   - Các endpoint mới bắt đầu trả kiểu cụ thể `ResponseEntity<Dto>` (30 endpoint).

## 2. Điểm cần cải thiện

### 2.1 Controller

| Vấn đề | Số liệu | Hệ quả |
|---|---|---|
| Trả về `ResponseEntity<?>` | HTTM 105 chỗ, so với 30 chỗ có kiểu cụ thể. `productGroup` 12/20 endpoint. `customerType`, `systemParamEntry`, `staffKpi` gần như mọi endpoint | Swagger không biết kiểu response. FE phải đoán hoặc gọi thử để biết response trông thế nào |
| Gần như không dùng Bean Validation (`@Valid`) | HTTM 18/214. 0 ở cả 4 controller còn lại | Kiểm tra đầu vào viết tay rải rác trong service, mỗi nơi một kiểu |
| Kiểm tra đầu vào ngay trong controller, trả chuỗi tiếng Anh | 5 chỗ, ví dụ `badRequest().body("Contains status unsupported !")` | Thành thêm một dạng lỗi nữa, ngoài `ApiError` và `{code, message}` (xem 2.3) |
| Chữ tiếng Việt bị lỗi font trong source | `RebateProgramController`: 2 `@Operation`. `RebateProgramServiceImpl`: 10 dòng, gồm cả nhãn hiển thị cho người dùng như `"CÃ i Ä‘áº·t chÆ°Æ¡ng trÃ¬nh"` thay vì "Cài đặt chương trình". Lỗi này có trên `develop` | **Đây là bug:** người dùng thấy chữ lỗi trên màn duyệt, Swagger cũng hiện chữ lỗi |

**Đặt tên path không thống nhất:**

- **Lẫn camelCase và kebab-case:** `productGroup`, `staffKpi` cạnh `out-payment`, `reward-rules`.
- **Động từ trong path:** `createProductGroup`, `editProductGroup`, `getStaffKpiAll`, `findLstTitle`.
- **Lộ tên bảng hoặc class nội bộ:** `save/saveProductInSysProductGroupRef`.
- **Viết tắt khó hiểu:** `cbbProductGroup`, `Lst`.
- **Hai endpoint danh sách song song:** `productGroup/list` và `productGroup/search`.
- **Năm cách lấy chi tiết theo id:**
  - `schemes/{id}`
  - `rebateProgram/id?id=`
  - `reward-rules/detail?id=`
  - `productGroup?id=`
  - `product/lotdate/id?productId=`

  Ngay trong HTTM đã có ba cách.
- **Excel mỗi màn đặt tên một kiểu:** khoảng 10 kiểu cho tải template (`excel/template`, `excelTemplate`, `getTemplate`, `export-template`…) và khoảng 10 kiểu cho export.
- **Tên tham số lệch nhau:** `idProductGroup` và `productGroupId`, `distIds` và `distUnitIds`. Có API cho FE vẫn nhận `tenantId` qua query, trong khi tenant nên lấy từ token.

**Dùng HTTP method chưa đúng nghĩa:**

- **Xóa bằng PUT** (`PUT productGroup/delete`). **DELETE có body JSON** (`productGroup/deleteProductInProductGroup`), và một số proxy bỏ body của DELETE.
- **POST tạo mới / sao chép nhưng dữ liệu ở query:** `rebateProgram/copy?id=&name=&fromDate=&toDate=`, `schemes/levels/copy`, `document-types?name=`. Tên tiếng Việt phải mã hóa trong URL, và dữ liệu lọt vào access log.
- **Cùng hành động "tạm ngưng" mà mỗi nơi một kiểu:** `customerType/pending/`, `productGroup/pendingProductGroup`, `rebateProgram/suspend`.
- **"Lưu ghi đè" khi dùng POST** (`rebateProgram/description`), **khi dùng PUT** (các form HTTM khác).

### 2.2 Service impl

| Vấn đề | Số liệu / ví dụ | Đề xuất |
|---|---|---|
| Class quá lớn (god class) | `RebateProgramServiceImpl` 3.370 dòng, `ProductGroupManagementServiceImpl` 3.358, `RebateAccumConditionServiceImpl` 2.310, `RebateScopeServiceImpl` 2.296 | Tách theo use case (đọc / ghi / import / export). HTTM đã đi đúng hướng ở phần Excel (`RebateProgramWorkbook`, `RebateProgramDetailExport`, `RebateProgramScopeImport`) |
| Truyền `HttpServletRequest` xuống service | 268 chỗ trong service HTTM, 18 trong `CustomerTypeServiceImpl` | Service bị dính vào tầng web. Khi chạy ngoài request (import từng dòng, async, job) phải vá bằng cờ: `RebateWorkbookImportContext.isActive()` xuất hiện 10 chỗ. Controller nên lấy sẵn user, token, tenant rồi truyền giá trị xuống |
| Ba cơ chế báo lỗi | `BusinessException` 73 (→ 409), `ResponseStatusException` 43 (→ mã riêng, chỉ được xử lý gọn trong package HTTM), `IllegalArgumentException` / `IllegalStateException` 52 (→ **500** nếu lọt ra controller) | Ví dụ: `applyGeneralInfo` ném `IllegalArgumentException("isSaleOrderRequired must be 0 or 1")` khi tạo hoặc sửa chương trình, nên FE nhận 500 "Xảy ra lỗi hệ thống" thay vì 400 kèm lý do. Lỗi đầu vào và nghiệp vụ nên dùng `BusinessException`. `IllegalStateException` chỉ dành cho lỗi lập trình |
| Kiểu inject trộn lẫn | HTTM: 203 field `@Autowired`, 45 class dùng `@RequiredArgsConstructor` | Dùng constructor injection cho code mới. Không dựa vào tên biến để chọn bean khi có 2 bean cùng kiểu (vụ `MinioService` làm develop không start được) |
| Phân trang trong bộ nhớ | HTTM: 16 chỗ `new PageImpl`, 8 chỗ `.skip()` | Màn danh sách chương trình tải **toàn bộ** chương trình user được xem rồi mới cắt trang, vì quyền được lọc bằng Java. Dữ liệu lớn sẽ chậm. Nên đẩy điều kiện quyền xuống query |
| Mapping trộn hai cách | 16 file dùng MapStruct, 11 hàm `toDTO` viết tay | Nhẹ. Nên chọn một cách cho code mới |

### 2.3 Response

- **Hai dạng JSON lỗi cùng tồn tại:** trong package HTTM, `RebateExceptionHandler` trả `{code, message}`; các trường hợp còn lại trả `ApiError` của common: `{code, message, target, details, traceId}`. Thêm cả chuỗi trần từ `badRequest().body("...")` là ba dạng.
- **Không có lớp bọc response chung.** Điều này chấp nhận được, nhưng cần ghi rõ cho FE.
- **`traceId` trả về FE gần như không dùng được:** lỗi dạng `{code, message}` không có `traceId`, và log hiện chưa ghi `traceId` nên không tra ngược được.

## 3. Đề xuất theo thứ tự ưu tiên

API đã có FE dùng thì **không đổi path**. Các mục 3–5 áp dụng cho code mới, hoặc khi sửa tới chỗ đó.

1. **Sửa chữ lỗi font** trong `RebateProgramController` và `RebateProgramServiceImpl`, rồi kiểm tra dữ liệu nhãn đã lưu xuống DB. Đây là bug người dùng nhìn thấy.
2. **Lỗi đầu vào ném `BusinessException`** thay vì `IllegalArgumentException` hoặc `badRequest().body("...")`. Thống nhất về một dạng JSON lỗi.
3. **Quy ước cho API mới:**
   - kebab-case, không động từ trong path;
   - `GET /<resource>/{id}`;
   - bộ Excel chuẩn: `excel/template`, `excel/import`, `excel/export`, `excel/export/by-ids`;
   - dữ liệu POST/PUT đi trong body JSON;
   - tên tham số dạng `<resource>Id`;
   - trả về `ResponseEntity<Dto>` thay vì `ResponseEntity<?>`.
4. **Service mới không nhận `HttpServletRequest`.** Dùng constructor injection.
5. **Tách dần các god class** khi sửa tới. Với danh sách lớn, đẩy lọc quyền và phân trang xuống DB.