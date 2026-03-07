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

### 3.1. Request Body

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

Response thành công:

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

- Đây là format chắc chắn đúng với **nhánh BCCS1**, vì nhánh này set `errorCode = "200"`.
- Với **nhánh BCCS3**, hiện trả `requestStatus = "Success"` nhưng **không set `errorCode = "200"`**.
- Do controller đang check `errorCode` để quyết định success/fail, nên nếu bám đúng code hiện tại thì nhánh BCCS3 có rủi ro bị controller coi là fail dù nghiệp vụ đã tạo invoice thành công.

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

> **Lưu ý:** Lỗi nghiệp vụ vẫn đi ra với HTTP status `200`, còn mã lỗi nằm trong body.

---

## 4. Luồng tạo Credit Invoice trong hệ thống

### 4.1. Luồng tổng quát tại `sale-im-service`

1. **Nhận request từ Logistics**
   - Hệ thống nhận request body chứa `requestId`, `saleTransCode`, `invoiceType`, `reason`.

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
   - Kiểm tra `invoice_type` của hóa đơn gốc phải bằng giá trị tương ứng với `SC`.
   - Nếu không phải `SC` -> trả lỗi `sale.logistics.create.credit.invoice.original.invoice.not.sc`.

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
   - Đọc thông tin shop theo `shop_id` từ giao dịch bán hàng:

   ```sql
   SELECT *
   FROM shop
   WHERE shop_id = :shopId
   ```

   - Nếu không tìm thấy -> trả lỗi `sale.logistics.create.credit.invoice.shop.not.found`.

8. **Xác định `shopUsedId` để cấp hóa đơn**
   - Hệ thống lấy `shop_used_id`:

   ```sql
   SELECT shop_used_id
   FROM shop_invoice_list
   WHERE shop_id = :shopId
     AND status = 1
   ```

   - Nếu không có `shop_used_id` thì fallback về chính `shop_id` của giao dịch.

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
    - Tính `invoiceId = currInvoice + 1` (số thứ tự trong dãy).
    - Ghép `invoiceNo` theo công thức:
      - **Phần 1 - Serial:** `serial_no` từ dãy hóa đơn (nếu null thì để rỗng).
      - **Phần 2 - Block:** 
        - Nếu `block_no` null hoặc `length_name` null → để rỗng.
        - Nếu có: padding `block_no` về độ dài `length_name` bằng số `0` phía trước.
        - Ví dụ: `block_no = "12"`, `length_name = 4` → `"0012"`.
      - **Phần 3 - Invoice ID:**
        - Lấy `length_invoice` từ bảng `invoice_type` (nếu null thì dùng `invoice_no_length`).
        - Padding `invoice_id` về độ dài `length_invoice` bằng số `0` phía trước.
        - Ví dụ: `invoice_id = 123`, `length_invoice = 6` → `"000123"`.
    - Kết quả: `invoiceNo = serialPart + blockPart + invoicePart`.
    - Ví dụ: `serialNo = "CR"`, `blockNo = "001"` (đã padding), `invoiceId = "000123"` → `invoiceNo = "CR001000123"`.

12. **Tạo bản ghi `invoice_used` mới loại `CR`**
    - Hệ thống tạo mới bản ghi `invoice_used` trên BCCS3 với mapping đầy đủ các field như sau:
    
    **Thông tin định danh hóa đơn:**
    - `invoice_used_id` → lấy từ sequence nội bộ
    - `invoice_list_id` → ID dãy hóa đơn vừa khóa
    - `invoice_datetime` → thời điểm hiện tại
    - `create_date` → thời điểm hiện tại
    - `invoice_type` → giá trị số tương ứng với `CR`
    - `from_invoice_used_id` → ID hóa đơn gốc `SC`
    - `invoice_no` → số hóa đơn vừa sinh ở bước 11
    - `invoice_id` → số thứ tự trong dãy
    - `serial_no` → serial number từ dãy hóa đơn
    
    **Thông tin shop và nhân viên:**
    - `shop_id` → shop ID từ giao dịch bán hàng
    - `staff_id` → staff ID từ giao dịch bán hàng
    - `receiver_id` → receiver ID từ giao dịch bán hàng
    - `receiver_type` → receiver type từ giao dịch bán hàng
    - `shop_name` → tên shop
    - `shop_address` → địa chỉ shop
    - `shop_tin` → mã số thuế shop (ưu tiên từ bảng shop, nếu rỗng thì lấy từ giao dịch)
    
    **Thông tin khách hàng (copy từ hóa đơn gốc):**
    - `sub_id` → subscriber ID từ giao dịch
    - `cust_id` → customer ID từ giao dịch
    - `cust_isdn` → số điện thoại từ giao dịch
    - `cust_name` → tên khách hàng từ hóa đơn gốc (trim, nếu null thì rỗng)
    - `cust_name_kh` → tên khách hàng tiếng Khmer từ hóa đơn gốc (trim, nếu null thì rỗng)
    - `cust_address` → địa chỉ khách hàng từ hóa đơn gốc (trim, nếu null thì rỗng)
    - `cust_address_kh` → địa chỉ khách hàng tiếng Khmer từ hóa đơn gốc (trim, nếu null thì rỗng)
    - `cust_company` → tên công ty từ hóa đơn gốc (trim, nếu null thì rỗng)
    - `cust_company_kh` → tên công ty tiếng Khmer từ hóa đơn gốc (trim, nếu null thì rỗng)
    - `cust_identity_no` → số CMND/CCCD từ hóa đơn gốc
    - `cust_email` → email khách hàng từ hóa đơn gốc
    
    **Thông tin tài chính (copy từ hóa đơn gốc):**
    - `amount_not_tax` → tổng tiền trước thuế từ hóa đơn gốc
    - `tax` → thuế từ hóa đơn gốc
    - `discount` → chiết khấu từ hóa đơn gốc
    - `vat` → VAT từ hóa đơn gốc (nếu null thì mặc định `10.0`)
    - `amount_tax` → tổng tiền sau thuế từ hóa đơn gốc
    - `currency` → loại tiền tệ từ hóa đơn gốc (nếu null thì mặc định `"USD"`)
    - `pay_method` → phương thức thanh toán từ hóa đơn gốc (nếu null thì mặc định `"1"`)
    
    **Thông tin dịch vụ:**
    - `telecom_service_id` → ID dịch vụ viễn thông từ giao dịch
    
    **Thông tin điều chỉnh hóa đơn:**
    - `note` → lý do điều chỉnh từ request (field `reason`)
    - `reason` → lý do điều chỉnh từ request (field `reason`)
    - `request_credit_inv` → `2` (trạng thái đã approve)
    - `approve_request_credit_inv_user` → `"LOGISTICS_API"`
    - `approve_request_credit_inv_date` → thời điểm hiện tại
    - `last_update_user` → `"LOGISTICS_API"`
    - `status` → `1` (active)
    
    - **SQL INSERT tương đương:**
    
    ```sql
    INSERT INTO bccs3_sale_trans.invoice_used (
        invoice_used_id, invoice_list_id, invoice_datetime, create_date,
        invoice_type, from_invoice_used_id, invoice_no, invoice_id, serial_no,
        shop_id, staff_id, receiver_id, receiver_type,
        sub_id, cust_id, cust_isdn,
        cust_name, cust_name_kh, cust_address, cust_address_kh,
        cust_company, cust_company_kh, cust_identity_no, cust_email,
        pay_method, note, reason, status,
        telecom_service_id, currency,
        amount_not_tax, discount, tax, vat, amount_tax,
        shop_name, shop_address, shop_tin,
        request_credit_inv, approve_request_credit_inv_user, 
        approve_request_credit_inv_date, last_update_user
    ) VALUES (
        :invoiceUsedId, :invoiceListId, :sysDate, :sysDate,
        :crType, :fromInvoiceUsedId, :invoiceNo, :invoiceId, :serialNo,
        :shopId, :staffId, :receiverId, :receiverType,
        :subId, :custId, :custIsdn,
        :custName, :custNameKh, :custAddress, :custAddressKh,
        :custCompany, :custCompanyKh, :custIdentityNo, :custEmail,
        :payMethod, :note, :reason, 1,
        :telecomServiceId, :currency,
        :amountNotTax, :discount, :tax, :vat, :amountTax,
        :shopName, :shopAddress, :shopTin,
        2, 'LOGISTICS_API', :sysDate, 'LOGISTICS_API'
    )
    ```

13. **Lưu credit invoice**
    - Ghi `invoice_used` mới vào `bccs3_sale_trans.invoice_used`.

14. **Cập nhật dãy hóa đơn**
    - Hệ thống tăng `curr_invoice` lên `+1`:

    ```sql
    UPDATE bccs3_sale_trans.invoice_list
    SET curr_invoice = :newCurrInvoice
    WHERE invoice_list_id = :invoiceListId
    ```

15. **Cập nhật hóa đơn gốc**
    - Hệ thống cập nhật hóa đơn gốc để đánh dấu đã approve điều chỉnh:

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

    - **Lưu ý:** nhánh này chưa set `errorCode = "200"`, trong khi controller lại dùng `errorCode` để xác định response success.

---

### 4.3. Luồng BCCS1 (gọi hệ thống cũ)

Khi cấu hình `useOldSystemForCreateInvoice = true`, `sale-im-service` không tự tạo hóa đơn trên BCCS3 mà gọi sang `bccs1-inventory-service` qua HTTP client.

1. **Gọi API BCCS1**
   - Hệ thống gọi endpoint:

   ```text
   POST /synchronized/create-credit-invoice-logistics
   ```

   - Tham số gửi sang (dạng query parameter):
     - `requestId` (String)
     - `saleTransCode` (String)
     - `reason` (String, có thể null)

2. **Nhận kết quả từ BCCS1**
   - Response từ BCCS1 chứa các thông tin:
     - `code`: mã trả về (`"200"` nếu thành công, `"400"` hoặc `"500"` nếu lỗi)
     - `success`: trạng thái thành công (`true` nếu thành công, `false` nếu lỗi)
     - `description`: mô tả lỗi hoặc thông báo thành công
     - `invoiceUsedDTO`: chứa `invoiceUsedId`, `invoiceNo`, `saleTransId` khi thành công
   
   - Nếu BCCS1 trả về lỗi (`success = false`):
     - Lấy `code` và `description` từ response BCCS1
     - Gán vào `errorCode` và `error` của response trả về cho Logistics
     - Trả về response với `requestStatus = "Fail"`

3. **Đọc lại giao dịch bán hàng trên BCCS3 để xác nhận đồng bộ**
   - Sau khi BCCS1 tạo thành công, hệ thống đọc lại giao dịch bán hàng:

   ```sql
   SELECT *
   FROM bccs3_sale_trans.sale_trans
   WHERE sale_trans_code = :saleTransCode
   ```

   - Nếu không tìm thấy giao dịch hoặc `invoice_used_id` vẫn null → trả lỗi:
     - `errorCode = "1"`
     - `error = "sale.logistics.create.invoice.invoice.created.but.trans.not.found"`

4. **Trả thành công**
   - Nếu đồng bộ thành công:
     - `requestStatus = "Success"`
     - `errorCode = "200"`

> **Lưu ý:** 
> - Nhánh BCCS1 hiện không tự in PDF trong API này; nó chỉ tạo hóa đơn ở hệ thống cũ rồi xác nhận lại trạng thái tại BCCS3.
> - BCCS1 sẽ tự xử lý việc lấy dãy hóa đơn `CR`, khóa dãy, tạo `invoice_used` trên schema `bccs_im`, cập nhật hóa đơn gốc, và cập nhật dãy hóa đơn.

#### 4.3.2. Luồng xử lý bên BCCS1 (bccs1-inventory-service)

Phần này mô tả **nghiệp vụ** khi hệ thống BCCS1 nhận yêu cầu tạo credit invoice logistics với tham số `requestId`, `saleTransCode`, `reason`.

1. **Kiểm tra tham số đầu vào**
   - Nếu `saleTransCode` rỗng/null → trả mã lỗi `"400"`, mô tả theo message `parameter.not.null`.

2. **Lấy giao dịch bán hàng theo `saleTransCode`**
   - Hệ thống tra cứu giao dịch bán hàng:

     ```sql
     SELECT *
     FROM bccs_im.sale_trans
     WHERE sale_trans_code = :saleTransCode
     ```

   - Nếu không tìm thấy giao dịch → trả:
     - `code = "400"`
     - `description = "Error: Sale trans not found for saleTransCode: {saleTransCode}"`

3. **Kiểm tra đã tồn tại credit invoice cho hóa đơn gốc**
   - Lấy `invoice_used_id` từ giao dịch.
   - Nếu có `invoice_used_id`, đọc hóa đơn gốc và kiểm tra xem đã có credit invoice chưa:

     ```sql
     SELECT *
     FROM bccs_im.invoice_used
     WHERE original_invoice = :originalInvoiceNo
       AND type = :creditType
       AND status = :status
     ```

   - Nếu đã tồn tại → trả:
     - `code = "400"`
     - `description = "Error: CR invoice already exists for saleTransCode: {saleTransCode}"`

4. **Kiểm tra giao dịch đã có hóa đơn gốc `SC` chưa**
   - Nếu `invoice_used_id` null hoặc `<= 0` → trả:
     - `code = "400"`
     - `description = "Error: SC invoice does not exist for saleTransCode: {saleTransCode}"`

5. **Đọc hóa đơn gốc `SC` và kiểm tra còn hiệu lực**
   - Tra cứu hóa đơn gốc theo `invoice_used_id`.
   - Nếu không tìm thấy hoặc `status` của hóa đơn gốc khác trạng thái active → trả:
     - `code = "400"`
     - `description = "Error: Original SC invoice not found or inactive for saleTransCode: {saleTransCode}"`

6. **Lấy thông tin shop của giao dịch**
   - Dùng `shop_id` trong giao dịch để lấy thông tin shop.
   - Nếu không tìm thấy → trả:
     - `code = "400"`
     - `description = "Error: Shop not found for shopId: {shopId}"`

7. **Lấy danh sách serial hóa đơn điều chỉnh được cấu hình**
   - Hệ thống đọc cấu hình `app_params` để xác định serial nào được phép dùng cho credit invoice:

     ```sql
     SELECT *
     FROM bccs_im.app_params
     WHERE type = 'GET_INVOICE_USED'
       AND code = 'CODE'
     ```

   - Sau đó chọn các `serial_no` từ `invoice_list` của shop cấu hình credit invoice (shop ID cố định), chỉ lấy những dãy còn hiệu lực và còn số:

     ```sql
     SELECT inv.serial_no AS serialNo
     FROM bccs_im.invoice_list inv
            JOIN bccs_im.book_type bt
              ON inv.book_type_id = bt.book_type_id
     WHERE inv.shop_id = :shopId
       AND inv.status = :status
       AND inv.curr_invoice_no >= inv.from_invoice
       AND inv.curr_invoice_no <= inv.to_invoice
       AND inv.invoice_type = :invoiceType
       AND bt.serial_real IN (
              SELECT value
              FROM bccs_im.app_params
              WHERE type = 'CREDIT_INVOICE_USED'
                AND status = 1
            )
     ORDER BY serialNo
     ```

   - Nếu không có serial khả dụng → trả:
     - `code = "400"`
     - `description = "SaleToRetailDAO.013"`

   > **Lưu ý:** Điều kiện `bt.serial_real IN (...)` chỉ được áp dụng khi cấu hình `GET_INVOICE_USED/CODE` có giá trị `"1"`.

8. **Lấy và khóa 1 dãy hóa đơn `CR` còn số**
   - Hệ thống lấy serial đầu tiên ở bước 7, sau đó tìm 1 dòng `invoice_list` loại hóa đơn điều chỉnh (`CR`) còn số và khóa luôn bản ghi bằng `FOR UPDATE NOWAIT` để tránh cấp trùng:

     ```sql
     SELECT il.invoice_list_id AS invoiceListId,
            il.serial_no       AS serialNo,
            il.block_no        AS blockNo,
            il.from_invoice    AS fromInvoice,
            il.to_invoice      AS toInvoice,
            il.curr_invoice_no AS currInvoiceNo,
            ( il.serial_no
              || LPAD(
                   il.block_no,
                   (SELECT bt.length_name
                    FROM   bccs_im.book_type bt
                    WHERE  bt.book_type_id = il.book_type_id),
                   '0'
                 )
              || LPAD(
                   il.curr_invoice_no,
                   (SELECT bt.length_invoice
                    FROM   bccs_im.book_type bt
                    WHERE  bt.book_type_id = il.book_type_id),
                   '0'
                 )
            ) AS invoiceNumber
     FROM   bccs_im.invoice_list il
     WHERE  il.curr_invoice_no > 0
       AND  il.curr_invoice_no BETWEEN il.from_invoice AND il.to_invoice
       AND  il.invoice_type = :invoiceType
       AND  il.status = :status
       AND  il.shop_id = :shopId
       AND  il.serial_no = :serialNo
     ORDER BY serialNo, currInvoiceNo
     FOR UPDATE NOWAIT
     ```

   - Nếu không lấy được dãy hóa đơn → trả:
     - `code = "400"`
     - `description = "SaleToRetailDAO.010"`

9. **Đảm bảo số hiện tại của dãy chưa được sử dụng**
   - Sau khi lấy được `invoice_list`, hệ thống kiểm tra số hiện tại trong dãy đã có trong `invoice_used` chưa:

     ```sql
     SELECT *
     FROM bccs_im.invoice_used
     WHERE invoice_list_id = :invoiceListId
       AND invoice_id      = :invoiceId
     ```

   - Nếu đã có bản ghi → trả:
     - `code = "400"`
     - `description = "SaleToRetailDAO.010"`

10. **Lấy khóa chính mới cho credit invoice**
    - Hệ thống lấy `invoice_used_id` mới từ sequence:

      ```sql
      SELECT bccs_im.invoice_used_seq.NEXTVAL AS invoiceUsedId
      FROM   dual
      ```

11. **Tạo bản ghi credit invoice trong `INVOICE_USED`**
    - Hệ thống tạo mới bản ghi `invoice_used` bằng cách:
      - Copy gần như toàn bộ dữ liệu từ hóa đơn gốc `SC`.
      - Ghi đè các thông tin dành cho hóa đơn điều chỉnh.

    - **Mapping chính của credit invoice:**
      - `invoice_used_id` → sequence `INVOICE_USED_SEQ`
      - `invoice_list_id`, `serial_no`, `block_no`, `invoice_id`, `invoice_no` → từ dãy hóa đơn lấy ở bước 8
      - `type` → giá trị số tương ứng với `CR`
      - `request_credit_inv` → trạng thái `APPROVE`
      - `approve_credit_inv_date` → thời điểm hiện tại
      - `approve_credit_inv_user` → `"LOGISTICS_API"`
      - `is_credit_inv` → `"1"`
      - `original_invoice` → `invoice_no` của hóa đơn gốc `SC`
      - `create_date` → thời điểm hiện tại
      - `note` → gán từ tham số `reason` nếu request có truyền
      - `print_type1`, `print_type2`, `request_print` → reset về `0`
      - Các thông tin khách hàng, số tiền, thuế, shop, dịch vụ... → giữ nguyên theo hóa đơn gốc

12. **Cập nhật lại hóa đơn gốc `SC`**
    - Sau khi tạo credit invoice thành công, hóa đơn gốc cũng được cập nhật trạng thái yêu cầu điều chỉnh:
      - `request_credit_inv` → `APPROVE`
      - `approve_credit_inv_date` → thời điểm hiện tại
      - `approve_credit_inv_user` → `"LOGISTICS_API"`

13. **Cập nhật dãy hóa đơn sau khi đã dùng 1 số**
    - Hệ thống tăng `curr_invoice_no` của `invoice_list`.
    - Nếu đã dùng đến cuối dãy thì chuyển trạng thái dãy sang đã dùng hết.

    ```sql
    -- Nếu còn số trong dãy
    UPDATE bccs_im.invoice_list
    SET    curr_invoice_no = curr_invoice_no + 1
    WHERE  invoice_list_id = :invoiceListId

    -- Nếu đã dùng hết dãy
    UPDATE bccs_im.invoice_list
    SET    curr_invoice_no = 0,
           status = :statusUsingCompleted
    WHERE  invoice_list_id = :invoiceListId
    ```

    - Nếu cập nhật dãy thất bại → phát sinh lỗi:
      - `description = "SaleToCreateInvoiceDAO.error.010"` hoặc message cấu hình tương ứng

14. **Trả kết quả thành công cho phía gọi**
    - Nếu toàn bộ các bước trên thành công, API trả:
      - `code = "200"`
      - `success = true`
      - `description = "Success: Credit invoice created for saleTransCode: {saleTransCode}"`
      - Thông tin hóa đơn: ID credit invoice mới, số hóa đơn `CR` vừa tạo, ID giao dịch bán hàng

15. **Xử lý exception**
    - `IllegalArgumentException` → `code = "400"`, `success = false`, `description = ex.getMessage()`.
    - Exception khác → `code = "500"`, `success = false`, `description = "ERR.SAE.118"`.

> **Lưu ý:** 
> - Dãy hóa đơn `CR` không lấy theo `shop_id` của giao dịch bán hàng mà đang lấy theo shop cấu hình riêng (shop ID cố định).
> - BCCS1 tạo `invoice_used` trên schema `bccs_im`, khác với BCCS3 tạo trên schema `bccs3_sale_trans`.

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

- API public nằm ở service `sale-im-service`, endpoint `/sale/logistics/create-credit-invoice`.
- Nếu chạy nhánh BCCS1, `sale-im-service` sẽ gọi sang `bccs1-inventory-service` qua endpoint:
  - `POST /synchronized/create-credit-invoice-logistics`
- Tài liệu này mô tả **API public mà phía Logistics gọi vào BCCS3**; còn endpoint BCCS1 chỉ là endpoint nội bộ để nhánh cấu hình hệ thống cũ sử dụng.
