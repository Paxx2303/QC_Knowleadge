# QC_Knowleadge
# Hướng Dẫn Manual Testing

## Giới thiệu
**Manual Testing**: Thực hiện việc test thủ công với vai trò là user để tìm bugs, error có thể nhận diện được ngay lập tức với vai trò là user sử dụng.

## Các kiểu Manual Testing

### 1. White Box Testing
Đọc code base để kiểm tra workflow, logics.

**2 kiểu test của White Box**:
- **Unit Testing**: Test 1 phần
- **Integration Testing**: Test toàn bộ

**Unit Testing có 3 phần**:
1. **Execution Testing**: So sánh output thực tế với expected output
2. **Operation Testing**: Kiểm tra metrics như performance, load, security, ...
3. **Mutation Testing**: Thay đổi 1 số trạng thái của code để kiểm tra xem test có nhận diện được thay đổi gây lỗi không

**Execution Testing gồm**:
- **Statement Coverage**: Tỉ lệ % của số dòng code đã test
- **Branch Coverage**: Tỉ lệ các nhánh của decision tree được test
- **Path Coverage**: Tỉ lệ các path đã test (mỗi path gồm statement và decision)

### 2. Black Box Testing
Test với việc chỉ dùng UI/UX, đảm bảo kết quả nhận diện như mong muốn.

**Gồm 5 techniques**:
- **Equivalence Partitioning**: Chia điều kiện kiểm thử thành các nhóm, mỗi nhóm test một trường hợp đại diện
- **Boundary Value Analysis**: Lấy 1 giá trị thật và nhận 1 khoảng sai số
- **Decision Table**: Lập bảng để test logic giữa các đầu vào
- **Exploratory Testing**: Test bởi expert
- **Error Guessing**: Không cụ thể vì làm bởi kinh nghiệm

### 3. Grey Box Testing
Nhận diện vấn đề về code qua các luồng xử lý function.

**Các techniques**:
- **Matrix Testing**: Định nghĩa tất cả các biến trong hệ thống
- **Pattern Testing**: Phân tích tại sao các lỗi lại xảy ra
- **Orthogonal Array Testing**: Kết hợp giữa White Box và Black Box để viết test có độ bao phủ tối đa với số test tối thiểu
- **Regression Testing**: Kiểm tra xem nếu kết hợp 1 chức năng thì có ảnh hưởng tới các chức năng khác không

## Viết Test Case
Test case gồm các trường hợp:

### Các loại Test Case
- **Tích cực (Positive)**: Input đúng → Output đúng
- **Tiêu cực (Negative)**: Input sai → Output sai
- **Limit Testing**: Kiểm tra giới hạn trên và dưới
- **Functional Testing**: Kiểm thử từng chức năng
- **Unit Test**: Test 1 chức năng hoặc component
- **Integration Test**: Test nhiều module tương tác với nhau
- **System Test**: Đảm bảo mọi component làm việc ổn định với nhau
- **User Acceptance Test (UAT)**: User test trực tiếp
- **UI Test**: Test visual
- **Performance Test**: Test hiệu suất trong các điều kiện khác nhau
- **Usability Test**: Kiểm tra xem mất bao lâu để user làm quen
- **Security Test**: Kiểm tra bảo mật của app
- **Smoke Test**: Kiểm tra các tính năng cốt lõi
- **Regression Test**: Đảm bảo các test trước đó hoạt động ổn định
- **Exploratory Test**: Khám phá các bugs
- **Alpha Test**: Test trên môi trường deploy
- **Beta Test**: 1 nhóm bên ngoài test
- **Database Test**: Xác thực tính toàn vẹn, tốc độ, độ chính xác của DB
