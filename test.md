# API Create Invoice - Tài liệu chi tiết

## Mục lục

1. [Tổng quan](#tổng-quan)
2. [Endpoint](#endpoint)
3. [Authentication](#authentication)
4. [Request](#request)
5. [Response](#response)
6. [Business Logic Chi Tiết](#business-logic-chi-tiết)
7. [Flow Diagram](#flow-diagram)
8. [Database Schema](#database-schema)
9. [Invoice Number Generation](#invoice-number-generation)
10. [PDF Generation Process](#pdf-generation-process)
11. [Validation Rules](#validation-rules)
12. [Error Handling](#error-handling)
13. [Transaction Management](#transaction-management)
14. [Performance Considerations](#performance-considerations)
15. [Security Considerations](#security-considerations)
16. [Testing Scenarios](#testing-scenarios)
17. [Code Examples](#code-examples)
18. [Related APIs](#related-apis)
19. [Version History](#version-history)

## Tổng quan

API này được sử dụng để tạo hóa đơn (invoice) cho một giao dịch bán hàng (sale transaction) từ phía Logistics. API sẽ tạo invoice mới, cập nhật trạng thái giao dịch và trả về file PDF hóa đơn dưới dạng Base64.

### Chức năng chính

- **Tạo Invoice:** Tạo bản ghi `InvoiceUsed` mới trong hệ thống
- **Generate Invoice Number:** Tự động tạo số hóa đơn theo format quy định
- **Update Transaction Status:** Cập nhật trạng thái giao dịch từ `NOT_BILLED` (2) sang `BILLED` (3)
- **Generate PDF:** Tạo file PDF hóa đơn từ template Excel
- **Lock Mechanism:** Sử dụng database lock để đảm bảo tính nhất quán

### Use Cases

1. **Logistics System:** Hệ thống Logistics gọi API này để tạo invoice cho các giao dịch bán hàng
2. **Automated Invoice Creation:** Tự động hóa quy trình tạo invoice sau khi hoàn tất giao dịch
3. **Invoice Management:** Quản lý và theo dõi các invoice đã được tạo

## Endpoint

```
POST /sale/logistics/create-invoice
```

## Authentication

API yêu cầu Bearer Token trong header:

```
Authorization: Bearer {token}
```

## Request

### Headers

| Header | Type | Required | Description |
|--------|------|----------|-------------|
| `Authorization` | String | Yes | Bearer token để xác thực |
| `Content-Type` | String | Yes | `application/json` |
| `Accept-Language` | String | No | Ngôn ngữ (ví dụ: `vi-la`) |

### Request Body

```json
{
  "requestId": "string",
  "saleTransCode": "string",
  "invoiceType": "string"
}
```

### Request Parameters

| Parameter | Type | Required | Description | Example |
|-----------|------|----------|-------------|---------|
| `requestId` | String | Yes | ID duy nhất của request (để tracking) | `"11111111111111"` |
| `saleTransCode` | String | Yes | Mã giao dịch bán hàng cần tạo invoice | `"SS000001000116261"` |
| `invoiceType` | String | Yes | Loại hóa đơn. Giá trị hợp lệ: `"SC"` (Standard Credit) | `"SC"` |

### Request Example

```bash
curl --location --request POST 'http://100.99.1.165:32015/sale/logistics/create-invoice' \
--header 'Accept-Language: vi-la' \
--header 'Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...' \
--header 'Content-Type: application/json' \
--data-raw '{
    "requestId": "11111111111111",
    "saleTransCode": "SS000001000116261",
    "invoiceType": "SC"
}'
```

## Response

### Success Response

**HTTP Status:** `200 OK`

```json
{
  "code": "200",
  "success": true,
  "description": "Success",
  "data": {
    "requestId": "11111111111111",
    "status": "Success",
    "invoiceFile": "JVBERi0xLjQKJeLjz9MKMyAwIG9iago8PC9MZW5ndGggND...",
    "annexFile": null,
    "errorCode": null,
    "errorDescription": null
  }
}
```

### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `requestId` | String | ID của request (giống với request gửi lên) |
| `status` | String | Trạng thái xử lý: `"Success"` hoặc error code |
| `invoiceFile` | String | File PDF hóa đơn được encode Base64 |
| `annexFile` | String | File phụ lục (nếu có), hiện tại luôn là `null` |
| `errorCode` | String | Mã lỗi (nếu có) |
| `errorDescription` | String | Mô tả lỗi (nếu có) |

### Error Response

**HTTP Status:** `400 Bad Request` hoặc `500 Internal Server Error`

```json
{
  "code": "400",
  "success": false,
  "description": "Sale trans already has invoice",
  "message": "Sale trans already has invoice"
}
```

## Business Logic Chi Tiết

### Quy trình xử lý từng bước

#### Bước 1: Validation và Kiểm tra Request

1. **Parse Request:**
   - Validate JSON format
   - Kiểm tra các field bắt buộc: `requestId`, `saleTransCode`, `invoiceType`
   - Validate `invoiceType` phải là `"SC"` (Standard Credit)

2. **Authentication:**
   - Kiểm tra Bearer token trong header
   - Validate token và lấy thông tin user từ `ActionUserHolder`
   - Nếu không có user context → trả về lỗi `400: User context not found`

#### Bước 2: Kiểm tra Sale Transaction

1. **Tìm Sale Transaction:**
   ```sql
   SELECT * FROM sale_trans WHERE sale_trans_code = ?
   ```
   - Tìm `SaleTrans` theo `saleTransCode`
   - Nếu không tìm thấy → trả về lỗi `400: Sale transaction not found`

2. **Kiểm tra Invoice đã tồn tại:**
   - Kiểm tra `saleTrans.invoiceUsedId != null && saleTrans.invoiceUsedId > 0`
   - Nếu đã có invoice → trả về lỗi `400: Sale trans already has invoice`

3. **Kiểm tra Transaction Status:**
   - Kiểm tra `saleTrans.status == "2"` (NOT_BILLED)
   - Nếu status khác `NOT_BILLED` → trả về lỗi `400: Sale transaction status is invalid for creating invoice. Status must be NOT_BILLED (2)`

4. **Lấy Shop Information:**
   ```sql
   SELECT * FROM shop WHERE shop_id = ?
   ```
   - Lấy thông tin shop từ `saleTrans.shopId`
   - Nếu không tìm thấy → trả về lỗi `400: Shop not found for sale transaction`

#### Bước 3: Lấy và Lock Invoice List

1. **Lấy Invoice List:**
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
   - Lấy invoice list available của shop với:
     - `shop_id = :shopUsedId` (shop đang sử dụng)
     - `invoice_type = :type` (loại invoice: SC/MC/CR)
     - Tất cả các bảng liên quan phải có `status = 1` (active)
     - `curr_invoice <= to_invoice` (còn số hóa đơn để dùng)
   - Join với `invoice_serial` để lấy `serial_no`
   - Join với `invoice_type` để lấy `invoice_type` và `invoice_no_length`
   - Sắp xếp theo `curr_invoice` tăng dần, lấy record đầu tiên
   - Nếu không có → trả về lỗi `400: No available invoice list found for shop`

2. **Lock Invoice List:**
   ```sql
   SELECT * FROM invoice_list 
   WHERE invoice_list_id = ? FOR UPDATE
   ```
   - Sử dụng `SELECT FOR UPDATE` để lock row
   - Nếu đang được lock bởi transaction khác → trả về lỗi `400: Invoice list is being used. Please try again`
   - Lock này đảm bảo không có 2 invoice cùng số được tạo đồng thời

#### Bước 4: Generate Invoice Number

**Công thức:**
```
invoiceNo = serialNo + blockNo (padded) + invoiceId (padded)
```

**Chi tiết:**

1. **Serial Part:**
   - Lấy từ `invoiceListDTO.serialNo`
   - Nếu null → empty string `""`

2. **Block Part:**
   - Lấy từ `invoiceListDTO.blockNo`
   - Padding với zeros dựa trên độ dài block number
   - Ví dụ: `blockNo = "12"`, padding đến độ dài cần thiết → `"0012"`

3. **Invoice ID Part:**
   - `invoiceId = invoiceListDTO.currInvoice + 1`
   - Padding với zeros: `"0".repeat(invoiceNoLength - invoiceId.length) + invoiceId`
   - `invoiceNoLength` lấy từ `invoiceListDTO.invoiceNoLength` (từ bảng `invoice_type`)
   - Ví dụ: `invoiceId = 123`, `invoiceNoLength = 7` → `"0000123"`

**Example:**
```
serialNo = "ABC" (từ invoice_serial)
blockNo = "12" (từ invoice_list) → padding → "0012"
invoiceId = currInvoice + 1 = 123 (từ invoice_list)
invoiceNoLength = 7 (từ invoice_type) → padding → "0000123"
Result: "ABC00120000123"
```

#### Bước 5: Tạo InvoiceUsed Record

**Các field được set:**

| Field | Source | Description |
|-------|--------|-------------|
| `invoiceUsedId` | `invoiceUsedRepo.getNextVal()` | Sequence ID |
| `invoiceListId` | `invoiceListDTO.invoiceListId` | Reference to invoice list |
| `invoiceDatetime` | `LocalDateTime.now()` | Thời gian tạo invoice |
| `shopId` | `saleTrans.shopId` | ID của shop |
| `staffId` | `saleTrans.staffId` | ID của nhân viên |
| `receiverId` | `saleTrans.receiverId` | ID người nhận |
| `receiverType` | `saleTrans.receiverType` | Loại người nhận |
| `serialNo` | `invoiceListDTO.serialNo` | Serial number |
| `invoiceId` | `invoiceId` | Invoice ID (tăng từ currInvoice) |
| `invoiceNo` | `invoiceNo` | Invoice number đã generate |
| `invoiceType` | `invoiceListDTO.invoiceType` | Loại invoice (1 = SC) |
| `subId` | `saleTrans.subId` | Subscription ID |
| `custId` | `saleTrans.custId` | Customer ID |
| `custIsdn` | `saleTrans.isdn` | Số điện thoại khách hàng |
| `custName` | `saleTrans.custName.trim()` | Tên khách hàng (EN) |
| `custNameKh` | `saleTrans.custNameKh.trim()` | Tên khách hàng (KH) |
| `custAddress` | `saleTrans.address.trim()` | Địa chỉ khách hàng (EN) |
| `custAddressKh` | `saleTrans.addressKh.trim()` | Địa chỉ khách hàng (KH) |
| `custCompany` | `saleTrans.company.trim()` | Tên công ty (EN) |
| `custCompanyKh` | `saleTrans.companyKh.trim()` | Tên công ty (KH) |
| `payMethod` | `saleTrans.payMethod ?? "1"` | Phương thức thanh toán |
| `note` | `saleTrans.note` | Ghi chú |
| `status` | `1L` | Status = Active |
| `currency` | `saleTrans.currency ?? "USD"` | Đơn vị tiền tệ |
| `amountNotTax` | `saleTrans.amountNotTax` | Số tiền chưa VAT |
| `tax` | `saleTrans.tax` | Thuế |
| `discount` | `saleTrans.discount` | Giảm giá |
| `vat` | `saleTrans.vat ?? 10.0` | VAT rate (%) |
| `amountTax` | `saleTrans.amountTax` | Tổng tiền có VAT |
| `shopName` | `shop.name` | Tên shop |
| `shopAddress` | `shop.address` | Địa chỉ shop |
| `shopTin` | `shop.tin ?? saleTrans.tin` | Tax ID của shop |
| `custEmail` | `saleTrans.email` | Email khách hàng |
| `lastUpdateUser` | `actionUserDTO.staffCode` | User tạo invoice |

**SQL Insert:**
```sql
INSERT INTO invoice_used (
    invoice_used_id, invoice_list_id, invoice_datetime,
    shop_id, staff_id, receiver_id, receiver_type,
    serial_no, invoice_id, invoice_no, invoice_type,
    sub_id, cust_id, cust_isdn, cust_name, cust_name_kh,
    cust_address, cust_address_kh, cust_company, cust_company_kh,
    pay_method, note, status, currency,
    amount_not_tax, tax, discount, vat, amount_tax,
    shop_name, shop_address, shop_tin, cust_email,
    last_update_user, create_date
) VALUES (?, ?, ?, ...)
```

#### Bước 6: Cập nhật SaleTrans

```sql
UPDATE sale_trans 
SET invoice_used_id = ?,
    invoice_create_date = ?,
    status = '3'
WHERE sale_trans_id = ?
```

- Set `invoiceUsedId` = ID của invoice vừa tạo
- Set `invoiceCreateDate` = thời gian hiện tại
- Set `status` = `"3"` (BILLED)

#### Bước 7: Cập nhật InvoiceList

```sql
UPDATE invoice_list 
SET curr_invoice = curr_invoice + 1
WHERE invoice_list_id = ?
```

- Tăng `currInvoice` lên 1 để dùng cho invoice tiếp theo

#### Bước 8: Generate PDF Invoice

1. **Build PrintInvoiceDTO:**
   ```java
   PrintInvoiceDTO printInvoiceDTO = buildPrintInvoiceDTO(
       invoiceUsedId,           // ID của invoice vừa tạo
       saleTrans,           // Sale transaction object
       Const.TRANS_TYPE.SALE, // transType = 1
       3L                   // addressType = 3 (dùng trực tiếp custAddress)
   );
   ```

2. **PrintInvoiceDTO Fields:**
   - `custProvinceName = ""`
   - `custDistrictName = ""`
   - `custVillageName = ""`
   - `custProvinceNameKh = ""`
   - `custDistrictNameKh = ""`
   - `custVillageNameKh = ""`
   - `invoiceUsedId` = ID invoice
   - `transId` = `saleTrans.saleTransId`
   - `transType` = `1` (SALE)
   - `addressType` = `3` (dùng trực tiếp custAddress)
   - `receiverEmail` = `saleTrans.email`
   - `custName` = `saleTrans.custName`
   - `custNameKh` = `saleTrans.custNameKh`
   - `custAddress` = `saleTrans.address` (nếu có)
   - `custAddressKh` = `saleTrans.addressKh` (nếu có)
   - `isViewMode` = `true`
   - `byFeign` = `true`

3. **Generate PDF:**
   - Gọi `saleService.printInvoiceInfoCam(printInvoiceDTO)`
   - Method này sẽ:
     - Load Excel template từ `classpath:template/`
     - Fill data vào template từ `TransInfo`
     - Convert Excel sang PDF
     - Return `byte[]` PDF

4. **Encode Base64:**
   ```java
   String invoiceFileBase64 = Base64.getEncoder().encodeToString(pdfBytes);
   ```

### Đặc điểm kỹ thuật

- **Address Type = 3:** API sử dụng `addressType = 3`, nghĩa là địa chỉ khách hàng được lấy trực tiếp từ `custAddress` trong `SaleTrans`, không cần parse từ province/district/village. Điều này giúp:
  - Linh hoạt hơn với các địa chỉ không theo cấu trúc chuẩn
  - Hỗ trợ địa chỉ quốc tế
  - Giảm thiểu lỗi khi parse địa chỉ

- **Invoice Number Format:** `{serialNo}{blockNo}{invoiceId}` (với padding zeros)
  - Đảm bảo tính duy nhất
  - Dễ đọc và tra cứu
  - Tuân thủ quy định về format invoice number

- **Transaction Status Flow:**
  ```
  NOT_BILLED (2) → [Create Invoice] → BILLED (3)
  ```

- **Concurrency Control:**
  - Sử dụng `SELECT FOR UPDATE` để lock invoice list
  - Đảm bảo không có 2 invoice cùng số được tạo đồng thời
  - Lock được release tự động khi transaction commit/rollback

## Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    POST /sale/logistics/create-invoice          │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
                    ┌────────────────┐
                    │ Validate Request│
                    │ - JSON format  │
                    │ - Required fields│
                    └────────┬───────┘
                             │
                             ▼
                    ┌────────────────┐
                    │ Authenticate   │
                    │ - Check token  │
                    │ - Get user info│
                    └────────┬───────┘
                             │
                             ▼
                    ┌────────────────┐
                    │ Find SaleTrans │
                    │ by saleTransCode│
                    └────────┬───────┘
                             │
                ┌────────────┴────────────┐
                │                         │
                ▼                         ▼
        ┌───────────────┐        ┌───────────────┐
        │ Not Found?    │        │ Has Invoice?  │
        │ Return 400    │        │ Return 400    │
        └───────────────┘        └───────────────┘
                │                         │
                └────────────┬────────────┘
                             │
                             ▼
                    ┌────────────────┐
                    │ Check Status   │
                    │ Must be = "2"  │
                    │ (NOT_BILLED)   │
                    └────────┬───────┘
                             │
                ┌────────────┴────────────┐
                │                         │
                ▼                         ▼
        ┌───────────────┐        ┌───────────────┐
        │ Invalid?      │        │ Get Shop Info │
        │ Return 400    │        └────────┬───────┘
        └───────────────┘                 │
                             ┌────────────┴────────────┐
                             │                         │
                             ▼                         ▼
                    ┌────────────────┐        ┌───────────────┐
                    │ Get InvoiceList│        │ Shop Not Found│
                    │ for Shop       │        │ Return 400    │
                    └────────┬───────┘        └───────────────┘
                             │
                ┌────────────┴────────────┐
                │                         │
                ▼                         ▼
        ┌───────────────┐        ┌───────────────┐
        │ Not Found?    │        │ Lock List     │
        │ Return 400    │        │ (SELECT FOR   │
        └───────────────┘        │  UPDATE)      │
                             └────────┬───────┘
                             │
                ┌────────────┴────────────┐
                │                         │
                ▼                         ▼
        ┌───────────────┐        ┌───────────────┐
        │ Locked?       │        │ Generate      │
        │ Return 400    │        │ Invoice Number│
        └───────────────┘        └────────┬───────┘
                             │
                             ▼
                    ┌────────────────┐
                    │ Create         │
                    │ InvoiceUsed    │
                    └────────┬───────┘
                             │
                             ▼
                    ┌────────────────┐
                    │ Update         │
                    │ SaleTrans      │
                    │ (status = 3)   │
                    └────────┬───────┘
                             │
                             ▼
                    ┌────────────────┐
                    │ Update         │
                    │ InvoiceList     │
                    │ (currInvoice++)│
                    └────────┬───────┘
                             │
                             ▼
                    ┌────────────────┐
                    │ Build          │
                    │ PrintInvoiceDTO│
                    │ (addressType=3)│
                    └────────┬───────┘
                             │
                             ▼
                    ┌────────────────┐
                    │ Generate PDF   │
                    │ from Template  │
                    └────────┬───────┘
                             │
                             ▼
                    ┌────────────────┐
                    │ Encode Base64  │
                    └────────┬───────┘
                             │
                             ▼
                    ┌────────────────┐
                    │ Return Success │
                    │ Response       │
                    └────────────────┘
```

## Database Schema

### Bảng liên quan

#### 1. `sale_trans` - Bảng giao dịch bán hàng

| Column | Type | Description | Constraints |
|--------|------|-------------|-------------|
| `sale_trans_id` | BIGINT | Primary key | NOT NULL, AUTO_INCREMENT |
| `sale_trans_code` | VARCHAR | Mã giao dịch | UNIQUE, NOT NULL |
| `shop_id` | BIGINT | ID của shop | FOREIGN KEY → shop |
| `staff_id` | BIGINT | ID nhân viên | FOREIGN KEY → staff |
| `receiver_id` | BIGINT | ID người nhận | |
| `receiver_type` | VARCHAR | Loại người nhận | |
| `status` | VARCHAR | Trạng thái | Values: "0", "1", "2", "3", "4", "6" |
| `invoice_used_id` | BIGINT | ID invoice đã tạo | FOREIGN KEY → invoice_used |
| `invoice_create_date` | DATETIME | Ngày tạo invoice | |
| `cust_id` | BIGINT | ID khách hàng | |
| `isdn` | VARCHAR | Số điện thoại | |
| `cust_name` | VARCHAR | Tên khách hàng (EN) | |
| `cust_name_kh` | VARCHAR | Tên khách hàng (KH) | |
| `address` | VARCHAR | Địa chỉ (EN) | |
| `address_kh` | VARCHAR | Địa chỉ (KH) | |
| `company` | VARCHAR | Tên công ty (EN) | |
| `company_kh` | VARCHAR | Tên công ty (KH) | |
| `email` | VARCHAR | Email | |
| `tin` | VARCHAR | Tax ID | |
| `currency` | VARCHAR | Đơn vị tiền tệ | Default: "USD" |
| `amount_not_tax` | DECIMAL | Số tiền chưa VAT | |
| `tax` | DECIMAL | Thuế | |
| `discount` | DECIMAL | Giảm giá | |
| `vat` | DECIMAL | VAT rate | Default: 10.0 |
| `amount_tax` | DECIMAL | Tổng tiền có VAT | |
| `pay_method` | VARCHAR | Phương thức thanh toán | Default: "1" |
| `note` | TEXT | Ghi chú | |
| `telecom_service_id` | BIGINT | ID dịch vụ viễn thông | |
| `sub_id` | BIGINT | Subscription ID | |

**Status Values:**
- `"0"` = TMP (Temporary)
- `"1"` = PENDING
- `"2"` = NOT_BILLED (Chưa có invoice - chỉ có thể tạo invoice ở status này)
- `"3"` = BILLED (Đã có invoice)
- `"4"` = CANCEL
- `"6"` = NEVER_CREATE_INVOICE

#### 2. `invoice_used` - Bảng invoice đã sử dụng

| Column | Type | Description | Constraints |
|--------|------|-------------|-------------|
| `invoice_used_id` | BIGINT | Primary key | NOT NULL, AUTO_INCREMENT |
| `invoice_list_id` | BIGINT | ID invoice list | FOREIGN KEY → invoice_list |
| `invoice_datetime` | DATETIME | Thời gian tạo invoice | NOT NULL |
| `shop_id` | BIGINT | ID shop | FOREIGN KEY → shop |
| `staff_id` | BIGINT | ID nhân viên | |
| `receiver_id` | BIGINT | ID người nhận | |
| `receiver_type` | VARCHAR | Loại người nhận | |
| `serial_no` | VARCHAR | Serial number | |
| `invoice_id` | BIGINT | Invoice ID | |
| `invoice_no` | VARCHAR | Invoice number | UNIQUE, NOT NULL |
| `invoice_type` | BIGINT | Loại invoice | 1=SC, 2=MC, 3=CR |
| `sub_id` | BIGINT | Subscription ID | |
| `cust_id` | BIGINT | Customer ID | |
| `cust_isdn` | VARCHAR | Số điện thoại | |
| `cust_name` | VARCHAR | Tên khách hàng (EN) | |
| `cust_name_kh` | VARCHAR | Tên khách hàng (KH) | |
| `cust_address` | VARCHAR | Địa chỉ (EN) | |
| `cust_address_kh` | VARCHAR | Địa chỉ (KH) | |
| `cust_company` | VARCHAR | Tên công ty (EN) | |
| `cust_company_kh` | VARCHAR | Tên công ty (KH) | |
| `pay_method` | VARCHAR | Phương thức thanh toán | |
| `note` | TEXT | Ghi chú | |
| `status` | BIGINT | Trạng thái | 1=Active |
| `currency` | VARCHAR | Đơn vị tiền tệ | Default: "USD" |
| `amount_not_tax` | DECIMAL | Số tiền chưa VAT | |
| `tax` | DECIMAL | Thuế | |
| `discount` | DECIMAL | Giảm giá | |
| `vat` | DECIMAL | VAT rate | |
| `amount_tax` | DECIMAL | Tổng tiền có VAT | |
| `shop_name` | VARCHAR | Tên shop | |
| `shop_address` | VARCHAR | Địa chỉ shop | |
| `shop_tin` | VARCHAR | Tax ID của shop | |
| `cust_email` | VARCHAR | Email khách hàng | |
| `last_update_user` | VARCHAR | User cập nhật cuối | |
| `create_date` | DATETIME | Ngày tạo | |

#### 3. `invoice_list` - Bảng danh sách invoice

| Column | Type | Description | Constraints |
|--------|------|-------------|-------------|
| `invoice_list_id` | BIGINT | Primary key | NOT NULL |
| `invoice_serial_id` | BIGINT | ID serial | FOREIGN KEY → invoice_serial |
| `shop_id` | BIGINT | ID shop | FOREIGN KEY → shop |
| `from_invoice` | BIGINT | Số invoice bắt đầu | |
| `to_invoice` | BIGINT | Số invoice kết thúc | |
| `curr_invoice` | BIGINT | Invoice ID hiện tại | |
| `block_no` | VARCHAR | Block number | |
| `status` | BIGINT | Trạng thái | 1=Active |

#### 3.1. `invoice_serial` - Bảng serial invoice

| Column | Type | Description | Constraints |
|--------|------|-------------|-------------|
| `invoice_serial_id` | BIGINT | Primary key | NOT NULL |
| `invoice_type_id` | BIGINT | ID loại invoice | FOREIGN KEY → invoice_type |
| `serial_no` | VARCHAR | Serial number | |
| `status` | BIGINT | Trạng thái | 1=Active |

#### 3.2. `invoice_type` - Bảng loại invoice

| Column | Type | Description | Constraints |
|--------|------|-------------|-------------|
| `invoice_type_id` | BIGINT | Primary key | NOT NULL |
| `invoice_type` | VARCHAR | Loại invoice | "SC", "MC", "CR" |
| `invoice_no_length` | BIGINT | Độ dài invoice ID | |
| `status` | BIGINT | Trạng thái | 1=Active |

#### 4. `shop` - Bảng shop

| Column | Type | Description |
|--------|------|-------------|
| `shop_id` | BIGINT | Primary key |
| `name` | VARCHAR | Tên shop |
| `address` | VARCHAR | Địa chỉ shop |
| `tin` | VARCHAR | Tax ID của shop |

## Invoice Number Generation

### Công thức chi tiết

```
invoiceNo = serialNo + blockNo_padded + invoiceId_padded
```

### Ví dụ cụ thể

**Input:**
- `serialNo` = `"ABC"` (từ `invoice_serial.serial_no`)
- `blockNo` = `"12"` (từ `invoice_list.block_no`)
- `currInvoice` = `122` (từ `invoice_list.curr_invoice`)
- `invoiceNoLength` = `7` (từ `invoice_type.invoice_no_length`)

**Processing:**

1. **Serial Part:**
   ```
   serialPart = "ABC" (lấy trực tiếp từ invoice_serial)
   ```

2. **Block Part (with padding):**
   ```
   blockNo = "12"
   // Padding block number theo độ dài cần thiết
   blockPart = "0012" (ví dụ padding đến 4 ký tự)
   ```

3. **Invoice ID Part (with padding):**
   ```
   invoiceId = currInvoice + 1 = 122 + 1 = 123
   invoiceNoLength = 7 (từ invoice_type.invoice_no_length)
   invoiceIdString = "123"
   padding = "0".repeat(7 - 3) = "0000"
   invoicePart = "0000" + "123" = "0000123"
   ```

4. **Final Result:**
   ```
   invoiceNo = "ABC" + "0012" + "0000123" = "ABC00120000123"
   ```

### Edge Cases

1. **BlockNo null hoặc empty:**
   ```
   blockNo = null → blockPart = ""
   Result: "ABC0000123"
   ```

2. **SerialNo null:**
   ```
   serialNo = null → serialPart = ""
   Result: "00120000123"
   ```

3. **BlockNo dài hơn độ dài padding:**
   ```
   blockNo = "12345"
   → Giữ nguyên blockNo, không padding thêm
   ```

4. **currInvoice >= toInvoice:**
   ```
   Nếu currInvoice >= toInvoice → Không lấy invoice_list này
   → Query sẽ không trả về record này (điều kiện: il.CURR_INVOICE <= il.TO_INVOICE)
   ```

## PDF Generation Process

### Template Loading

1. **Template Location:**
   - Templates được lưu trong `classpath:template/`
   - Tên template được xác định bởi `getTemplateName()` method
   - Format: `{invoiceType}_{templateType}.xlsx`

2. **Template Selection:**
   - Dựa trên `invoiceType` (SC, MC, CR)
   - Dựa trên `invoiceNo`
   - Dựa trên `invoiceUsedAdj` (nếu có)

### Data Mapping

**TransInfo → Excel Cells:**

| TransInfo Field | Excel Key | Description |
|----------------|-----------|-------------|
| `custName` | `{CUST_NAME}` | Tên khách hàng |
| `custNameKh` | `{CUST_NAME_KH}` | Tên khách hàng (KH) |
| `custAddress` | `{ADDRESS}` | Địa chỉ (EN) |
| `custAddressKh` | `{ADDRESS_KHR}` | Địa chỉ (KH) |
| `custIsdn` | `{ISDN}` | Số điện thoại |
| `receiverName` | `{RECEIVER_NAME}` | Tên người nhận |
| `receiverEmail` | `{RECEIVER_MAIL}` | Email người nhận |
| `tin` | `{VAT_TIN}` | Tax ID |
| `subscriber` | `{SUBSCRIBER}` | Số thuê bao |
| `invoiceNo` | `{INVOICE_NO}` | Số hóa đơn |
| `invoiceDate` | `{INVOICE_DATE}` | Ngày hóa đơn |
| `transDate` | `{TRANS_DATE}` | Ngày giao dịch |
| `currency` | `{CURRENCY}` | Đơn vị tiền tệ |
| `wordMoney` | `{WORD_MONEY}` | Số tiền bằng chữ |

### PDF Conversion

1. **Load Excel Template:**
   ```java
   XSSFWorkbook workbook = new XSSFWorkbook(file.getInputStream());
   ```

2. **Fill Data:**
   - Iterate qua các cells
   - Replace placeholders với giá trị thực
   - Tính toán row height dựa trên content length

3. **Convert to PDF:**
   - Sử dụng Apache POI để đọc Excel
   - Convert sang PDF format
   - Return `byte[]`

4. **Encode Base64:**
   ```java
   String base64 = Base64.getEncoder().encodeToString(pdfBytes);
   ```

## Validation Rules

### Request Validation

| Field | Rules | Error Message |
|-------|-------|---------------|
| `requestId` | Required, Not null, Not empty | "Request ID is required" |
| `saleTransCode` | Required, Not null, Not empty, Format: `SS\d+` | "Sale transaction code is required" |
| `invoiceType` | Required, Must be `"SC"` | "Invoice type must be SC" |

### Business Validation

| Validation | Condition | Error Code | Error Message |
|------------|-----------|------------|---------------|
| SaleTrans exists | `saleTrans == null` | 400 | "Sale transaction not found: {saleTransCode}" |
| Invoice not exists | `saleTrans.invoiceUsedId != null && saleTrans.invoiceUsedId > 0` | 400 | "Sale trans already has invoice" |
| Status is NOT_BILLED | `saleTrans.status != "2"` | 400 | "Sale transaction status is invalid for creating invoice. Status must be NOT_BILLED (2)." |
| Shop exists | `shop == null` | 400 | "Shop not found for sale transaction" |
| InvoiceList exists | `invoiceListDTO == null` | 400 | "No available invoice list found for shop" |
| InvoiceList not locked | Lock failed | 400 | "Invoice list is being used. Please try again" |
| User context exists | `actionUserDTO == null` | 400 | "User context not found" |

## Error Handling

### Error Response Structure

```json
{
  "code": "400",
  "success": false,
  "description": "Error message",
  "message": "Error message"
}
```

### Error Codes Chi Tiết

| Error Code | HTTP Status | Scenario | Root Cause | Solution |
|------------|-------------|----------|------------|----------|
| `400` | 400 Bad Request | Sale transaction not found | `saleTransCode` không tồn tại trong DB | Kiểm tra lại `saleTransCode`, đảm bảo transaction đã được tạo |
| `400` | 400 Bad Request | Sale trans already has invoice | Transaction đã có `invoiceUsedId` | Không thể tạo invoice mới, sử dụng API print-invoice để in lại |
| `400` | 400 Bad Request | Sale transaction status is invalid | Status không phải `NOT_BILLED` (2) | Chỉ có thể tạo invoice khi status = 2 |
| `400` | 400 Bad Request | No available invoice list found for shop | Shop chưa được cấu hình invoice list | Cần cấu hình invoice list cho shop trước |
| `400` | 400 Bad Request | Invoice list is being used | Invoice list đang được lock bởi transaction khác | Retry sau vài giây |
| `400` | 400 Bad Request | User context not found | Token không hợp lệ hoặc expired | Refresh token hoặc đăng nhập lại |
| `400` | 400 Bad Request | Shop not found | Shop ID trong sale_trans không tồn tại | Kiểm tra dữ liệu shop |
| `500` | 500 Internal Server Error | Database error | Lỗi kết nối DB hoặc constraint violation | Liên hệ DBA hoặc support |
| `500` | 500 Internal Server Error | PDF generation error | Lỗi khi generate PDF từ template | Kiểm tra template file, liên hệ support |

### Retry Strategy

**Khi nào nên retry:**
- Error code `400: Invoice list is being used` → Retry sau 1-2 giây
- Error code `500: Internal Server Error` → Retry sau 5 giây (tối đa 3 lần)

**Không nên retry:**
- Error code `400: Sale trans already has invoice` → Transaction đã có invoice
- Error code `400: Sale transaction not found` → Dữ liệu không đúng
- Error code `400: User context not found` → Vấn đề authentication

## Transaction Management

### Database Transaction

API sử dụng `@Transactional` annotation để đảm bảo tính nhất quán:

```java
@Transactional
public GetInvoiceResponseDTO getInvoiceLogistics(CreateInvoiceRequestDTO request)
```

### Transaction Scope

1. **Lock InvoiceList:**
   ```sql
   SELECT * FROM invoice_list WHERE invoice_list_id = ? FOR UPDATE
   ```
   - Lock được giữ cho đến khi transaction commit/rollback

2. **Insert InvoiceUsed:**
   ```sql
   INSERT INTO invoice_used (...)
   ```

3. **Update SaleTrans:**
   ```sql
   UPDATE sale_trans SET invoice_used_id = ?, status = '3' WHERE sale_trans_id = ?
   ```

4. **Update InvoiceList:**
   ```sql
   UPDATE invoice_list SET curr_invoice = curr_invoice + 1 WHERE invoice_list_id = ?
   ```

### Rollback Scenarios

Transaction sẽ rollback nếu:
- Exception xảy ra trong quá trình xử lý
- Validation fail
- Database constraint violation
- PDF generation error

### Isolation Level

- Default isolation level: `READ_COMMITTED`
- Lock mechanism: `SELECT FOR UPDATE` (pessimistic locking)

## Performance Considerations

### Optimization Tips

1. **Database Indexes:**
   - Đảm bảo có index trên `sale_trans.sale_trans_code`
   - Đảm bảo có index trên `invoice_list.shop_id`
   - Đảm bảo có index trên `invoice_used.invoice_no`

2. **Connection Pooling:**
   - Sử dụng connection pool để tối ưu database connections
   - Monitor pool size và adjust nếu cần

3. **PDF Generation:**
   - Template files nên được cache trong memory
   - Consider async PDF generation cho high-load scenarios

4. **Lock Duration:**
   - Giữ lock càng ngắn càng tốt
   - Generate PDF sau khi commit transaction nếu có thể

### Performance Metrics

| Metric | Target | Notes |
|--------|--------|-------|
| API Response Time | < 3s | Bao gồm cả PDF generation |
| Database Query Time | < 500ms | Cho mỗi query |
| PDF Generation Time | < 2s | Phụ thuộc vào template size |
| Lock Wait Time | < 1s | Trong điều kiện bình thường |

## Security Considerations

### Authentication & Authorization

1. **Bearer Token:**
   - Token phải được validate trước khi xử lý request
   - Token phải chứa đầy đủ thông tin user (`staffCode`)

2. **User Context:**
   - API yêu cầu user context từ `ActionUserHolder`
   - User context được dùng để log `lastUpdateUser`

### Data Protection

1. **Sensitive Data:**
   - Customer information (name, address, email) được lưu trong invoice
   - Đảm bảo encryption khi lưu trữ
   - Đảm bảo HTTPS khi truyền tải

2. **Invoice Number:**
   - Invoice number phải unique
   - Không được expose logic generate invoice number

### Input Validation

- Validate tất cả input từ client
- Sanitize data trước khi lưu vào database
- Prevent SQL injection (sử dụng prepared statements)

## Testing Scenarios

### Unit Test Cases

1. **Happy Path:**
   - Input: Valid request với saleTransCode chưa có invoice
   - Expected: Invoice được tạo thành công, PDF được generate

2. **SaleTrans Not Found:**
   - Input: Invalid saleTransCode
   - Expected: Error 400 "Sale transaction not found"

3. **Invoice Already Exists:**
   - Input: saleTransCode đã có invoice
   - Expected: Error 400 "Sale trans already has invoice"

4. **Invalid Status:**
   - Input: saleTrans với status != "2"
   - Expected: Error 400 "Sale transaction status is invalid"

5. **InvoiceList Locked:**
   - Input: InvoiceList đang được lock
   - Expected: Error 400 "Invoice list is being used"

### Integration Test Cases

1. **End-to-End Flow:**
   - Tạo sale transaction → Gọi create-invoice → Verify invoice được tạo → Verify PDF

2. **Concurrency Test:**
   - Gọi API đồng thời với cùng shop → Verify chỉ 1 invoice được tạo mỗi lần

3. **Error Recovery:**
   - Simulate database error → Verify transaction rollback

## Error Codes

| Error Code | HTTP Status | Description | Solution |
|------------|-------------|-------------|----------|
| `400` | 400 Bad Request | Sale transaction not found | Kiểm tra `saleTransCode` có đúng không |
| `400` | 400 Bad Request | Sale trans already has invoice | Transaction đã có invoice rồi, không thể tạo mới |
| `400` | 400 Bad Request | Sale transaction status is invalid for creating invoice. Status must be NOT_BILLED (2) | Transaction phải ở trạng thái `NOT_BILLED` (2) |
| `400` | 400 Bad Request | No available invoice list found for shop | Shop chưa có invoice list được cấu hình |
| `400` | 400 Bad Request | Invoice list is being used. Please try again | Invoice list đang được sử dụng, thử lại sau |
| `400` | 400 Bad Request | User context not found | Token không hợp lệ hoặc thiếu thông tin user |
| `400` | 400 Bad Request | Shop not found for sale transaction | Shop của transaction không tồn tại |
| `500` | 500 Internal Server Error | Lỗi hệ thống | Liên hệ support |

## Notes

1. **Idempotency:** API không hỗ trợ idempotency. Nếu gọi lại với cùng `saleTransCode` đã có invoice, sẽ trả về lỗi.

2. **Invoice File Format:** File PDF được trả về dưới dạng Base64 string. Cần decode để sử dụng:
   ```java
   byte[] pdfBytes = Base64.getDecoder().decode(invoiceFile);
   ```

3. **Address Handling:** 
   - Khi `addressType = 3`, địa chỉ được lấy trực tiếp từ `SaleTrans.address` và `SaleTrans.addressKh`
   - Không cần cung cấp `custProvinceName`, `custDistrictName`, `custVillageName`

4. **Concurrency:** API sử dụng database lock để đảm bảo không có 2 invoice cùng số được tạo đồng thời.

## Code Examples

### Java Example

#### Complete Request/Response Handling

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.http.*;
import org.springframework.web.client.RestTemplate;
import java.util.Base64;
import java.nio.file.Files;
import java.nio.file.Paths;

public class CreateInvoiceClient {
    
    private static final String API_URL = "http://100.99.1.165:32015/sale/logistics/create-invoice";
    private static final String BEARER_TOKEN = "your-bearer-token-here";
    
    public void createInvoice(String saleTransCode) {
        RestTemplate restTemplate = new RestTemplate();
        
        // Build request
        CreateInvoiceRequest request = new CreateInvoiceRequest();
        request.setRequestId(UUID.randomUUID().toString());
        request.setSaleTransCode(saleTransCode);
        request.setInvoiceType("SC");
        
        // Set headers
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.set("Authorization", "Bearer " + BEARER_TOKEN);
        headers.set("Accept-Language", "vi-la");
        
        HttpEntity<CreateInvoiceRequest> entity = new HttpEntity<>(request, headers);
        
        try {
            // Call API
            ResponseEntity<ApiResponse> response = restTemplate.exchange(
                API_URL,
                HttpMethod.POST,
                entity,
                ApiResponse.class
            );
            
            if (response.getStatusCode() == HttpStatus.OK && response.getBody().isSuccess()) {
                // Get invoice file
                String base64Invoice = response.getBody().getData().getInvoiceFile();
                
                // Decode and save
                byte[] pdfBytes = Base64.getDecoder().decode(base64Invoice);
                Files.write(Paths.get("invoice_" + saleTransCode + ".pdf"), pdfBytes);
                
                System.out.println("Invoice created successfully!");
            } else {
                System.err.println("Error: " + response.getBody().getDescription());
            }
        } catch (Exception e) {
            System.err.println("Error calling API: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    // DTO classes
    public static class CreateInvoiceRequest {
        private String requestId;
        private String saleTransCode;
        private String invoiceType;
        // getters and setters
    }
    
    public static class ApiResponse {
        private String code;
        private boolean success;
        private String description;
        private InvoiceData data;
        // getters and setters
    }
    
    public static class InvoiceData {
        private String requestId;
        private String status;
        private String invoiceFile;
        private String annexFile;
        // getters and setters
    }
}
```

#### Error Handling

```java
public void createInvoiceWithRetry(String saleTransCode, int maxRetries) {
    int retryCount = 0;
    
    while (retryCount < maxRetries) {
        try {
            ApiResponse response = callCreateInvoiceAPI(saleTransCode);
            
            if (response.isSuccess()) {
                return; // Success
            }
            
            // Check if retryable error
            if (response.getCode().equals("400") && 
                response.getDescription().contains("Invoice list is being used")) {
                retryCount++;
                Thread.sleep(2000); // Wait 2 seconds
                continue;
            }
            
            // Non-retryable error
            throw new RuntimeException("Cannot create invoice: " + response.getDescription());
            
        } catch (Exception e) {
            if (retryCount >= maxRetries - 1) {
                throw e;
            }
            retryCount++;
            try {
                Thread.sleep(5000); // Wait 5 seconds
            } catch (InterruptedException ie) {
                Thread.currentThread().interrupt();
                throw new RuntimeException(ie);
            }
        }
    }
}
```

### Python Example

#### Complete Request/Response Handling

```python
import requests
import base64
import uuid
from typing import Optional, Dict, Any

class CreateInvoiceClient:
    def __init__(self, base_url: str, bearer_token: str):
        self.base_url = base_url
        self.bearer_token = bearer_token
        self.headers = {
            "Content-Type": "application/json",
            "Authorization": f"Bearer {bearer_token}",
            "Accept-Language": "vi-la"
        }
    
    def create_invoice(self, sale_trans_code: str, invoice_type: str = "SC") -> Dict[str, Any]:
        """
        Create invoice for a sale transaction
        
        Args:
            sale_trans_code: Sale transaction code
            invoice_type: Invoice type (default: "SC")
            
        Returns:
            Dictionary containing response data
        """
        url = f"{self.base_url}/sale/logistics/create-invoice"
        
        payload = {
            "requestId": str(uuid.uuid4()),
            "saleTransCode": sale_trans_code,
            "invoiceType": invoice_type
        }
        
        try:
            response = requests.post(url, json=payload, headers=self.headers, timeout=30)
            response.raise_for_status()
            
            result = response.json()
            
            if result.get("success"):
                # Decode and save PDF
                invoice_file_base64 = result["data"]["invoiceFile"]
                pdf_bytes = base64.b64decode(invoice_file_base64)
                
                filename = f"invoice_{sale_trans_code}.pdf"
                with open(filename, "wb") as f:
                    f.write(pdf_bytes)
                
                print(f"Invoice created successfully: {filename}")
                return result
            else:
                error_msg = result.get("description", "Unknown error")
                raise Exception(f"Failed to create invoice: {error_msg}")
                
        except requests.exceptions.HTTPError as e:
            error_response = e.response.json() if e.response.content else {}
            error_msg = error_response.get("description", str(e))
            raise Exception(f"HTTP Error: {error_msg}")
        except Exception as e:
            raise Exception(f"Error calling API: {str(e)}")
    
    def create_invoice_with_retry(self, sale_trans_code: str, max_retries: int = 3) -> Dict[str, Any]:
        """
        Create invoice with retry logic for transient errors
        """
        import time
        
        for attempt in range(max_retries):
            try:
                return self.create_invoice(sale_trans_code)
            except Exception as e:
                error_msg = str(e)
                
                # Check if retryable error
                if "Invoice list is being used" in error_msg and attempt < max_retries - 1:
                    wait_time = 2 * (attempt + 1)  # Exponential backoff
                    print(f"Retryable error, waiting {wait_time} seconds...")
                    time.sleep(wait_time)
                    continue
                
                # Non-retryable error or max retries reached
                raise

# Usage
if __name__ == "__main__":
    client = CreateInvoiceClient(
        base_url="http://100.99.1.165:32015",
        bearer_token="your-bearer-token-here"
    )
    
    try:
        result = client.create_invoice_with_retry("SS000001000116261")
        print("Success:", result)
    except Exception as e:
        print("Error:", e)
```

### JavaScript/Node.js Example

```javascript
const axios = require('axios');
const fs = require('fs');
const { v4: uuidv4 } = require('uuid');

class CreateInvoiceClient {
    constructor(baseUrl, bearerToken) {
        this.baseUrl = baseUrl;
        this.bearerToken = bearerToken;
    }
    
    async createInvoice(saleTransCode, invoiceType = 'SC') {
        const url = `${this.baseUrl}/sale/logistics/create-invoice`;
        
        const payload = {
            requestId: uuidv4(),
            saleTransCode: saleTransCode,
            invoiceType: invoiceType
        };
        
        const headers = {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${this.bearerToken}`,
            'Accept-Language': 'vi-la'
        };
        
        try {
            const response = await axios.post(url, payload, { headers, timeout: 30000 });
            
            if (response.data.success) {
                // Decode and save PDF
                const base64Invoice = response.data.data.invoiceFile;
                const pdfBuffer = Buffer.from(base64Invoice, 'base64');
                
                const filename = `invoice_${saleTransCode}.pdf`;
                fs.writeFileSync(filename, pdfBuffer);
                
                console.log(`Invoice created successfully: ${filename}`);
                return response.data;
            } else {
                throw new Error(response.data.description || 'Unknown error');
            }
        } catch (error) {
            if (error.response) {
                const errorMsg = error.response.data?.description || error.message;
                throw new Error(`HTTP Error: ${errorMsg}`);
            }
            throw error;
        }
    }
    
    async createInvoiceWithRetry(saleTransCode, maxRetries = 3) {
        for (let attempt = 0; attempt < maxRetries; attempt++) {
            try {
                return await this.createInvoice(saleTransCode);
            } catch (error) {
                const errorMsg = error.message;
                
                if (errorMsg.includes('Invoice list is being used') && attempt < maxRetries - 1) {
                    const waitTime = 2000 * (attempt + 1); // Exponential backoff
                    console.log(`Retryable error, waiting ${waitTime}ms...`);
                    await new Promise(resolve => setTimeout(resolve, waitTime));
                    continue;
                }
                
                throw error;
            }
        }
    }
}

// Usage
const client = new CreateInvoiceClient(
    'http://100.99.1.165:32015',
    'your-bearer-token-here'
);

client.createInvoiceWithRetry('SS000001000116261')
    .then(result => console.log('Success:', result))
    .catch(error => console.error('Error:', error));
```

### cURL Example

#### Basic Request

```bash
curl --location --request POST 'http://100.99.1.165:32015/sale/logistics/create-invoice' \
--header 'Accept-Language: vi-la' \
--header 'Authorization: Bearer YOUR_TOKEN_HERE' \
--header 'Content-Type: application/json' \
--data-raw '{
    "requestId": "11111111111111",
    "saleTransCode": "SS000001000116261",
    "invoiceType": "SC"
}'
```

#### Save Response to File

```bash
curl --location --request POST 'http://100.99.1.165:32015/sale/logistics/create-invoice' \
--header 'Accept-Language: vi-la' \
--header 'Authorization: Bearer YOUR_TOKEN_HERE' \
--header 'Content-Type: application/json' \
--data-raw '{
    "requestId": "11111111111111",
    "saleTransCode": "SS000001000116261",
    "invoiceType": "SC"
}' \
| jq -r '.data.invoiceFile' \
| base64 -d > invoice.pdf
```

### C# Example

```csharp
using System;
using System.Net.Http;
using System.Text;
using System.Text.Json;
using System.Threading.Tasks;
using System.IO;

public class CreateInvoiceClient
{
    private readonly string _baseUrl;
    private readonly string _bearerToken;
    private readonly HttpClient _httpClient;
    
    public CreateInvoiceClient(string baseUrl, string bearerToken)
    {
        _baseUrl = baseUrl;
        _bearerToken = bearerToken;
        _httpClient = new HttpClient();
        _httpClient.DefaultRequestHeaders.Add("Authorization", $"Bearer {bearerToken}");
        _httpClient.DefaultRequestHeaders.Add("Accept-Language", "vi-la");
    }
    
    public async Task<string> CreateInvoiceAsync(string saleTransCode, string invoiceType = "SC")
    {
        var url = $"{_baseUrl}/sale/logistics/create-invoice";
        
        var request = new
        {
            requestId = Guid.NewGuid().ToString(),
            saleTransCode = saleTransCode,
            invoiceType = invoiceType
        };
        
        var json = JsonSerializer.Serialize(request);
        var content = new StringContent(json, Encoding.UTF8, "application/json");
        
        try
        {
            var response = await _httpClient.PostAsync(url, content);
            response.EnsureSuccessStatusCode();
            
            var responseJson = await response.Content.ReadAsStringAsync();
            var result = JsonSerializer.Deserialize<ApiResponse>(responseJson);
            
            if (result.Success)
            {
                // Decode and save PDF
                var base64Invoice = result.Data.InvoiceFile;
                var pdfBytes = Convert.FromBase64String(base64Invoice);
                
                var filename = $"invoice_{saleTransCode}.pdf";
                await File.WriteAllBytesAsync(filename, pdfBytes);
                
                Console.WriteLine($"Invoice created successfully: {filename}");
                return filename;
            }
            else
            {
                throw new Exception($"Failed to create invoice: {result.Description}");
            }
        }
        catch (HttpRequestException e)
        {
            throw new Exception($"HTTP Error: {e.Message}", e);
        }
    }
}

public class ApiResponse
{
    public string Code { get; set; }
    public bool Success { get; set; }
    public string Description { get; set; }
    public InvoiceData Data { get; set; }
}

public class InvoiceData
{
    public string RequestId { get; set; }
    public string Status { get; set; }
    public string InvoiceFile { get; set; }
    public string AnnexFile { get; set; }
}
```

## Related APIs

- `POST /sale/logistics/create-credit-invoice` - Tạo credit invoice
- `POST /sale/invoice/print-invoice` - In lại invoice đã tồn tại

## Version History

| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 1.0 | 2024 | Initial version | Development Team |

## Appendix

### A. Invoice Type Values

| Value | Description | Full Name |
|-------|-------------|-----------|
| `"SC"` | Standard Credit | Standard Credit Invoice |

### B. Transaction Status Values

| Status | Value | Description |
|--------|-------|-------------|
| TMP | `"0"` | Temporary transaction |
| PENDING | `"1"` | Pending approval |
| NOT_BILLED | `"2"` | Not yet invoiced (can create invoice) |
| BILLED | `"3"` | Already invoiced |
| CANCEL | `"4"` | Cancelled |
| NEVER_CREATE_INVOICE | `"6"` | Never create invoice |

### C. Address Type Values

| Address Type | Value | Description |
|--------------|-------|-------------|
| Type 1 | `1L` | Manual input with province/district/village |
| Type 2 | `2L` | Select from system with codes |
| Type 3 | `3L` | Direct address string (used by this API) |

### D. Common Constants

```java
// Transaction Types
TRANS_TYPE.SALE = 1L

// Invoice Types
INVOICE_TYPE_NUM.SC = 1L
INVOICE_TYPE_NUM.MC = 2L
INVOICE_TYPE_NUM.CR = 3L

// Status Values
SALE_TRANS_STATUS.NOT_BILLED = "2"
SALE_TRANS_STATUS.BILLED = "3"
```

### E. Template File Locations

Templates được lưu trong: `classpath:template/`

Format tên file: `{invoiceType}_{templateType}.xlsx`

Ví dụ:
- `SC_standard.xlsx`
- `SC_credit.xlsx`
- `MC_standard.xlsx`

### F. FAQ

**Q: Tại sao API trả về lỗi "Sale trans already has invoice"?**

A: Transaction đã có invoice rồi. Để in lại invoice, sử dụng API `POST /sale/invoice/print-invoice` thay vì API này.

**Q: Có thể tạo nhiều invoice cho cùng một transaction không?**

A: Không. Mỗi transaction chỉ có thể có 1 invoice. Nếu cần tạo invoice mới, phải tạo credit invoice.

**Q: Invoice number được generate như thế nào?**

A: Invoice number = `serialNo + blockNo (padded) + invoiceId (padded)`. Chi tiết xem phần [Invoice Number Generation](#invoice-number-generation).

**Q: PDF có thể được customize không?**

A: Có. PDF được generate từ Excel template. Có thể chỉnh sửa template trong `classpath:template/`.

**Q: API có hỗ trợ async processing không?**

A: Hiện tại không. API xử lý đồng bộ và trả về kết quả ngay. PDF generation có thể mất vài giây tùy vào template size.

**Q: Làm sao để handle concurrent requests?**

A: API sử dụng database lock (`SELECT FOR UPDATE`) để đảm bảo không có 2 invoice cùng số được tạo đồng thời. Nếu gặp lỗi "Invoice list is being used", retry sau vài giây.

### G. Support & Contact

- **Technical Support:** support@example.com
- **Documentation:** https://docs.example.com
- **Issue Tracker:** https://github.com/example/issues

---

**Last Updated:** 2024  
**API Version:** 1.0  
**Maintained by:** Development Team

