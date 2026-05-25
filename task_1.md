# Task 1: Test Case - Equivalence Partitioning & Decision Table

## 1. Equivalence Partitioning (Phân vùng tương đương)

**Trường kiểm thử: Tuổi**

| Phân vùng     | Trạng thái | Khoảng giá trị |
|---------------|------------|----------------|
| Invalid       | Không hợp lệ | < 18          |
| Valid         | Hợp lệ     | 18 - 69        |
| Invalid       | Không hợp lệ | > 69          |

**Trường kiểm thử: Email**

| Phân vùng       | Trạng thái     | Mô tả / Ví dụ                              | Format đại diện                  |
|-----------------|----------------|--------------------------------------------|----------------------------------|
| **Invalid 1**   | Không hợp lệ   | Không có ký tự `@`                         | `usergmail.com`                  |
| **Invalid 2**   | Không hợp lệ   | `@` ở đầu                                  | `@gmail.com`                     |
| **Invalid 3**   | Không hợp lệ   | `@` ở cuối                                 | `user@`                          |
| **Invalid 4**   | Không hợp lệ   | Nhiều ký tự `@`                            | `user@@gmail.com`                |
| **Invalid 5**   | Không hợp lệ   | Không có phần domain (sau `@`)             | `user@`                          |
| **Invalid 6**   | Không hợp lệ   | Không có phần mở rộng domain (.com, .vn..) | `user@gmail`                     |
| **Invalid 7**   | Không hợp lệ   | Chứa ký tự không cho phép                  | `user#name@gmail.com`            |
| **Valid**       | Hợp lệ         | Email đúng định dạng chuẩn                 | `user@example.com`<br>`test.user123@gmail.com.vn` |

**Trường kiểm thử: Password**
| Phân vùng       | Trạng thái     | Mô tả / Ví dụ                              | Format đại diện                  |
|-----------------|----------------|--------------------------------------------|----------------------------------|
| **Invalid**   | Không hợp lệ   | Sai kích thước ( 6 - 12 kí tự)                         | usergmail.com (13 kí tự)             |
| **Valid**       | Hợp lệ         |Đúng kích thước    ( 6 - 12 kí tự)              | 123456 |

## 2. Decision Table (Bảng Quyết Định)

**Mục tiêu**: Kiểm tra chức năng **Tạo Tài Khoản (Tạo TK)**


### Decision Table

| Input          | Case 1 | Case 2 | Case 3 | Case 4 | Case 5 | Case 6 | Case 7 | Case 8 |
|----------------|--------|--------|--------|--------|--------|--------|--------|--------|
| **Tuổi**       | Sai    | Đúng   | Sai    | Đúng   | Đúng   | Đúng   | Sai    | Sai    |
| **Email**      | Sai    | Sai    | Đúng   | Đúng   | Đúng   | Sai    | Sai    | Đúng   |
| **Password**   | Sai    | Đúng   | Đúng   | Sai    | Đúng   | Sai    | Đúng   | Sai    |
| **Tạo TK**     | **Thất bại** | **Thất bại** | **Thất bại** | **Thất bại** | **Thành công** | **Thất bại** | **Thất bại** | **Thất bại** |

### Giải thích:
- **Case 5** là trường hợp duy nhất thành công (Tuổi đúng + Email đúng + Password đúng).
- Các trường hợp còn lại đều thất bại do ít nhất một điều kiện không thỏa mãn.

---

# Task 2 
## 1. Equivalence Partitioning (Phân vùng tương đương)

**Trường kiểm thử: Số vé**

| Phân vùng     | Trạng thái | Khoảng số vế  |
|---------------|------------|----------------|
| Invalid       | Không hợp lệ | < 1          |
| Valid         | Hợp lệ     | 1 - 10      |
| Invalid       | Không hợp lệ | > 10        |

**Trường kiểm thử: Thời gian đặt vé**

| Phân vùng     | Trạng thái | Khoảng số vế  |
|---------------|------------|----------------|
| Invalid       | Không hợp lệ | 22h01' - 5h59' sáng hôm sau        |
| Valid         | Hợp lệ     | 6h - 22h    |

## 2. Decision Table (Bảng Quyết Định)

**Mục tiêu**: Kiểm tra chức năng **Tạo Tài Khoản (Tạo TK)**

- **Kết quả mong đợi**:
  - **TC** = Thành công (Tạo tài khoản thành công)
  - **TB** = Thất bại (Tạo tài khoản thất bại)

### Decision Table 

| Input          | Case 1 | Case 2 | Case 3 | Case 4 | 
|----------------|--------|--------|--------|--------|
| **Số vé**       | Sai    | Đúng   | Sai    | Đúng   | 
| **Thời gian đặt vé**      | Sai    | Sai    | Đúng   | Đúng   | 
| **Đăng kí vé**     | **Thất bại** |**Thất bại** | **Thất bại** | **Thành công** |
### Giải thích:
- **Case 4** là trường hợp duy nhất thành công (thời gian đúng + số vé đúng ).
- Các trường hợp còn lại đều thất bại do ít nhất một điều kiện không thỏa mãn.

# Task 3  
### Decision Table + Orthogonal Array Testing

| Input          | Case 1 | Case 2 | Case 3 | Case 4 | 
|----------------|--------|--------|--------|--------|
| **Email**      | Sai    | Sai    | Đúng   | Đúng   |
| **Password**   | Sai    | Đúng   | Đúng   | Sai    | 
| **Gửi OTP**     | **Không gửi OTP** | **Không gửi OTP** |**Gửi OTP**|   **Không gửi OTP**  | 

**Nếu đăng nhập thành công**
| Input          | Case 1 | Case 2 |
|----------------|--------|--------|
| **OTP**      | Sai    | Đúng   |
| **Đăng nhập**     |  **Thất bại** | **Thành công** |

# Task 4 

##  Equivalence Partitioning & Boundary Value Analysis

| Test ID | Distance (km) | VIP? | Expected Fare (VND) |         Loại giá                   |
|---------|---------------|------|---------------------|--------------------------|
| TC01    | 0             | No   | 0 hoặc Error        |  Edge case                |
| TC02    | 1-5           | No   | 10.000              |  Tier 1       |
| TC04    | 6-14          | No   | 8.000              | Tier 2                   |
| TC06    | >=15          | No   | 6.000              |  Tier 3                   |


### 2. Decision Table

| Test Case | Khoảng Km     | VIP    | Expected Result          |
|-----------|---------------|--------|--------------------------|
| DT01      | < 1           | Yes/No | 0                |
| DT02      | 1 - 5         | No     | 10.000 × km              |
| DT03      | 1 - 5         | Yes    | 10.000 × km × 0.9        |
| DT04      | 6 - 15        | No     | (5×10.000) + (km-5)×8.000      |
| DT05      | 6 - 15        | Yes    | [(5×10.000) + (km-5)×8.000] × 0.9 |
| DT06      | > 15          | No     | (5×10.000) + (9×8.000) + (km-14)×6.000 |
| DT07      | > 15          | Yes    | ((5×10.000) + (9×8.000) + (km-14)×6.000) × 0.9              |



**Ghi chú**: 
- "Đúng" = Input hợp lệ
- "Sai" = Input không hợp lệ
