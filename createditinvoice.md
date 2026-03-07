
## API CREATE CREDIT INVOICE

Tài liệu mô tả chi tiết nghiệp vụ tạo hóa đơn điều chỉnh `CR` cho luồng Logistics trên `sale-im-service`, bao gồm:
- **Endpoint public phía BCCS3**
- **Request/response thực tế**
- **Luồng tạo hóa đơn theo cấu hình BCCS3 hoặc BCCS1**
- **SQL chính để lấy/lock `InvoiceList`, tạo `InvoiceUsed`, cập nhật hóa đơn gốc**

---

## 1. Tổng quan nghiệp vụ

- API public cho phía Logistics nằm ở `sale-im-service`.
- Endpoint nhận `saleTransCode`, `invoiceType`, `reason`.
- `invoiceType` chỉ chấp nhận `CR`.
- Hệ thống có 2 nhánh xử lý:
  - Nếu `useOldSystemForCreateInvoice = false` -> tạo credit invoice trực tiếp trên BCCS3.
  - Nếu `useOldSystemForCreateInvoice = true` -> gọi sang `bccs1-inventory-service` để tạo credit invoice trên hệ thống cũ.
- Mục tiêu nghiệp vụ:
  - Giao dịch phải đã có hóa đơn gốc `SC`.
  - Không được tồn tại credit invoice `CR` active trước đó cho cùng hóa đơn gốc.
  - Khi tạo thành công, hóa đơn gốc được đánh dấu đã approve điều chỉnh, đồng thời hệ thống cấp số hóa đơn `CR` mới.

---

## 2. Endpoint & Authentication

### 2.1. Endpoint

```text
POST /sale/logistics/create-credit-invoice
```

### 2.2. Authentication

- API chạy trong `sale-im-service`, theo cơ chế auth chung của service.
- Thông thường hệ thống yêu cầu Bearer Token.
- Nếu môi trường có whitelist security thì endpoint có thể được mở theo cấu hình triển khai thực tế.

---

## 3. Request / Response

### 3.1. Request Body (`CreateCreditInvoiceRequestDTO`)

```json
{
  "requestId": "REQ_20260307_0001",
  "saleTransCode": "ST202603070001",
  "invoiceType": "CR",
  "reason": "Adjust invoice for logistics request"
}
```

| Field | Type | Required | Ghi chú |
|------|------|----------|--------|
| `requestId` | String | No | Mã request để trace log và phản hồi |
| `saleTransCode` | String | Yes | Mã giao dịch bán hàng cần tạo hóa đơn điều chỉnh |
| `invoiceType` | String | Yes | Chỉ chấp nhận giá trị `CR` |
| `reason` | String | No | Lý do điều chỉnh, sẽ ghi vào `note` / `reason` của hóa đơn `CR` |

### 3.2. Response thành công

Controller trả về `ResponseMessage<CreateCreditInvoiceResponseDTO>`:

```json
{
  "code": "200",
  "success": true,
  "data": {
    "requestId": "REQ_20260307_0001",
    "saleTransCode": "ST202603070001",
    "requestStatus": "Success",
    "errorCode": "200",
    "error": null
  }
}
```

- Đây là format chắc chắn đúng với **nhánh BCCS1**, vì `createCreditInvoiceFromBccs1(...)` set `errorCode = "200"`.
- Với **nhánh BCCS3**, method `createCreditInvoiceFromBccs3(...)` hiện trả `requestStatus = "Success"` nhưng **không set `errorCode = "200"`**.
- Do controller đang check `response.getErrorCode()` để quyết định success/fail, nên nếu bám đúng code hiện tại thì nhánh BCCS3 có rủi ro bị controller coi là fail dù nghiệp vụ đã tạo invoice thành công.

### 3.3. Response lỗi

Khi nghiệp vụ fail, controller vẫn trả HTTP `200` nhưng `success = false`:

```json
{
  "code": "400",
  "success": false,
  "description": "CR invoice already exists",
  "data": null
}
```

> **Lưu ý:** Trong code hiện tại, controller dùng `toExceptionResultOk(...)`, nên lỗi nghiệp vụ vẫn đi ra với HTTP status `200`, còn mã lỗi nằm trong body.

---

## 4. Luồng tạo Credit Invoice trong hệ thống

### 4.1. Luồng tổng quát tại `sale-im-service`

1. **Nhận request từ Logistics**
   - Controller `LogisticsController#createCreditInvoice` nhận body `CreateCreditInvoiceRequestDTO`.

2. **Khởi tạo response mặc định**
   - `requestStatus = "Fail"`.
   - Gán sẵn `requestId`, `saleTransCode` để trace.

3. **Kiểm tra `invoiceType`**
   - Chỉ cho phép `invoiceType = "CR"`.
   - Nếu request truyền giá trị khác -> trả:
     - `errorCode = "INVALID_INVOICE_TYPE"`
     - `error = "Invoice type must be CR (Credit invoice), but got: {invoiceType}"`.

4. **Rẽ nhánh theo cấu hình**
   - Nếu `useOldSystemForCreateInvoice = true` -> đi nhánh BCCS1 tại mục `4.3`.
   - Nếu `useOldSystemForCreateInvoice = false` -> đi nhánh BCCS3 tại mục `4.2`.

---

### 4.2. Luồng BCCS3 (tạo credit invoice trực tiếp trên `sale-im-service`)

1. **Kiểm tra `saleTransCode`**
   - Nếu rỗng/null -> trả lỗi `400`.

2. **Lấy giao dịch bán hàng**
   - Hệ thống tra cứu `SaleTrans` theo `saleTransCode`:

   ```sql
   SELECT *
   FROM bccs3_sale_trans.sale_trans
   WHERE sale_trans_code = :saleTransCode
   ```

   - Nếu không tìm thấy -> trả lỗi `400`.

3. **Kiểm tra giao dịch đã có hóa đơn gốc `SC`**
   - Đọc `invoice_used_id` từ `sale_trans`.
   - Nếu null hoặc `<= 0` -> trả lỗi `sale.logistics.create.credit.invoice.sc.invoice.not.found`.

4. **Đọc hóa đơn gốc đang active**
   - Hệ thống đọc `invoice_used` theo `invoiceUsedId`, `status = 1`:

   ```sql
   SELECT *
   FROM bccs3_sale_trans.invoice_used
   WHERE invoice_used_id = :invoiceUsedId
     AND status = 1
   ```

   - Nếu không có bản ghi -> trả lỗi `sale.logistics.create.credit.invoice.original.invoice.not.found`.

5. **Đảm bảo hóa đơn gốc là loại `SC`**
   - Nếu `originalInvoice.invoiceType != SC` -> trả lỗi `sale.logistics.create.credit.invoice.original.invoice.not.sc`.

6. **Kiểm tra đã tồn tại credit invoice `CR` chưa**
   - Hệ thống kiểm tra `invoice_used` active với `from_invoice_used_id = invoiceUsedId` và `invoice_type = CR`:

   ```sql
   SELECT *
   FROM bccs3_sale_trans.invoice_used
   WHERE from_invoice_used_id = :invoiceUsedId
     AND invoice_type = :crType
     AND status = 1
   ```

   - Nếu đã tồn tại -> trả lỗi `sale.logistics.create.credit.invoice.cr.invoice.already.exists`.

7. **Lấy thông tin shop**
   - Đọc `shop` theo `saleTrans.shopId`.
   - Nếu không tìm thấy -> trả lỗi `sale.logistics.create.credit.invoice.shop.not.found`.

8. **Xác định `shopUsedId` để cấp hóa đơn**
   - Hệ thống lấy `shop_used_id`:

   ```sql
   SELECT shop_used_id
   FROM shop_invoice_list
   WHERE shop_id = :shopId
     AND status = 1
   ```

   - Nếu không có `shop_used_id` thì fallback về chính `saleTrans.shopId`.

9. **Lấy một dãy hóa đơn `CR` còn số**
   - Chọn bản ghi đầu tiên còn hiệu lực trong `invoice_list`:

   ```sql
   SELECT is2.SERIAL_NO,
          it.INVOICE_TYPE,
          il.INVOICE_LIST_ID,
          il.FROM_INVOICE,
          il.TO_INVOICE,
          il.CURR_INVOICE,
          il.SHOP_ID,
          il.STATUS,
          it.INVOICE_NO_LENGTH
   FROM bccs3_sale_trans.invoice_list il,
        bccs3_sale_trans.invoice_serial is2,
        bccs3_sale_trans.invoice_type it
   WHERE il.INVOICE_SERIAL_ID = is2.INVOICE_SERIAL_ID
     AND is2.INVOICE_TYPE_ID = it.INVOICE_TYPE_ID
     AND il.STATUS = 1
     AND is2.STATUS = 1
     AND it.STATUS = 1
     AND il.CURR_INVOICE <= il.TO_INVOICE
     AND il.SHOP_ID = :shopUsedId
     AND it.INVOICE_TYPE = :type
   ORDER BY il.CURR_INVOICE
   ```

   - Nếu không có dãy hóa đơn -> trả lỗi `sale.logistics.create.credit.invoice.invoice.list.not.found`.

10. **Khóa dãy hóa đơn**
    - Khóa bản ghi `invoice_list` để tránh cấp trùng:

    ```sql
    SELECT il.INVOICE_LIST_ID,
           il.FROM_INVOICE,
           il.TO_INVOICE,
           il.CURR_INVOICE,
           il.SHOP_ID,
           il.STATUS
    FROM bccs3_sale_trans.invoice_list il
    WHERE il.STATUS = 1
      AND il.INVOICE_LIST_ID = :invoiceListId
      AND il.CURR_INVOICE <= il.TO_INVOICE
    FOR UPDATE
    ```

    - Nếu lock không thành công -> trả lỗi `sale.logistics.create.invoice.invoice.list.locked`.

11. **Sinh số hóa đơn `CR`**
    - `invoiceId = currInvoice + 1`.
    - `invoiceNo = serialNo + blockNo + zero-padding(invoiceId)`.
    - Độ dài phần số dùng `lengthInvoice` hoặc `invoiceNoLength`.

12. **Tạo bản ghi `invoice_used` mới loại `CR`**
    - Hệ thống tạo mới `InvoiceUsed` trên BCCS3 với các field chính:
      - `invoice_used_id` -> lấy sequence nội bộ qua `invoiceUsedRepo.getNextVal()`
      - `invoice_list_id` -> dãy hóa đơn vừa khóa
      - `invoice_datetime`, `create_date` -> `LocalDateTime.now()`
      - `invoice_type` -> `CR`
      - `from_invoice_used_id` -> `invoiceUsedId` của hóa đơn gốc
      - `invoice_no`, `invoice_id`, `serial_no` -> số hóa đơn mới sinh
      - `note`, `reason` -> từ request `reason`
      - `request_credit_inv = 2`
      - `approve_request_credit_inv_user = "LOGISTICS_API"`
      - `approve_request_credit_inv_date = sysDate`
      - `last_update_user = "LOGISTICS_API"`
      - Các thông tin khách hàng, tiền, thuế, shop... chủ yếu copy từ hóa đơn gốc / giao dịch

13. **Lưu credit invoice**
    - Ghi `invoice_used` mới vào `bccs3_sale_trans.invoice_used`.

14. **Cập nhật dãy hóa đơn**
    - Code hiện tại tăng `lockedInvoiceList.currInvoice` lên `+1` rồi `save`.
    - Về nghiệp vụ tương đương:

    ```sql
    UPDATE bccs3_sale_trans.invoice_list
    SET curr_invoice = :newCurrInvoice
    WHERE invoice_list_id = :invoiceListId
    ```

15. **Cập nhật hóa đơn gốc**
    - Hệ thống gọi `approveCreditInvoiceTemp(invoiceUsedId, "LOGISTICS_API")`:

    ```sql
    UPDATE bccs3_sale_trans.invoice_used
    SET request_credit_inv = 2,
        status = -1,
        approve_request_credit_inv_date = :sysdate,
        approve_request_credit_inv_user = :currentUser,
        last_update_user = :lastUpdateUser
    WHERE invoice_used_id = :invoiceUsedId
    ```

16. **Trả kết quả thành công**
    - Response DTO:

    ```json
    {
      "requestId": "REQ_20260307_0001",
      "saleTransCode": "ST202603070001",
      "requestStatus": "Success"
    }
    ```

    - **Lưu ý code hiện tại:** nhánh này chưa set `errorCode = "200"`, trong khi controller lại dùng `errorCode` để xác định response success.

---

### 4.3. Luồng BCCS1 (gọi hệ thống cũ)

Khi `useOldSystemForCreateInvoice = true`, `sale-im-service` không tự tạo hóa đơn trên BCCS3 mà gọi sang BCCS1 qua Feign `Bccs1InventoryService`.

1. **Gọi API BCCS1**
   - Feign gọi:

   ```text
   POST /synchronized/create-credit-invoice-logistics
   ```

   - Tham số gửi sang:
     - `requestId`
     - `saleTransCode`
     - `reason`

2. **Nhận kết quả từ BCCS1**
   - Nếu `createInvoiceResult.success = false`:
     - lấy `code`, `description` từ BCCS1
     - trả ngược vào `errorCode`, `error`

3. **Đọc lại `SaleTrans` trên BCCS3**
   - Sau khi BCCS1 tạo thành công, `sale-im-service` đọc lại:

   ```sql
   SELECT *
   FROM bccs3_sale_trans.sale_trans
   WHERE sale_trans_code = :saleTransCode
   ```

   - Nếu không tìm thấy `SaleTrans` hoặc `invoiceUsedId` vẫn null -> trả lỗi `sale.logistics.create.invoice.invoice.created.but.trans.not.found`.

4. **Trả thành công**
   - Nếu đồng bộ thành công:
     - `requestStatus = "Success"`
     - `errorCode = "200"`

> **Lưu ý:** Nhánh BCCS1 hiện không tự in PDF trong API này; nó chỉ tạo hóa đơn ở hệ thống cũ rồi xác nhận lại trạng thái tại BCCS3.

---

## 5. Các lỗi nghiệp vụ chính

- `INVALID_INVOICE_TYPE`
  - Khi `invoiceType` khác `CR`.
- `sale.logistics.create.credit.invoice.sc.invoice.not.found`
  - Giao dịch chưa có hóa đơn gốc `SC`.
- `sale.logistics.create.credit.invoice.original.invoice.not.found`
  - Không tìm thấy hóa đơn gốc active.
- `sale.logistics.create.credit.invoice.original.invoice.not.sc`
  - Hóa đơn gốc không phải loại `SC`.
- `sale.logistics.create.credit.invoice.cr.invoice.already.exists`
  - Đã tồn tại hóa đơn `CR` active cho hóa đơn gốc.
- `sale.logistics.create.credit.invoice.shop.not.found`
  - Không tìm thấy shop của giao dịch.
- `sale.logistics.create.credit.invoice.invoice.list.not.found`
  - Không tìm thấy dãy hóa đơn `CR`.
- `sale.logistics.create.invoice.invoice.list.locked`
  - Dãy hóa đơn đang bị process khác sử dụng.
- `sale.logistics.create.invoice.invoice.created.but.trans.not.found`
  - BCCS1 tạo xong nhưng BCCS3 không đọc lại được `SaleTrans`/`invoiceUsedId`.

---

## 6. Ghi chú tích hợp

- API public nằm ở `com.viettel.bccs3.sale.controller.LogisticsController#createCreditInvoice`.
- Business logic nằm ở `com.viettel.bccs3.sale.service.impl.LogisticsServiceImpl#createCreditInvoice`.
- DTO request/response:
  - `com.viettel.bccs3.sale.common.dto.CreateCreditInvoiceRequestDTO`
  - `com.viettel.bccs3.sale.common.dto.CreateCreditInvoiceResponseDTO`
- Nếu chạy nhánh BCCS1, `sale-im-service` gọi:
  - `com.viettel.bccs3.sale.service.feign.Bccs1InventoryService#createCreditInvoiceLogistics`
- Endpoint phía BCCS1 thực tế là:
  - `POST /synchronized/create-credit-invoice-logistics`
- Tài liệu này mô tả **API public mà phía Logistics gọi vào BCCS3**; còn endpoint BCCS1 chỉ là endpoint nội bộ để nhánh cấu hình hệ thống cũ sử dụng.
