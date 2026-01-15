
## API CLEAR DEBIT LOGISTICS – SALE-IM-SERVICE

Tài liệu này mô tả **nghiệp vụ và SQL chính** cho API clear công nợ logistics thông qua ví điện tử (eMoney), được expose trên hệ thống **sale-im-service** và dùng bởi phía **Logistics**.

- **Service**: `sale-im-service`
- **Controller**: `com.viettel.bccs3.sale.controller.LogisticsController`
- **Service impl**: `com.viettel.bccs3.sale.service.impl.LogisticsServiceImpl`
- **Mục đích**: Sau khi thanh toán thành công qua ví (eMoney), API này nhận callback từ Logistics/eMoney để:
  - Kiểm tra & đối soát danh sách giao dịch (`sale_trans`) cần clear công nợ.
  - Kiểm tra số tiền phải thu theo hệ thống có khớp với số tiền Logistics gửi sang.
  - Ghi nhận bản ghi clear công nợ vào bảng `clear_money_debit`.
  - Xóa các bản ghi tạm trong `trans_tmp_clear_debit`.

---

## 1. Endpoint

### 1.1. URL & Method

```text
POST /sale/logistics/clear-debit
```

### 1.2. Request Body (ClearDebitRequestDTO)

```json
{
  "requestId": "string",
  "tid": "string",
  "saleTrans": [
    {
      "saleTransCode": "string",
      "amount": 0
    }
  ]
}
```

- **`requestId`**: ID request từ phía Logistics/eMoney để trace log.
- **`tid`**: transaction id trên hệ thống eMoney (dùng để map với `sale_payment_info`).
- **`saleTrans`**: danh sách các giao dịch cần clear công nợ:
  - `saleTransCode`: mã giao dịch bán hàng bên BCCS3.
  - `amount`: số tiền thanh toán cho giao dịch đó (theo yêu cầu từ Logistics).

### 1.3. Response (ClearDebitResponse)

```json
{
  "requestId": "string",
  "success": true,
  "data": {
    "requestStatus": "SUCCESS | FAIL | PARTIAL_SUCCESS",
    "saleTrans": [
      {
        "saleTransCode": "string",
        "amount": 0,
        "status": "SUCCESS | FAIL",
        "errorCode": "string or null",
        "errorMessage": "string or null"
      }
    ]
  }
}
```

- `requestStatus`:
  - `"SUCCESS"`: tất cả giao dịch trong danh sách được clear công nợ thành công.
  - `"FAIL"`: không giao dịch nào được clear (thường do lỗi validate/đối soát).
  - `"PARTIAL_SUCCESS"`: một số giao dịch thành công, một số thất bại.

---

## 2. Luồng xử lý chính clear công nợ logistics

Phần này mô tả **nghiệp vụ** khi API `/sale/logistics/clear-debit` được gọi, không cần quan tâm tới tên hàm Java, chỉ tập trung vào **logic + SQL** để BA/tester dễ theo dõi.

### 2.1. Khởi tạo log callback

1. Hệ thống ghi log callback vào bảng `emoney_mapping_debit_log`:
   - Lưu **request gốc** (toString của `ClearDebitRequestDTO`).
   - `tid`, `traceId`, `createDatetime`.
   - `action` = CALLBACK, `responseStatus` mặc định = CANCELED.
2. Mục đích:
   - Theo dõi lịch sử callback giữa eMoney và BCCS3.
   - Nếu có lỗi, vẫn có log để điều tra.

> Phần này chỉ là ghi log, **không có SQL phức tạp liên quan đến nghiệp vụ clear công nợ**, nên có thể bỏ qua khi test nghiệp vụ.

### 2.2. Validate request đầu vào

Hệ thống kiểm tra:

- `requestDTO` **không được null**.
- `tid` **không được null**.
- Danh sách `saleTrans` **không được rỗng**.

Nếu **không đạt**:
- Hệ thống trả về:
  - `success = false`
  - `data.requestStatus = "FAIL"`
  - `data.saleTrans` rỗng.
- Đồng thời update log `emoney_mapping_debit_log` với response này rồi **kết thúc**.

### 2.3. Lấy danh sách `sale_trans_id` theo `saleTransCode`

1. Từ request, hệ thống lấy danh sách code:
   - `codes = [saleTrans.saleTransCode for all saleTrans != null]`.
2. Thực hiện query trên bảng `sale_trans` để lấy id tương ứng:

```sql
SELECT st.sale_trans_code,
       st.sale_trans_id
FROM   bccs3_sale_trans.sale_trans st
WHERE  st.sale_trans_code IN (:codes);
```

- Kết quả được convert thành **Map**: `code → sale_trans_id`.
- Với mỗi phần tử `saleTrans` trong request:
  - Nếu không tìm được `sale_trans_id` theo `saleTransCode` → giao dịch đó **không được tính** vào danh sách clear công nợ.
- Sau khi lọc:
  - Nếu danh sách `saleTransId` rỗng → hệ thống trả về response **FAIL** (như bước 2.2).

### 2.4. Lấy bản ghi tạm trong `trans_tmp_clear_debit`

1. Dựa trên danh sách `saleTransId` vừa tìm, hệ thống đọc bảng `trans_tmp_clear_debit` để lấy các giao dịch **đang chờ clear công nợ** cho luồng Logistics:

```sql
SELECT ttcd.*
FROM   bccs3_sale_trans.sale_trans st
JOIN   bccs3_sale_trans.trans_tmp_clear_debit ttcd
       ON ttcd.trans_id   = st.sale_trans_id
      AND ttcd.trans_type = 1
      AND ttcd.status     = 1
WHERE  st.status = 3
  AND  st.sale_trans_id IN (:saleTransIds);
```

- Điều kiện chính:
  - `sale_trans.status = 3` (đã xuất hóa đơn / đã hoàn thành giao dịch).
  - `trans_tmp_clear_debit.status = 1` (bản ghi tạm **còn hiệu lực**).
  - `trans_tmp_clear_debit.trans_type = 1` (loại transaction bán hàng).

2. Kiểm tra:
   - Nếu `lstTmp` rỗng.
   - Hoặc `lstTmp.size() != saleTransIds.size()` (số giao dịch có bản ghi tạm **không khớp** số giao dịch trong request).

→ Hệ thống coi như **không đủ điều kiện clear công nợ**, trả về response **FAIL** như bước 2.2 và lưu log.

### 2.5. Lấy danh sách công nợ phải thu thực tế (getLstTransDebit)

1. Hệ thống dùng `saleTransIds` và `transType` (lấy từ `lstTmp.get(0).getTransType()`) để tính **số tiền phải thu thực tế** cho từng giao dịch:

```sql
-- Trường hợp transType = 1 (giao dịch bán hàng thông thường)
SELECT st.isdn                                 AS account,
       SUM(ROUND((NVL(std.price, 0) * NVL(std.quantity, 0) - NVL(discount, 0)), 4))
           * (CASE WHEN st.sale_trans_type = 41 THEN -1 ELSE 1 END) AS amount,
       CASE
         WHEN ip.cust_name IS NOT NULL
          AND LENGTH(ip.cust_name) > 35
           THEN CONCAT(SUBSTRING(ip.cust_name, 1, 32), '...')
         ELSE ip.cust_name
       END                                    AS customerName,
       s.name                                 AS staffName,
       st.sale_trans_id                       AS transId,
       st.sale_trans_date                     AS transDate,
       st.sale_trans_code                     AS transCode,
       1                                      AS transType,
       NVL(st.currency, 'USD')                AS currency
FROM   bccs3_sale_trans.sale_trans         st
LEFT JOIN bccs3_sale_trans.invoice_print   ip  ON ip.invoice_used_id = st.invoice_used_id
                                               AND ip.status = 1
JOIN      bccs3_sale_trans.sale_trans_detail std ON st.sale_trans_id = std.sale_trans_id
JOIN      bccs3_sale_trans.staff           s   ON s.staff_id = st.staff_id
WHERE  st.sale_trans_id IN (:transIds)
GROUP BY st.sale_trans_id;
```

> Với `transType = 2` sẽ dùng bảng `sale_trans_general`, nhưng luồng Logistics thông thường dùng `transType = 1`, tài liệu này tập trung vào trường hợp đó.

2. Nếu không lấy được bất kỳ bản ghi công nợ nào (`lstDebit` rỗng) → trả về **FAIL** (như bước 2.2).

### 2.6. Đối soát tổng số tiền

1. Hệ thống tính:
   - `totalAmount` = tổng `amount` từ `lstDebit` (số tiền phải thu theo BCCS3).
   - `requestAmount` = tổng `amount` từ `requestDTO.saleTrans` (số tiền Logistics gửi sang).
2. Nếu `totalAmount` **khác** `requestAmount`:
   - Hệ thống **không clear công nợ**.
   - Trả về response `success = false`, `requestStatus = "FAIL"`.
   - Ghi log và kết thúc.

> Điều này đảm bảo số tiền khách đã thanh toán qua ví **khớp hoàn toàn** với số công nợ đang treo trên hệ thống.

### 2.7. Ghi nhận thanh toán qua ví (`sale_payment_info`)

Khi tổng tiền khớp, hệ thống tạo 1 bản ghi thanh toán qua ví:

```sql
INSERT INTO bccs3_sale_trans.sale_payment_info (
  type,
  pay_method,
  status,
  trans_pay_id,
  trans_pay_code,
  amount,
  create_user_id,
  create_datetime,
  last_update_date,
  last_update_user,
  staff_id,
  shop_id,
  trans_date,
  currency
)
VALUES (
  1,                     -- type = PAYMENT
  'WALLET',              -- thanh toán qua ví
  1,                     -- ACTIVE
  :tid,                  -- từ requestDTO.tid
  CONCAT('EMONEY', :tid),
  :totalAmount,
  NULL,
  SYSDATE,
  SYSDATE,
  'EMONEY_SYSTEM',
  NULL,
  NULL,
  SYSDATE,
  'USD'
);
```

- ID của bản ghi này (`sale_payment_info.id`) sẽ được dùng để link vào `clear_money_debit.clear_by_trans_id`.

### 2.8. Ghi nhận clear công nợ vào `clear_money_debit` & xóa bản ghi tạm

Với mỗi bản ghi trong `lstTmp` (tương ứng 1 giao dịch đang treo công nợ):

1. Hệ thống tạo bản ghi clear công nợ:

```sql
INSERT INTO bccs3_sale_trans.clear_money_debit (
  id,
  trans_date,
  trans_type,
  trans_id,
  create_date,
  create_by,
  status,
  clear_by_trans_id,
  date_must_approve,
  clear_type,
  max_day_debit,
  amount,
  shop_id,
  staff_id,
  telecom_service_id
)
VALUES (
  :clearMoneyDebitId,          -- từ sequence clear_money_debit_seq
  :transTmp.transDate,
  :transTmp.transType,
  :transTmp.transId,
  SYSDATE,
  'EMONEY_SYSTEM',
  1,                           -- ACTIVE
  :salePaymentInfoId,          -- id vừa tạo ở bước 2.7
  :transTmp.dateMustApprove,
  4,                           -- clear_type = 4 (luồng Logistics)
  :transTmp.maxDayDebit,
  :transTmp.amount,
  :transTmp.shopId,
  :transTmp.staffId,
  :transTmp.telecomServiceId
);
```

2. Sau khi insert thành công:

```sql
DELETE FROM bccs3_sale_trans.trans_tmp_clear_debit
WHERE id = :transTmp.id;
```

3. Đồng thời build danh sách `saleTransStatus` trả về:
   - Nếu insert/delete thành công:
     - `status = "SUCCESS"`.
   - Nếu xảy ra lỗi với 1 giao dịch:
     - Toàn bộ request có thể chuyển `requestStatus` = `"PARTIAL_SUCCESS"`.
     - Giao dịch lỗi có:
       - `status = "FAIL"`
       - `errorCode = "TRANS_NOT_FOUND"` (hoặc tương đương)
       - `errorMessage` = message chi tiết.

### 2.9. Hoàn tất response & cập nhật log

1. Nếu **tất cả giao dịch** xử lý OK:
   - `response.success = true`
   - `data.requestStatus = "SUCCESS"`
   - `data.saleTrans` = danh sách tất cả giao dịch với `status = "SUCCESS"`.
2. Nếu **một phần lỗi**:
   - `response.success = false`
   - `data.requestStatus = "PARTIAL_SUCCESS"`
   - Một số dòng trong `data.saleTrans` có `status = "FAIL"`.
3. Hệ thống cập nhật lại bản ghi log trong `emoney_mapping_debit_log`:
   - `response` (toString của `ClearDebitResponse`).
   - `responseStatus` = SUCCESS nếu toàn bộ thành công.
   - `endDatetime` = thời điểm kết thúc xử lý.

---

## 3. Ghi chú cho BA/Tester

- **Trọng tâm khi test**:
  - Đảm bảo giao dịch cần clear đã có bản ghi trong `trans_tmp_clear_debit` với `status = 1` và `sale_trans.status = 3`.
  - Số tiền Logistics gửi (`request.saleTrans.amount`) **khớp** với số tiền hệ thống tính (`lstDebit.amount`) cho từng `saleTransId`.
  - Sau khi gọi API:
    - Bản ghi trong `trans_tmp_clear_debit` tương ứng được **xóa**.
    - Bảng `clear_money_debit` có thêm bản ghi mới link với `sale_payment_info`.
- **Các tình huống lỗi hay gặp**:
  - Gửi sai `saleTransCode` → không tìm được `sale_trans_id` → `requestStatus = FAIL`.
  - Thiếu bản ghi trong `trans_tmp_clear_debit` (chưa “lên bảng tạm”) → `requestStatus = FAIL`.
  - Tổng tiền từ Logistics **không khớp** tổng công nợ thực tế → `requestStatus = FAIL`.
  - Một số bản ghi insert vào `clear_money_debit` lỗi → `requestStatus = PARTIAL_SUCCESS`.


