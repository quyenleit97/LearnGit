# Chi tiết các hàm xử lý eSIM ABA

## Tổng quan
File `CmWS.java` expose 13 API endpoints cho hệ thống eSIM ABA thông qua SOAP WebService. Mỗi hàm trong CmWS đóng vai trò như một wrapper, nhận request từ client và delegate xuống controller tương ứng.

---

## 1. checkFirstAccessEsim

### Signature
```java
@WebMethod(operationName = "checkFirstAccessEsim")
public CheckFirstAccessEsimOut checkFirstAccessEsim(
    @WebParam(name = "token") String token,
    @WebParam(name = "locale") String locale,
    @WebParam(name = "accountABA") String accountABA)
```

### Mục đích
Kiểm tra xem user có phải lần đầu tiên truy cập vào hệ thống eSIM hay không.

### Input Parameters
- **token**: JWT token để authentication
- **locale**: Ngôn ngữ (en/km) cho response messages
- **accountABA**: Số tài khoản ABA của user

### Output
```java
CheckFirstAccessEsimOut {
    String errorCode;
    String errorDescription;
    String isDisplay;  // "1" = first time, "0" = not first time
    List<SupportedDevice> supportedDevices;
}
```

### Logic xử lý
1. **Validate token** - Kiểm tra tính hợp lệ của JWT token
2. **Validate accountABA** - Kiểm tra accountABA không được empty
3. **Check UserEsimAccess** - Query database để kiểm tra user đã access chưa
4. **Insert/Update record** - Nếu chưa có thì insert record mới
5. **Get supported devices** - Lấy danh sách thiết bị hỗ trợ eSIM
6. **Return result** - Trả về isDisplay và danh sách devices

### Use Case
- User mở app lần đầu → Show tutorial/introduction
- User đã sử dụng → Skip introduction, đi thẳng vào main flow

---

## 2. getListIsdnRecommendForYou

### Signature
```java
@WebMethod(operationName = "getListIsdnRecommendForYou")
public ListIsdnRecommendOut getListIsdnRecommendForYou(
    @WebParam(name = "token") String token,
    @WebParam(name = "locale") String locale)
```

### Mục đích
Lấy danh sách số điện thoại eSIM được đề xuất cho user.

### Input Parameters
- **token**: JWT token
- **locale**: Ngôn ngữ

### Output
```java
ListIsdnRecommendOut {
    String errorCode;
    String errorDescription;
    Integer totalIsdn;
    List<EsimIsdn> esimIsdns;
}

EsimIsdn {
    String isdn;
    Double price;
    String status;
    // ... other fields
}
```

### Logic xử lý
1. **Validate token**
2. **Query recommended ISDNs** - Lấy danh sách ISDN được recommend
3. **Return result** - Trả về danh sách với total count

### Use Case
- Show "Recommended for you" section trong app
- Hiển thị các số đẹp, số may mắn

---

## 3. getListPackageEsimABA

### Signature
```java
@WebMethod(operationName = "getListPackageEsimABA")
public ListPackageEsimABAOut getListPackageEsimABA(
    @WebParam(name = "token") String token,
    @WebParam(name = "locale") String locale)
```

### Mục đích
Lấy danh sách các gói cước eSIM có sẵn.

### Input Parameters
- **token**: JWT token
- **locale**: Ngôn ngữ

### Output
```java
ListPackageEsimABAOut {
    String errorCode;
    String errorDescription;
    List<PackageEsimABA> packageEsimABAS;
}

PackageEsimABA {
    String packageCode;
    String packageName;
    Double price;
    String description;
    String validity;
    // ... other fields
}
```

### Logic xử lý
1. **Validate token**
2. **Query packages** - Lấy tất cả gói cước active
3. **Return result** - Trả về danh sách packages

### Use Case
- User chọn gói cước khi mua eSIM
- Hiển thị tất cả options available

---

## 4. searchIsdnEsimABA

### Signature
```java
@WebMethod(operationName = "searchIsdnEsimABA")
public SearchEsimABAOut searchIsdnEsimABA(
    @WebParam(name = "token") String token,
    @WebParam(name = "locale") String locale,
    @WebParam(name = "msisdn") String msisdn,
    @WebParam(name = "priceFrom") String priceFrom,
    @WebParam(name = "priceTo") String priceTo)
```

### Mục đích
Tìm kiếm số điện thoại eSIM theo các tiêu chí của user.

### Input Parameters
- **token**: JWT token
- **locale**: Ngôn ngữ
- **msisdn**: Pattern số điện thoại cần tìm (có thể partial)
- **priceFrom**: Giá từ
- **priceTo**: Giá đến

### Output
```java
SearchEsimABAOut {
    String errorCode;
    String errorDescription;
    Integer totalIsdn;
    List<EsimIsdn> esimIsdns;
}
```

### Logic xử lý
1. **Validate token**
2. **Get search limit** - Lấy limit config từ DB (default 30)
3. **Search with criteria** - Query ISDN theo:
   - Pattern matching với msisdn
   - Price range từ priceFrom đến priceTo
   - Limit kết quả
4. **Return result** - Trả về danh sách matching ISDNs

### Use Case
- User search số điện thoại cụ thể
- Filter theo price range
- Tìm số có pattern đặc biệt

---

## 5. checkDailyPurchaseLimit

### Signature
```java
@WebMethod(operationName = "checkDailyPurchaseLimit")
public CheckDailyPurchaseLimitOut checkDailyPurchaseLimit(
    @WebParam(name = "token") String token,
    @WebParam(name = "locale") String locale,
    @WebParam(name = "accountABA") String accountABA)
```

### Mục đích
Kiểm tra xem user đã đạt giới hạn mua hàng ngày chưa.

### Input Parameters
- **token**: JWT token
- **locale**: Ngôn ngữ
- **accountABA**: Số tài khoản ABA

### Output
```java
CheckDailyPurchaseLimitOut {
    String errorCode;
    String errorDescription;
    String message;
}
```

### Logic xử lý
1. **Validate token và accountABA**
2. **Get purchase limit** - Lấy limit từ config (resource file)
3. **Count today's orders** - Đếm số order của user trong ngày:
   - Pending orders
   - Success orders
4. **Check limits**:
   - Nếu pending >= limit → Error "purchase.limit.pending"
   - Nếu success >= limit → Error "purchase.limit.success"
   - Ngược lại → Success
5. **Return result**

### Use Case
- Prevent user mua quá nhiều eSIM trong 1 ngày
- Risk management và fraud prevention

---

## 6. idCardReaderV2EsimABA

### Signature
```java
@WebMethod(operationName = "idCardReaderV2EsimABA")
public IdCardReaderV2Out idCardReaderV2EsimABA(
    @WebParam(name = "token") String token,
    @WebParam(name = "locale") String locale,
    @WebParam(name = "image") String image,
    @WebParam(name = "isdn") String isdn,
    @WebParam(name = "accountABA") String accountABA)
```

### Mục đích
Đọc thông tin từ ảnh CMND/CCCD bằng AI/OCR.

### Input Parameters
- **token**: JWT token
- **locale**: Ngôn ngữ
- **image**: Base64 encoded image của CMND/CCCD
- **isdn**: Số điện thoại eSIM
- **accountABA**: Số tài khoản ABA

### Output
```java
IdCardReaderV2Out {
    String errorCode;
    String errorDescription;
    IdCardInfo idCardInfo;
}

IdCardInfo {
    String idNumber;
    String fullName;
    String dateOfBirth;
    String gender;
    String address;
    String issueDate;
    String placeOfIssue;
    // ... other fields
}
```

### Logic xử lý
1. **Validate request parameters**
2. **Prepare BCCS Gateway call** - Tạo request với:
   - Hard-coded token cho BCCS
   - Image data
   - ISDN và dealer info
3. **Call BCCS Gateway** - Gọi service `idCardReaderV2`
4. **Parse response** - Extract thông tin cá nhân
5. **Return result** - Trả về structured data

### Use Case
- eKYC process - tự động điền thông tin từ CMND
- Giảm manual input, tăng accuracy

---

## 7. passportReaderV2EsimABA

### Signature
```java
@WebMethod(operationName = "passportReaderV2EsimABA")
public PassportReaderV2Out passportReaderV2EsimABA(
    @WebParam(name = "token") String token,
    @WebParam(name = "locale") String locale,
    @WebParam(name = "image") String image,
    @WebParam(name = "isdn") String isdn,
    @WebParam(name = "accountABA") String accountABA)
```

### Mục đích
Đọc thông tin từ ảnh Passport bằng AI/OCR.

### Input Parameters
- **token**: JWT token
- **locale**: Ngôn ngữ
- **image**: Base64 encoded image của Passport
- **isdn**: Số điện thoại eSIM
- **accountABA**: Số tài khoản ABA

### Output
```java
PassportReaderV2Out {
    String errorCode;
    String errorDescription;
    PassportInfo passportInfo;
}

PassportInfo {
    String passportNumber;
    String fullName;
    String dateOfBirth;
    String gender;
    String nationality;
    String issueDate;
    String expiryDate;
    // ... other fields
}
```

### Logic xử lý
1. **Validate request parameters**
2. **Prepare BCCS Gateway call** - Tương tự idCardReader
3. **Call BCCS Gateway** - Gọi service `passportReaderV2`
4. **Parse response** - Extract passport info
5. **Return result**

### Use Case
- eKYC cho người nước ngoài
- Support multiple ID types

---

## 8. faceMatchV2EsimABA & mascomFaceMatchV2EsimABA

### Signature
```java
@WebMethod(operationName = "faceMatchV2EsimABA")
public FaceMatchOut faceMatchV2EsimABA(
    @WebParam(name = "token") String token,
    @WebParam(name = "locale") String locale,
    @WebParam(name = "transId") String transId,
    @WebParam(name = "pathFront") String pathFront,
    @WebParam(name = "image") String image,
    @WebParam(name = "accountABA") String accountABA)
```

### Mục đích
So khớp khuôn mặt giữa ảnh selfie và ảnh trên ID card/passport.

### Input Parameters
- **token**: JWT token
- **locale**: Ngôn ngữ
- **transId**: Transaction ID từ ID reader
- **pathFront**: Path đến ảnh ID card/passport
- **image**: Base64 ảnh selfie
- **accountABA**: Số tài khoản ABA

### Output
```java
FaceMatchOut {
    String errorCode;
    String errorDescription;
    FaceMatchInfo face;
}

FaceMatchInfo {
    Confidence confidence;
    Boolean isMatch;
}

Confidence {
    Double score;
    String level;  // HIGH, MEDIUM, LOW
}
```

### Logic xử lý
1. **Validate parameters**
2. **Call face matching service** - 2 variants:
   - `faceMatchV2EsimABA` - Standard service
   - `mascomFaceMatchV2EsimABA` - Mascom specific service
3. **Get confidence score** - AI trả về độ tin cậy
4. **Return result** - Match/Not match + confidence

### Use Case
- Xác thực danh tính trong eKYC
- Prevent identity fraud

---

## 9. getListOrderEsimABA

### Signature
```java
@WebMethod(operationName = "getListOrderEsimABA")
public GetListOrderEsimABAOut getListOrderEsimABA(
    @WebParam(name = "token") String token,
    @WebParam(name = "locale") String locale,
    @WebParam(name = "status") String status,
    @WebParam(name = "orderCode") String orderCode,
    @WebParam(name = "msisdn") String msisdn,
    @WebParam(name = "fromDate") String fromDate,
    @WebParam(name = "toDate") String toDate,
    @WebParam(name = "pageSize") Integer pageSize,
    @WebParam(name = "pageNumber") Integer pageNumber,
    @WebParam(name = "accountABA") String accountABA)
```

### Mục đích
Lấy danh sách đơn hàng eSIM của user với filtering và pagination.

### Input Parameters
- **token**: JWT token
- **locale**: Ngôn ngữ
- **status**: Filter theo trạng thái (SUCCESS, PENDING, FAIL)
- **orderCode**: Filter theo mã đơn hàng
- **msisdn**: Filter theo số điện thoại
- **fromDate/toDate**: Filter theo time range
- **pageSize/pageNumber**: Pagination
- **accountABA**: Số tài khoản ABA

### Output
```java
GetListOrderEsimABAOut {
    String errorCode;
    String errorDescription;
    Long totalOrder;
    Integer currentPage;
    List<OrderedNumber> orderedNumbers;
}

OrderedNumber {
    String orderCode;
    String isdn;
    String status;
    Date createdDate;
    Double totalAmount;
    String packageCode;
    String description;
    // ... other fields
}
```

### Logic xử lý
1. **Validate token và accountABA**
2. **Apply filters** - Build query với các điều kiện:
   - Status matching
   - Order code matching
   - MSISDN matching  
   - Date range
   - AccountABA
3. **Apply pagination** - Limit và offset
4. **Execute query** - Get data và total count
5. **Return result** - Paginated list với metadata

### Use Case
- User xem lịch sử đơn hàng
- Support search và filter
- Pagination cho performance

---

## 10. preBuyEsimABA

### Signature
```java
@WebMethod(operationName = "preBuyEsimABA")
public BuyEsimOut preBuyEsimABA(
    @WebParam(name = "token") String token,
    @WebParam(name = "locale") String locale,
    @WebParam(name = "frontSide") String frontSide,
    @WebParam(name = "backSide") String backSide,
    @WebParam(name = "portrait") String portrait,
    @WebParam(name = "encrypt") String encrypt,
    @WebParam(name = "signature") String signature)
```

### Mục đích
**Bước chuẩn bị mua eSIM** - Upload documents, validate, tạo order và tính tổng tiền.

### Input Parameters
- **token**: JWT token
- **locale**: Ngôn ngữ
- **frontSide**: Base64 ảnh mặt trước ID
- **backSide**: Base64 ảnh mặt sau ID (optional cho một số loại ID)
- **portrait**: Base64 ảnh selfie
- **encrypt**: Encrypted data chứa thông tin mua hàng
- **signature**: Digital signature để verify

### Encrypted Data Structure
```java
DataBuyEsim {
    String accountABA;
    String isdn;
    String idNumber;
    String fullName;
    String dob;
    String gender;
    String province;
    String packageCode;
    String packageName;
    String packagePrice;
    String isdnPrice;
    String transIdeKYC;
    String idType;
    String isEkyc;
    String totalAmount;
}
```

### Output
```java
BuyEsimOut {
    String errorCode;
    String errorDescription;
    String orderCode;      // Mã đơn hàng được tạo
    String requestId;      // ID cho payment
    Double totalAmount;    // Tổng tiền phải trả
    Double fee;           // Phí dịch vụ
}
```

### Logic xử lý chi tiết

#### Phase 1: Validation
1. **Decrypt & validate** - Giải mã encrypt data, verify signature
2. **Check required params** - Validate tất cả required fields
3. **Special validation** - Nếu idType=3 (passport) thì backSide required
4. **Validate merchant** - Check merchant tồn tại
5. **Validate agent** - Check agent mapping với merchant

#### Phase 2: Staff & Shop Validation  
6. **Check VTC Pay staff** - Validate staff ID có tồn tại
7. **Check shop** - Validate shop của staff

#### Phase 3: ISDN Processing
8. **Format ISDN** - Chuẩn hóa format số điện thoại
9. **Check ISDN availability** - Verify ISDN chưa bị lock/active
10. **Calculate pricing**:
    - Get sale service code
    - Calculate saleServicePrice
    - Calculate totalAmount = isdnPrice + packagePrice + saleServicePrice
11. **Lock ISDN** - Reserve ISDN cho user này

#### Phase 4: Document Processing
12. **Process images**:
    - FRONT_SIDE (required)
    - BACK_SIDE (conditional)  
    - PORTRAIT (required)
13. **Generate image names** - Unique filename với timestamp
14. **Upload to FTP** - Save ảnh lên FTP server (hiện tại đang comment)
15. **Insert image details** - Lưu metadata vào DB

#### Phase 5: Order Creation
16. **Generate order code** - Format "OD" + timestamp + sequence
17. **Generate request ID** - Format yyyyMMddHH + 10-digit sequence
18. **Calculate expiry** - Order expires sau 15 phút
19. **Insert order record** - Status = WAITING_FOR_PAYMENT
20. **Commit transaction**

### Use Case
- User hoàn tất form mua eSIM
- System validate và tạo order chờ thanh toán
- User được redirect đến ABA để thanh toán

---

## 11. buyEsimABA ⭐ (Hàm quan trọng nhất)

### Signature
```java
@WebMethod(operationName = "buyEsimABA")
public BuyEsimOut buyEsimABA(
    @WebParam(name = "token") String token,
    @WebParam(name = "locale") String locale,
    @WebParam(name = "encrypt") String encrypt,
    @WebParam(name = "signature") String signature)
```

### Mục đích
**Thực hiện mua eSIM** sau khi user thanh toán thành công. Đây là hàm core của toàn bộ hệ thống.

### Input Parameters
- **token**: JWT token
- **locale**: Ngôn ngữ  
- **encrypt**: Encrypted data chứa thông tin order
- **signature**: Digital signature

### Encrypted Data Structure
```java
DataBuyEsim {
    String accountABA;
    String isdn;
    String orderCode;        // Order từ preBuy
    String idType;
    String idNumber;
    String fullName;
    String dob;
    String gender;
    String province;
    String packageCode;
    String packageName;
    String packagePrice;
    String isdnPrice;
    String transIdeKYC;
    String paymentReference;  // ABA payment reference
    String totalAmount;
}
```

### Output
```java
BuyEsimOut {
    String errorCode;
    String errorDescription;
    String isdn;              // Số điện thoại eSIM
    String serial;            // Serial number của eSIM
    String qrEsim;           // QR code để activate eSIM
    String simExpireDate;    // Ngày hết hạn SIM
}
```

### Logic xử lý chi tiết

#### Phase 1: Validation & Setup
1. **Open 6 database sessions**:
   - SESSION_CM_PRE (Customer Pre-paid)
   - SESSION_MERCHANT (Merchant data)
   - SESSION_IM (Inventory Management)
   - SESSION_CM_POS (Point of Sale)
   - SESSION_ANY_PAY (Payment)
   - SESSION_PRODUCT (Product)

2. **Decrypt & validate** - Giải mã data, verify signature
3. **Validate required params** - Check tất cả mandatory fields
4. **Validate merchant & agent** - Kiểm tra merchant và agent mapping

#### Phase 2: Order Validation
5. **Format ISDN** - Chuẩn hóa số điện thoại
6. **Check ISDN lock status** - Verify ISDN đang bị lock (status=5)
7. **Get order by orderCode** - Lấy order từ preBuy step
8. **Validate order exists** - Order phải tồn tại và valid

#### Phase 3: Inventory Management
9. **Check serial stock** - Verify còn serial eSIM trong kho
10. **Validate staff & shop** - Double-check staff và shop
11. **Get available serial** - Lấy 1 serial từ EsimQRManagement
12. **Deduct stock** - Trừ stock và reserve serial

#### Phase 4: Service Connection
13. **Bind ISDN to Serial**:
    - Get IMSI từ serial
    - Link ISDN với serial trong DB
    - Update inventory records

14. **Generate QR Code** - Tạo QR code từ serial info

#### Phase 5: Customer Management
15. **Check existing customer**:
    - Nếu idType != "5" (others) → Search by idNumber
    - Nếu tìm thấy → Link với customer cũ
    - Nếu không → Tạo customer mới

16. **Process customer images**:
    - Link uploaded images với customer
    - Update customer info

17. **Handle SubMb record**:
    - Check SubMb cho ISDN+serial
    - Update customer info trong SubMb
    - Set personal details (name, DOB, gender, address)

#### Phase 6: Service Activation  
18. **Update order status** = SUCCESS
19. **Save QR code** và payment reference
20. **Commit all transactions**

21. **Activate package** (if packageCode exists):
    - Call BCCS service để active gói cước
    - Service type = "EXCHANGE_DATA_OSJA"
    - Handle success/failure

#### Phase 7: Finalization
22. **Activate subscriber** - Call activeSubFirst()
23. **Log success** - Ghi log chi tiết giao dịch
24. **Return result** - ISDN, serial, QR code, expiry date

### Error Handling
- **Transaction rollback** nếu có lỗi ở bất kỳ step nào
- **Stock rollback** nếu BCCS call fail
- **Detailed logging** cho debugging
- **Graceful degradation** cho optional services

### Use Case
- User thanh toán xong ở ABA
- ABA redirect về app với payment confirmation
- App call buyEsimABA để hoàn tất việc mua
- User nhận được eSIM có thể sử dụng ngay

---

## 12. checkPaymentStatusABA

### Signature
```java
@WebMethod(operationName = "checkPaymentStatusABA")
public BuyEsimOut checkPaymentStatusABA(
    @WebParam(name = "token") String token,
    @WebParam(name = "locale") String locale,
    @WebParam(name = "encrypt") String encrypt,
    @WebParam(name = "signature") String signature)
```

### Mục đích
Kiểm tra trạng thái thanh toán từ ABA Bank để confirm payment trước khi call buyEsimABA.

### Input Parameters
- **token**: JWT token
- **locale**: Ngôn ngữ
- **encrypt**: Encrypted data chứa requestId và amount
- **signature**: Digital signature

### Encrypted Data Structure
```java
DataBuyEsim {
    String requestId;    // Request ID từ preBuy
    String amount;       // Số tiền đã thanh toán
}
```

### Output
```java
BuyEsimOut {
    String errorCode;
    String errorDescription;
    CheckStatusResponseDTO checkStatusResponseDTO;
}

CheckStatusResponseDTO {
    String state;              // Payment state từ ABA
    String paymentReference;   // ABA payment reference
    String transactionId;      // ABA transaction ID
    // ... other ABA fields
}
```

### Logic xử lý chi tiết

#### Phase 1: Validation
1. **Decrypt & validate** - Giải mã data, verify signature
2. **Validate requestId** - RequestId không được empty
3. **Prepare ABA request** - Tạo CheckStatusRequestDTO

#### Phase 2: ABA Integration
4. **Call ABA Gateway**:
   - Service: `MiniAppGwClient.paymentCheckStatus()`
   - Input: requestId + amount
   - Timeout handling

5. **Log transaction**:
   - Insert vào LogPartnerAba table
   - Log request content
   - Log response content
   - Track API call

#### Phase 3: Response Processing
6. **Handle ABA response**:
   - **Success**: Parse payment state và reference
   - **Failure**: Log error và return appropriate code
   - **Timeout**: Return timeout error

7. **Payment state mapping**:
   - **"5000"**: Payment success
   - **"3000"**: Payment pending  
   - **"2002"**: Payment processing
   - **Other**: Payment failed

8. **Return result** with payment details

### ABA Payment States
- **5000**: Transaction successful
- **3000**: Transaction pending
- **2002**: Transaction processing
- **4000**: Transaction failed
- **1000**: Transaction cancelled

### Use Case
- User thanh toán xong nhưng chưa chắc success
- App poll status để confirm trước khi proceed
- Retry mechanism cho pending payments
- Error handling cho failed payments

---

## Luồng tích hợp hoàn chỉnh

### Happy Path Flow
```
1. checkFirstAccessEsim          → Show tutorial (first time)
2. getListPackageEsimABA         → User chọn package
3. searchIsdnEsimABA             → User chọn ISDN
4. checkDailyPurchaseLimit       → Verify can purchase
5. idCardReaderV2EsimABA         → Read ID document
6. faceMatchV2EsimABA           → Verify identity
7. preBuyEsimABA                → Create order, upload docs
8. [User goes to ABA to pay]    → External payment
9. checkPaymentStatusABA         → Verify payment (optional)
10. buyEsimABA                  → Complete purchase, get eSIM
11. getListOrderEsimABA         → View order history
```

### Error Scenarios
- **Invalid token** → Authentication failed
- **Daily limit exceeded** → Cannot purchase more
- **ISDN not available** → Stock out or locked
- **eKYC failed** → Identity verification failed
- **Payment failed** → ABA payment unsuccessful
- **System error** → DB/Service failures

### Security Features
- **JWT token validation** cho tất cả APIs
- **Encrypt/Signature** cho sensitive operations
- **ISDN locking** để prevent race conditions
- **Transaction management** với rollback
- **Audit logging** cho compliance

### Performance Considerations
- **Database session pooling** (6 different sessions)
- **Connection management** với proper cleanup
- **Pagination** cho large datasets
- **Timeout handling** cho external calls
- **Stock management** optimization

Đây là một hệ thống phức tạp với nhiều integration points, đòi hỏi careful error handling và transaction management để đảm bảo data consistency và user experience tốt.
