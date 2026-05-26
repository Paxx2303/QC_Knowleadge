# Task 1: Test Case - Equivalence Partitioning & Decision Table


##  Boundary Value Analysis
**Trường kiểm thử: Tuổi**

 | Biên | Khoảng giá trị | Ví dụ  |
|------------|----------------|----------------|
 |  **Invalid** | < 18          | 8  |
| **Valid**     | 18 - 69        |  28  |
| **Invalid** | > 69          |  74  |
| **Invalid** | Kí tự khác kí tự số số          |  dfdf |
##  Equivalence Partitioning (Phân vùng tương đương)
**Trường kiểm thử: Email**

| Phân vùng     | Mô tả / Ví dụ                              | Format đại diện                  |
|-----------------|--------------------------------------------|----------------------------------|
| **Invalid 1**   | Không có ký tự `@`                         | `usergmail.com`                  |
| **Invalid 2**   | `@` ở đầu                                  | `@yahoo.com`                     |
| **Invalid 3**    | `@` ở cuối                                 | `devtest@`                          |
| **Invalid 4**   | Nhiều ký tự `@`                            | `chester@@gmail.com`                |
| **Invalid 6**    | Không có phần mở rộng domain (.com, .vn..) | `bloker@gmail`                     |
| **Invalid 7**     | Chứa ký tự không cho phép                  | `lessin#name@gmail.com`            |
| **Invalid 8**     | Không có miền ở giữa phần mở rộng và @                 | `lessin#name.com`            |
| **Valid**           | Email đúng định dạng chuẩn                 | `user@example.com`<br>`test.user123@gmail.com.vn` |

**Trường kiểm thử: Password**
| Phân vùng        | Mô tả / Ví dụ                              | Format đại diện                  |
|-----------------|--------------------------------------------|----------------------------------|
| **Invalid**  | Sai kích thước ( 6 - 12 kí tự)                         | usergmail.com (13 kí tự)             |
| **Valid**     |Đúng kích thước    ( 6 - 12 kí tự)              | 123456 |
| **Invalid**  | Sai kích thước ( 6 - 12 kí tự)                         | null ( 0 kí tự)             |

## 2. Decision Table (Bảng Quyết Định)

**Mục tiêu**: Kiểm tra chức năng **Tạo Tài Khoản (Tạo TK)**


### Decision Table

| Input          | Case 1 | Case 2 | Case 3 | Case 4 | Case 5 | Case 6 | Case 7 | Case 8 |
|----------------|--------|--------|--------|--------|--------|--------|--------|--------|
| **Tuổi**       | Sai  (17)  | Đúng (20)   | Sai  (70)   | Đúng (36) | Đúng ( 69) | Đúng (55)  | Sai (15)   | Sai (ddd)   |
| **Email**      | Sai (using@edu)   | Sai  ( tester.com)  | Đúng (namnq@ut.vn ) | Đúng  (tester@docs.bv)  | Đúng (testting@bkav.group )    | Sai  (hhaaa@@gmail.com)  | Sai (tesss@.com)   | Đúng (lessin@name.com)   |
| **Password**   | Sai   (12345) | Đúng (12345678)   | Đúng  (devops123)  | Sai (null)   | Đúng (@1233114N) | Sai (aaaaaaaaaaaa)    | Đúng (11223344)   | Sai (dsdifhsddfssddsf)   |
| **Tạo TK**     | **Thất bại** | **Thất bại** | **Thất bại** | **Thất bại** | **Thành công** | **Thất bại** | **Thất bại** | **Thất bại** |

### Giải thích:
- **Case 5** là trường hợp duy nhất thành công (Tuổi đúng + Email đúng + Password đúng).
- Các trường hợp còn lại đều thất bại do ít nhất một điều kiện không thỏa mãn.

---

# Task 2 
##   Boundary Value Analysis

**Trường kiểm thử: Số vé**

| Biên     | Khoảng số vế  | Ví dụ  |
|---------------|----------------|----------------|
| Invalid       | < 1          |    0 |
| Valid           | 1 - 10      | 9|
| Invalid      | > 10        | 20|

**Trường kiểm thử: Thời gian đặt vé**

| Biên    | Khoảng số vế  |Ví dụ  |
|--------------|----------------|----------------|
| Invalid      | 22h01' - 5h59' sáng hôm sau        | 22h21'|
| Valid         | 6h - 22h    | 20h21'|

## 2. Decision Table (Bảng Quyết Định)

**Mục tiêu**: Kiểm tra chức năng **Đăng kí vé**

### Decision Table 

| Input          | Case 1 | Case 2 | Case 3 | Case 4 | 
|----------------|--------|--------|--------|--------|
| **Số vé**       | Sai  (30)  | Đúng  (10)  | Sai (0)   | Đúng (8)  | 
| **Thời gian đặt vé**      | Sai (5h00)   | Sai  (22h01')   | Đúng (21h00)  | Đúng (9h00)   | 
| **Đăng kí vé**     | **Thất bại** |**Thất bại** | **Thất bại** | **Thành công** |
### Giải thích:
- **Case 4** là trường hợp duy nhất thành công (thời gian đúng + số vé đúng ).
- Các trường hợp còn lại đều thất bại do ít nhất một điều kiện không thỏa mãn.

# Task 3  

 ##Equivalence Partitioning 

**#Trường kiểm thử: OTP**
| Phân vùng        | Mô tảụ                              |
|-----------------|--------------------------------------------|
| **Invalid**  | Sai Mã OTP ( Yêu cầu : 1111 , Nhập : 1112 )                         | 
| **Valid**     |Đúng       ( Yêu cầu : 1111 , Nhập : 1111 )           |
| **Invalid**  | Sai Thời gian nhập  (  nhập đúng mã OTP , thời gian yêu cầu : 60s , thời gian nhập : 70s )  |
| **Invalid**  | Sai OTP do bấn yêu cầu gửi lại quá nhiều lần   (  nhập đúng mã OTP cũ 2345  , trong khi mã mới nhất hệ thống tạo 5678 )               |


### Decision Table + Orthogonal Array Testing
| Input          | Case 1 | Case 2 | Case 3 | Case 4 | 
|----------------|--------|--------|--------|--------|
| **Email**      | Sai    | Sai    | Đúng   | Đúng   |
| **Password**   | Sai    | Đúng   | Đúng   | Sai    | 
| **Gửi OTP**     | **Không gửi OTP** | **Không gửi OTP** |**Gửi OTP**|   **Không gửi OTP**  | 

**Nếu đăng nhập thành công và đang chờ OTP**
| Input          | Case 1 | Case 2 |
|----------------|--------|--------|
| **OTP**      | Sai OTP do bấn yêu cầu gửi lại quá nhiều lần   (  nhập đúng mã OTP cũ 2345  , trong khi mã mới nhất hệ thống tạo 5678 )         | Đúng ( Yêu cầu : 1111 , Nhập : 1111 )   |
| **Đăng nhập**     |  **Thất bại** | **Thành công** |
### Giải thích:
- **Case 2** là trường hợp thành công vì User đã đăng nhập thành công trước đó sau đó điền đúng OTP.


# Task 4 

##  Equivalence Partitioning & Boundary Value Analysis

| Test ID | Distance (km) | VIP | Expected Fare (VND) |         Loại giá                   |
|---------|---------------|------|---------------------|--------------------------|
| TC01    | 0             | No   | 0 hoặc Error        |  Edge case                |
| TC02    | 1-5           | No   | 10.000              |  Tier 1       |
| TC04    | 6-14          | No   | 8.000              | Tier 2                   |
| TC06    | >=15          | No   | 6.000              |  Tier 3                   |


### 2. Decision Table

| Test Case | Distance (km)    | VIP    | Expected Result          |
|-----------|---------------|--------|--------------------------|
| DT01      | < 1           | Yes/No | 0                |
| DT02      | 1 - 5         | No     | 10.000 × km              |
| DT03      | 1 - 5         | Yes    | 10.000 × km × 0.9        |
| DT04      | 6 - 14        | No     | (5×10.000) + (km-5)×8.000      |
| DT05      | 6 - 14        | Yes    | ((5×10.000) + (km-5)×8.000) × 0.9 |
| DT06      | > 14          | No     | (5×10.000) + (9×8.000) + (km-14)×6.000 |
| DT07      | > 14          | Yes    | ((5×10.000) + (9×8.000) + (km-14)×6.000) × 0.9              |



**Ghi chú**: 
- "Đúng" = Input hợp lệ
- "Sai" = Input không hợp lệ
