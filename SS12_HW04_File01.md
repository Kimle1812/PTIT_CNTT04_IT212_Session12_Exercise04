Đóng vai Senior QA Engineer có 10 năm kinh nghiệm kiểm thử hệ thống ngân hàng số.
Nhiệm vụ: Sinh Test Case, Mock Data và Unit Test cho API "Đăng ký mở tài khoản cơ bản" (Spring Boot).

Ngữ cảnh: Code API ở Bài 03 đã hoàn thành. Input: fullName, phone, email, citizenId. Output: accountId, accountNumber, status (PENDING/ACTIVE).

Yêu cầu:

Sinh Test Case: Phân tích các trường fullName, email, phone, citizenId.

Positive Test Case: Các trường hợp hợp lệ.

Negative Test Case: Các trường hợp lỗi (để trống, sai định dạng, trùng lặp...).

Boundary Test Case: Các trường hợp biên (chuỗi quá dài/quá ngắn).
→ Trả về danh sách Test Case dạng bảng.

Sinh Dữ liệu kiểm thử (Mock Data): Sinh 50 dòng dữ liệu giả lập dưới định dạng CSV, gồm các trường fullName, phone, email, citizenId.

Dữ liệu phải khớp với định dạng Việt Nam (tên VN, đầu số điện thoại VN, email hợp lệ, citizenId 12 số).

Sinh mã Unit Test: Dựa vào code Service/Controller ở Bài 03, viết các class JUnit Test (sử dụng thêm Mockito nếu cần) để test luồng Đăng ký tài khoản.

Bao phủ các trường hợp Positive, Negative, Boundary.

Output:

File 02: Bảng Test Case (Markdown).

File 03: Dữ liệu kiểm thử mock_customers.csv.

File 04: Mã nguồn class JUnit Test.
