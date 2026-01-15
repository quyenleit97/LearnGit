## API CREATE INVOICE (LOGISTICS & NORMAL FLOW)

Tài liệu mô tả chi tiết nghiệp vụ tạo hóa đơn (Invoice) cho sale transaction, bao gồm:
- **Nghiệp vụ tổng thể**
- **SQL chính để lấy/lock Invoice List, tạo InvoiceUsed**
- **Logic chọn Excel Template (kể cả luồng Logistics với State_Burden)**

---

## 1. Tổng quan nghiệp vụ

- Hệ thống tạo hóa đơn dựa trên `InvoiceUsed` và `InvoiceList`.
- Đối với Logistics:
  - API `/sale/logistics/create-invoice` nhận `saleTransCode`, `invoiceType` (SC/CR).
  - Nếu `SaleTrans` đã có `invoiceUsedId` thì chỉ generate PDF.
  - Nếu chưa có hóa đơn:
    - Kiểm tra trạng thái giao dịch (`NOT_BILLED` = 2).
    - Lấy và lock một bản ghi `InvoiceList` khả dụng cho shop.
    - Tạo mới `InvoiceUsed`, cập nhật `SaleTrans`, cập nhật `InvoiceList`.
    - In hóa đơn bằng Excel template tương ứng.

### LUỒNG XỬ LÝ ĐANG ÁP DỤNG

- Trước mắt, việc tạo invoice được xử lý trên **con 1** (BCCS1).
- Sau khi tạo xong, dữ liệu invoice được đồng bộ sang **con 3** (BCCS3).
- Nguyên nhân: **con 3 chưa khai dải hóa đơn** nên chưa thể tự tạo invoice độc lập.

---

## 2. Endpoint

### 2.1. Endpoint

```text
POST /sale/logistics/create-invoice
```

### 2.2. Ví dụ cURL

```bash
curl --location --request POST 'http://100.99.1.165:32015/sale/logistics/create-invoice' \
--header 'Accept-Language: vi-la' \
--header 'Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...token...' \
--header 'Content-Type: application/json' \
--header 'Cookie: JSESSIONID=8E259F542AD91E085D877EE7B03B53B2' \
--data-raw '{
    "requestId": "11111111111111",
    "saleTransCode": "SS000001000136109",
    "invoiceType": "SC"
}'
```

---

## 3. Request / Response

### 3.1. Request Body

```json
{
  "requestId": "string",
  "saleTransCode": "string",
  "invoiceType": "string"
}
```

| Field          | Type   | Required | Ghi chú                                       |
|---------------|--------|----------|-----------------------------------------------|
| `requestId`   | String | Yes      | ID request để trace log/response             |
| `saleTransCode` | String | Yes    | Mã giao dịch bán hàng                        |
| `invoiceType` | String | Yes      | `SC` (Standard Credit) hoặc `CR` (Credit)    |

### 3.2. Response (GetInvoiceResponseDTO)

```json
{
    "requestId": "11111111111111",
    "status": "Success",
  "invoiceFile": "base64-pdf",
    "annexFile": null,
    "errorCode": null,
    "errorDescription": null
}
```

---

## 4. Luồng tạo Invoice trong hệ thống

### 4.1. Luồng BCCS1 (gọi sang hệ thống cũ qua API `/synchronized/create-invoice-logistics`)

Khi cấu hình sử dụng **hệ thống cũ BCCS1** cho việc tạo hóa đơn, `sale-im-service` sẽ gọi sang service `bccs1-inventory-service` qua API HTTP.

- **Endpoint phía BCCS1**  
  - `POST /synchronized/create-invoice-logistics` với tham số `saleTransCode`.

Phần này mô tả **nghiệp vụ** khi hệ thống BCCS1 nhận yêu cầu tạo hóa đơn logistics với tham số `saleTransCode`.

1. **Kiểm tra tham số đầu vào**
   - Nếu `saleTransCode` rỗng → trả `code = "400"`, mô tả theo cấu hình message `parameter.not.null`.

2. **Lấy giao dịch bán hàng cần tạo hóa đơn**
   - Thực hiện query trên bảng giao dịch bán hàng:
   
   ```sql
   SELECT *
   FROM bccs_im.sale_trans
   WHERE sale_trans_code = :saleTransCode
   ```
   
   - Nếu không có bản ghi → trả `code = "400"`, `description = "sale.trans.not.found"`.

3. **Kiểm tra giao dịch đã có hóa đơn chưa**
   - Nếu cột `invoice_used_id` của giao dịch **khác NULL** → coi như đã có hóa đơn, trả:
     - `code = "400"`  
     - `description = "Sale trans already has invoice"`.

4. **Kiểm tra trạng thái giao dịch đủ điều kiện xuất hóa đơn**
   - Chỉ cho phép tạo hóa đơn khi cột `status` của giao dịch = `"2"` (NOT_BILLED).  
   - Nếu khác `"2"` → trả:
     - `code = "400"`  
     - `description = "Sale trans status is not valid for creating invoice"`.

5. **Lấy dãy hóa đơn (InvoiceList) còn số cho shop**
   - Dùng **shop của giao dịch** và **loại hóa đơn bán hàng** để chọn một dòng `invoice_list` còn số, trạng thái đang hoạt động:
     - `shop_id` = shop thực hiện giao dịch (`shop_id` trong `sale_trans`).
     - `invoice_type` = `1` (mã loại hóa đơn bán hàng/ACTIVE).
     - `status` = `1` (mã trạng thái "đang phát hành tại cửa hàng"/AVAILABLE_IN_SHOP).
   - Nếu không tìm thấy → coi như **hết số hóa đơn**, trả:
     - `code = "400"`  
     - `description = "SaleToRetailDAO.010"`.
   - **SQL chọn dãy hóa đơn (getCurrentInvoiceList):**
   
   ```sql
   SELECT il.invoice_list_id AS invoiceListId,
          il.serial_no       AS serialNo,
          il.block_no        AS blockNo,
          il.from_invoice    AS fromInvoice,
          il.to_invoice      AS toInvoice,
          il.curr_invoice_no AS currInvoiceNo,
          ( il.serial_no
            || LPAD(il.block_no,
                    (SELECT bt.length_name
                       FROM bccs_im.book_type bt
                      WHERE bt.book_type_id = il.book_type_id),
                    '0')
            || LPAD(il.curr_invoice_no,
                    (SELECT bt.length_invoice
                       FROM bccs_im.book_type bt
                      WHERE bt.book_type_id = il.book_type_id),
                    '0')
          ) AS invoiceNumber
   FROM   bccs_im.invoice_list il
   WHERE  il.invoice_list_id = (
       SELECT invoice_list_id
       FROM   (
           SELECT il2.invoice_list_id
           FROM   bccs_im.invoice_list il2
           WHERE  il2.curr_invoice_no > 0
             AND  il2.curr_invoice_no >= il2.from_invoice
             AND  il2.curr_invoice_no <= il2.to_invoice
             AND  il2.invoice_type = :invoiceType
             AND  il2.shop_id      = :shopId
             AND  il2.status       = :status
           ORDER BY il2.serial_no, il2.curr_invoice_no
       )
       WHERE ROWNUM = 1
   )
   FOR UPDATE
   ```

6. **Khóa bản ghi InvoiceList để tránh trùng số (NOWAIT)**
   - Dựa vào `invoice_list_id` vừa tìm được, thực hiện câu lệnh khóa bản ghi:
   
   ```sql
   SELECT *
   FROM bccs_im.invoice_list
   WHERE invoice_list_id = :invoiceListId
   FOR UPDATE NOWAIT
   ```
   
   - Nếu khóa thất bại (có process khác đang giữ khóa) → trả:
     - `code = "400"`  
     - `description = "Error. Invoice list is being used. Please wait!"`.

7. **Đảm bảo số hóa đơn hiện tại chưa được dùng**
   - Kiểm tra trong bảng `INVOICE_USED` xem đã có bản ghi nào dùng cặp (`invoice_list_id`, `curr_invoice_no`) hay chưa:
   
   ```sql
   SELECT 1
   FROM bccs_im.invoice_used
   WHERE invoice_list_id = :invoiceListId
     AND invoice_id      = :currInvoiceNo
   ```
   
   - Nếu đã tồn tại bản ghi → coi như dãy hóa đơn này không còn dùng được, trả:
     - `code = "400"`  
     - `description = "SaleToRetailDAO.010"`.

8. **Lấy khóa chính cho hóa đơn mới (`invoice_used_id`)**
   - Lấy giá trị từ sequence `invoice_used_seq` trong schema `bccs_im`:
   
   ```sql
   SELECT bccs_im.invoice_used_seq.NEXTVAL AS invoiceUsedId FROM dual
   ```

9. **Tạo bản ghi hóa đơn mới trong `INVOICE_USED`**
   - Dùng thông tin dãy hóa đơn (InvoiceList) + thông tin giao dịch để chèn 1 bản ghi vào bảng `bccs_im.invoice_used`:
   
   ```sql
   INSERT INTO bccs_im.invoice_used (
     invoice_used_id, invoice_list_id, type,
     shop_id, staff_id,
     serial_no, block_no, invoice_id, invoice_no,
     cust_name, address, company, currency,
     cust_name_kh, company_kh, isdn, tin,
     pay_method, note,
     amount, tax, amount_tax, discount, promotion, vat,
     status, create_date, invoice_date,
     telecom_service_id, reason_id, exchange_rate, sub_id
   ) VALUES (
     :invoiceUsedId, :invoiceListId, :invoiceType,
     :shopId, :staffId,
     :serialNo, :blockNo, :invoiceId, :invoiceNo,
     :custName, :address, :company, :currency,
     :custNameKh, :companyKh, :isdn, :tin,
     :payMethod, :note,
     :amount, :tax, :amountTax, :discount, :promotion, :vat,
     :status, :createDate, :invoiceDate,
     :telecomServiceId, :reasonId, :exchangeRate, :subId
   )
   ```
   
   - **Mapping chính InvoiceUsed từ dữ liệu:**
     - `invoice_used_id` → sequence `invoice_used_seq`
     - `invoice_list_id`, `type`, `serial_no`, `block_no`, `invoice_id`, `invoice_no` → từ `invoiceListBean` (kết quả `getCurrentInvoiceList`)
     - `shop_id`, `staff_id`, `cust_name`, `cust_name_kh`, `address`, `company`, `company_kh`, `isdn`, `tin`, `pay_method`, `note` → từ `SaleTrans` tương ứng (trim, default `""` hoặc `"1"` nếu null)
     - `amount`, `tax`, `amount_tax`, `discount`, `promotion`, `vat`, `currency` → từ `SaleTrans.amountNotTax`, `tax`, `amountTax`, `discount`, `promotion`, `vat`, `currency` (default `"USD"` nếu null)
     - `status` → `Const.INVOICE_USED.STATUS_USE`
     - `create_date`, `invoice_date` → `LocalDateTime.now()` / `saleTrans.saleTransDate`
     - `telecom_service_id`, `reason_id`, `exchange_rate`, `sub_id` → các field tương ứng của `SaleTrans`.

10. **Gắn `invoice_used_id` vào giao dịch và đổi trạng thái sang đã xuất hóa đơn**
    - Cập nhật lại bản ghi giao dịch để biết giao dịch này đã có hóa đơn:
    
    ```sql
    UPDATE bccs_im.sale_trans
    SET invoice_used_id = :invoiceUsedId,
        status          = :statusBilled,
        invoice_create_date = SYSDATE
    WHERE sale_trans_id = :saleTransId
    ```

11. **Cập nhật lại dãy hóa đơn (InvoiceList) sau khi đã dùng 1 số**
    - Nếu số hiện tại (`curr_invoice_no`) còn nhỏ hơn số cuối (`to_invoice`) → chỉ cần tăng số hiện tại thêm 1:
    
    ```sql
    UPDATE bccs_im.invoice_list
    SET curr_invoice_no = curr_invoice_no + 1
    WHERE invoice_list_id = :invoiceListId
      AND curr_invoice_no = :invoiceNo
      AND invoice_type    = :invoiceType
    ```
    
    - Nếu `curr_invoice_no` đúng bằng `to_invoice` → coi như đã dùng hết dãy, cập nhật trạng thái FULL:
    
    ```sql
    UPDATE bccs_im.invoice_list
    SET status = :statusFull
    WHERE invoice_list_id = :invoiceListId
      AND invoice_type    = :invoiceType
    ```

12. **Trả kết quả thành công cho phía gọi**
    - `code = "200"`  
    - `success = true`  
    - `description = "ERR.SAE.117"` (theo chuẩn mã lỗi hệ thống cũ).  
    - Set `invoiceUsedDTO.invoiceUsedId` và `invoiceUsedDTO.saleTransId` trong response.

13. **Xử lý exception**
    - `IllegalArgumentException` → `code = "400"`, `success = false`, `description = ex.getMessage()`.
    - Exception khác → `code = "500"`, `success = false`, `description = "ERR.SAE.118"`.

14. **Đọc lại `SaleTrans` trên BCCS3** (sau khi BCCS1 xử lý xong)
    - Yêu cầu sau khi BCCS1 xử lý xong thì `SaleTrans.invoiceUsedId` trên BCCS3 đã được đồng bộ (qua cơ chế tích hợp sẵn có).
    - Nếu không tìm thấy `SaleTrans` hoặc `invoiceUsedId` vẫn null → trả lỗi `sale.logistics.create.invoice.invoice.created.but.trans.not.found`.

15. **In PDF từ InvoiceUsed vừa được tạo**
    - Hệ thống sử dụng `invoiceUsedId` vừa có để lấy đủ thông tin hóa đơn và sinh file PDF (dùng cùng cơ chế in như luồng BCCS3).

- **Kết luận luồng BCCS1:**  
  - BCCS1 chịu trách nhiệm **tạo hóa đơn** và cập nhật `invoiceUsedId` cho giao dịch.  
  - BCCS3 chỉ **đọc lại dữ liệu** và **in file PDF** dựa trên `InvoiceUsed` đã tồn tại.

### 4.2. Luồng BCCS3 (tạo hóa đơn trực tiếp trên sale-im-service)

Phần này mô tả **nghiệp vụ** khi hệ thống BCCS3 tự tạo hóa đơn 

1. **Tìm giao dịch bán hàng cần in hóa đơn**
   - Hệ thống tra cứu giao dịch theo `saleTransCode`.
   
   ```sql
   SELECT * 
   FROM sale_trans st 
   WHERE st.sale_trans_code = :saleTransCode
   ```
   
   - Nếu không tìm thấy giao dịch → trả lỗi `sale.logistics.create.invoice.sale.trans.not.found`.

2. **Kiểm tra giao dịch đủ điều kiện xuất hóa đơn**
   - Kiểm tra trường `status` của giao dịch vừa tìm được.
   - Chỉ cho phép tạo hóa đơn khi trạng thái giao dịch là **"chưa xuất hóa đơn"** (status = 2).  
   - Nếu trạng thái khác (đã xuất, đã hủy, …) → trả lỗi `sale.logistics.create.invoice.status.invalid`.
   
   > **Lưu ý:** Không cần query SQL riêng, chỉ kiểm tra giá trị field `status` từ kết quả bước 1.

3. **Lấy thông tin cửa hàng thực hiện giao dịch**
   - Dựa trên `shop_id` của giao dịch để lấy tên cửa hàng, địa chỉ, mã số thuế (`shop_tin`) phục vụ in hóa đơn.
   
   ```sql
   SELECT * 
   FROM shop 
   WHERE shop_id = :shopId
   ```
   
   - Nếu không tìm thấy cửa hàng → trả lỗi `sale.logistics.create.invoice.shop.not.found`.

4. **Chọn một dãy hóa đơn (InvoiceList) còn số cho cửa hàng**
   - Tìm trong bảng `invoice_list` 1 dãy hóa đơn:
     - Thuộc đúng cửa hàng đang giao dịch.  
     - Loại hóa đơn bán hàng (invoice_type = 1).  
     - Trạng thái "đang phát hành tại cửa hàng" (status = 1).  
     - Số hiện tại (`curr_invoice`) còn nhỏ hơn hoặc bằng số cuối (`to_invoice`).  
   
   ```sql
   SELECT is2.SERIAL_NO,
          it.INVOICE_TYPE,
          il.INVOICE_LIST_ID,
          il.FROM_INVOICE,
          il.TO_INVOICE,
          il.CURR_INVOICE,
          il.SHOP_ID,
          il.STATUS,
          it.INVOICE_NO_LENGTH,
          il.BLOCK_NO
   FROM invoice_list il,
        invoice_serial is2,
        invoice_type it
   WHERE il.INVOICE_SERIAL_ID = is2.INVOICE_SERIAL_ID
     AND is2.INVOICE_TYPE_ID = it.INVOICE_TYPE_ID
     AND il.STATUS = 1
     AND is2.STATUS = 1
     AND it.STATUS = 1
     AND il.CURR_INVOICE <= il.TO_INVOICE
     AND il.SHOP_ID = :shopUsedId
     AND it.INVOICE_TYPE = :type
   ORDER BY il.CURR_INVOICE
   LIMIT 1
   ```
   
   - Nếu không tìm thấy dãy hóa đơn thỏa mãn → trả lỗi `sale.logistics.create.invoice.invoice.list.not.found`.

5. **Khóa dãy hóa đơn đang chọn**
   - Khóa bản ghi `invoice_list` đó trong CSDL (SELECT ... FOR UPDATE) để tránh hai giao dịch cùng dùng một số hóa đơn.  
   
   ```sql
   SELECT * 
   FROM invoice_list il 
   WHERE il.STATUS = 1 
     AND il.INVOICE_LIST_ID = :invoiceListId
     AND il.CURR_INVOICE <= il.TO_INVOICE
   FOR UPDATE
   ```
   
   - Nếu không khóa được (đang bị process khác sử dụng) → trả lỗi `sale.logistics.create.invoice.invoice.list.locked`.

6. **Sinh số hóa đơn mới**
   - Tăng số thứ tự trong dãy (`invoiceId = curr_invoice + 1`).  
   - Ghép thành `invoiceNo` theo quy tắc: **kí hiệu hóa đơn (serial_no)** + **block_no** (định dạng độ dài `length_name`) + **số hóa đơn** (định dạng độ dài `length_invoice`).
   
   > **Lưu ý:** Không cần query SQL, chỉ tính toán dựa trên dữ liệu từ `InvoiceList` đã khóa ở bước 5.

7. **Tạo bản ghi hóa đơn (`invoice_used`)**
   - Ghi nhận đầy đủ thông tin:
     - Cửa hàng, nhân viên in hóa đơn.  
     - Khách hàng (tên, địa chỉ, mã số thuế nếu có, số điện thoại, công ty…).  
     - Số hóa đơn vừa sinh, ký hiệu, loại hóa đơn.  
     - Tổng tiền trước thuế, chiết khấu, VAT, tổng tiền sau thuế.  
     - Loại tiền tệ (USD/KHR), phương thức thanh toán.
   
   ```sql
   INSERT INTO invoice_used (
       invoice_used_id, invoice_list_id, invoice_datetime,
       shop_id, staff_id, receiver_id, receiver_type,
       serial_no, invoice_id, invoice_no, invoice_type, from_invoice_used_id,
       sub_id, cust_id, cust_isdn,
       cust_name, cust_name_kh, cust_address, cust_address_kh,
       cust_company, cust_company_kh,
       pay_method, note, status, create_date,
       telecom_service_id, currency,
       amount_not_tax, discount, vat, amount_tax,
       shop_name, shop_address, shop_tin,
       cust_email, request_credit_inv, last_update_user
   ) VALUES (
       :invoiceUsedId, :invoiceListId, :sysDate,
       :shopId, :staffId, :receiverId, :receiverType,
       :serialNo, :invoiceId, :invoiceNo, :invoiceType, NULL,
       :subId, :custId, :custIsdn,
       :custName, :custNameKh, :custAddress, :custAddressKh,
       :custCompany, :custCompanyKh,
       :payMethod, :note, 1, :sysDate,
       :telecomServiceId, :currency,
       :amountNotTax, :discount, :vat, :amountTax,
       :shopName, :shopAddress, :shopTin,
       :custEmail, 0, :lastUpdateUser
   )
   ```
   
   > **Lưu ý:** `invoice_used_id` được sinh từ sequence `invoice_used_seq`.

8. **Gắn số hóa đơn vào giao dịch**
   - Cập nhật lại giao dịch bán hàng:
     - Lưu `invoice_used_id` vừa tạo.  
     - Cập nhật ngày tạo hóa đơn.  
     - Đổi trạng thái giao dịch sang **"đã xuất hóa đơn"** (status = 3).
   
   ```sql
   UPDATE sale_trans 
   SET invoice_used_id = :invoiceUsedId,
       invoice_create_date = NOW(),
       status = '3',
       last_update_date = NOW()
   WHERE sale_trans_id = :saleTransId
   ```

9. **Cập nhật dãy hóa đơn**
   - Tăng số hiện tại (`curr_invoice`) trong bảng `invoice_list` để lần sau dùng số kế tiếp.  
   - Nếu đã dùng tới số cuối dãy thì đánh dấu dãy hóa đơn là **đã dùng hết** (status = 3).
   
   ```sql
   -- Nếu còn số trong dãy (curr_invoice + 1 < to_invoice)
   UPDATE invoice_list 
   SET curr_invoice = :newCurrInvoice
   WHERE invoice_list_id = :invoiceListId
   
   -- Nếu đã dùng hết dãy (curr_invoice + 1 = to_invoice)
   UPDATE invoice_list 
   SET curr_invoice = :newCurrInvoice,
       status = 3
   WHERE invoice_list_id = :invoiceListId
   ```

10. **Sinh file PDF hóa đơn**
    - Hệ thống chọn đúng file template Excel tương ứng (theo loại hóa đơn, TIN, loại khách hàng Logistics…) rồi đổ dữ liệu vào.  
    - Từ template Excel này, hệ thống generate ra file PDF và trả về cho API gọi in hóa đơn.
    
    > **Lưu ý:** Không cần query SQL, chỉ đọc dữ liệu từ `invoice_used` và `sale_trans` đã lưu để điền vào template Excel và chuyển đổi sang PDF.

- **Kết luận luồng BCCS1:**  
  - BCCS1 chịu trách nhiệm **tạo hóa đơn** và cập nhật `invoiceUsedId` cho giao dịch.  
  - BCCS3 chỉ **đọc lại dữ liệu** và **in file PDF** dựa trên `InvoiceUsed` đã tồn tại.

---

## 5. Logic chọn Template Excel cho Logistics (`templateNameLogistic`)

Phần này mô tả **duy nhất** logic chọn template cho luồng Logistics (khi `PrintInvoiceDTO.isTemplateLogistic = true`).  
Các luồng in hóa đơn thông thường (không phải Logistics) dùng logic cũ (`templateName`) và **không nằm trong phạm vi tài liệu này**.

Nếu sau khi áp dụng các rule bên dưới mà không tìm được template phù hợp, hệ thống trả lỗi `sale.invoice.print.invoice.type.invalid`.

### 5.1. Template cho Logistics (`getTemplateNameLogistic`)

Phần này dành cho **tester/BA**, nên chỉ mô tả **nghiệp vụ** (không cần hiểu Java code).

#### 5.1.1. Tham số đầu vào

- `custType` lấy từ `SaleTrans.CUST_TYPE` (mapping nghiệp vụ):
  - `INDIVIDUAL`  → Khách hàng cá nhân
  - `ORGANIZATION` → Doanh nghiệp/tổ chức
  - `STATE_BURDEN` → Hóa đơn Nhà nước chịu thuế (State Burden)
- `invoiceType`:
  - `SC` – Hóa đơn bán hàng (Standard Credit)
  - `CR` – Hóa đơn điều chỉnh (Credit)

#### 5.1.2. Ghi chú sử dụng

- Luồng Logistics **bắt buộc** set `PrintInvoiceDTO.isTemplateLogistic = true` và truyền `SaleTrans` vào `PrintInvoiceDTO.saleTrans`.  
- Chỉ khi `isTemplateLogistic = true` thì hệ thống mới áp dụng bảng logic ở trên.  
- Đối với template Logistics, phần hiển thị VAT/TAX ở footer có thể được cấu hình hiển thị `"-"` (không tính thuế) theo yêu cầu nghiệp vụ cho State Burden.

