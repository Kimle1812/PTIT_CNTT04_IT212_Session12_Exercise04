
Trò chuyện với Gemini
package org.example.session12.controller;



import com.fasterxml.jackson.databind.ObjectMapper;

import org.example.session12.config.SecurityConfig;

import org.example.session12.dto.AccountRegisterRequest;

import org.example.session12.dto.AccountRegisterResponse;

import org.example.session12.service.AccountService;

import org.junit.jupiter.api.DisplayName;

import org.junit.jupiter.api.Test;

import org.springframework.beans.factory.annotation.Autowired;

import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;

import org.springframework.boot.test.mock.mockito.MockBean;

import org.springframework.context.annotation.Import;

import org.springframework.http.MediaType;

import org.springframework.test.web.servlet.MockMvc;

import static org.mockito.ArgumentMatchers.any;

import static org.mockito.Mockito.when;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;

import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;

import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;



/**

* Lớp Unit Test kiểm thử API Controller đăng ký tài khoản (AccountController).

* Sử dụng MockMvc để giả lập các cuộc gọi HTTP RESTful, kiểm định dữ liệu validation và mã HTTP status trả về.

*

* @author Senior QA Engineer

* @since 2026-07-01

*/

@WebMvcTest(AccountController.class)

@Import(SecurityConfig.class)

class AccountControllerTest {



@Autowired

private MockMvc mockMvc;



@MockBean

private AccountService accountService;



@Autowired

private ObjectMapper objectMapper;



@Test

@DisplayName("TC-01: Gửi Request hợp lệ -> Trả về 201 Created")

void registerBasicAccount_ValidRequest_ReturnsCreated() throws Exception {

// GIVEN: Tạo DTO hợp lệ

AccountRegisterRequest request = AccountRegisterRequest.builder()

.fullName("Nguyen Van An")

.phone("0325412365")

.email("an.nguyen@gmail.com")

.citizenId("001095012345")

.build();



AccountRegisterResponse response = AccountRegisterResponse.builder()

.accountId("550e8400-e29b-41d4-a716-446655440000")

.accountNumber("999123456789")

.status("PENDING")

.build();



when(accountService.registerBasicAccount(any(AccountRegisterRequest.class))).thenReturn(response);



// WHEN & THEN: Thực thi cuộc gọi REST POST và xác thực kết quả

mockMvc.perform(post("/api/v1/accounts/register")

.contentType(MediaType.APPLICATION_JSON)

.content(objectMapper.writeValueAsString(request)))

.andExpect(status().isCreated())

.andExpect(jsonPath("$.accountId").value("550e8400-e29b-41d4-a716-446655440000"))

.andExpect(jsonPath("$.accountNumber").value("999123456789"))

.andExpect(jsonPath("$.status").value("PENDING"));

}



@Test

@DisplayName("TC-04: Để trống Họ tên -> Trả về 400 Bad Request")

void registerBasicAccount_EmptyFullName_ReturnsBadRequest() throws Exception {

// GIVEN: Tên trống

AccountRegisterRequest request = AccountRegisterRequest.builder()

.fullName("")

.phone("0325412365")

.email("an.nguyen@gmail.com")

.citizenId("001095012345")

.build();



// WHEN & THEN

mockMvc.perform(post("/api/v1/accounts/register")

.contentType(MediaType.APPLICATION_JSON)

.content(objectMapper.writeValueAsString(request)))

.andExpect(status().isBadRequest())

.andExpect(jsonPath("$.error").value("Bad Request"))

.andExpect(jsonPath("$.message").value("Dữ liệu đầu vào không hợp lệ"))

.andExpect(jsonPath("$.details.fullName").value("Họ và tên không được để trống"));

}



@Test

@DisplayName("TC-07: Sai định dạng Số điện thoại -> Trả về 400 Bad Request")

void registerBasicAccount_InvalidPhone_ReturnsBadRequest() throws Exception {

// GIVEN: SĐT đầu 02 (không phải di động) hoặc không đúng định dạng VN

AccountRegisterRequest request = AccountRegisterRequest.builder()

.fullName("Nguyen Van An")

.phone("0243123456")

.email("an.nguyen@gmail.com")

.citizenId("001095012345")

.build();



// WHEN & THEN

mockMvc.perform(post("/api/v1/accounts/register")

.contentType(MediaType.APPLICATION_JSON)

.content(objectMapper.writeValueAsString(request)))

.andExpect(status().isBadRequest())

.andExpect(jsonPath("$.details.phone").value("Số điện thoại không đúng định dạng nhà mạng Việt Nam"));

}



@Test

@DisplayName("TC-12: Sai định dạng Email -> Trả về 400 Bad Request")

void registerBasicAccount_InvalidEmail_ReturnsBadRequest() throws Exception {

// GIVEN: Email không có @

AccountRegisterRequest request = AccountRegisterRequest.builder()

.fullName("Nguyen Van An")

.phone("0325412365")

.email("an.nguyen.gmail.com")

.citizenId("001095012345")

.build();



// WHEN & THEN

mockMvc.perform(post("/api/v1/accounts/register")

.contentType(MediaType.APPLICATION_JSON)

.content(objectMapper.writeValueAsString(request)))

.andExpect(status().isBadRequest())

.andExpect(jsonPath("$.details.email").value("Định dạng Email không hợp lệ"));

}



@Test

@DisplayName("TC-18: Số CCCD có mã tỉnh không hợp lệ (000) -> Trả về 400 Bad Request")

void registerBasicAccount_InvalidCitizenIdProvince_ReturnsBadRequest() throws Exception {

// GIVEN: CCCD bắt đầu bằng 000 (tỉnh không tồn tại)

AccountRegisterRequest request = AccountRegisterRequest.builder()

.fullName("Nguyen Van An")

.phone("0325412365")

.email("an.nguyen@gmail.com")

.citizenId("000095012345")

.build();



// WHEN & THEN

mockMvc.perform(post("/api/v1/accounts/register")

.contentType(MediaType.APPLICATION_JSON)

.content(objectMapper.writeValueAsString(request)))

.andExpect(status().isBadRequest())

.andExpect(jsonPath("$.details.citizenId").value("Mã tỉnh/thành phố (3 số đầu) trên CCCD không hợp lệ"));

}

}

package org.example.session12.service.impl;



import org.example.session12.dto.AccountRegisterRequest;

import org.example.session12.dto.AccountRegisterResponse;

import org.example.session12.entity.Account;

import org.example.session12.entity.AccountStatus;

import org.example.session12.exception.DuplicateResourceException;

import org.example.session12.repository.AccountRepository;

import org.junit.jupiter.api.BeforeEach;

import org.junit.jupiter.api.DisplayName;

import org.junit.jupiter.api.Test;

import org.junit.jupiter.api.extension.ExtendWith;

import org.mockito.InjectMocks;

import org.mockito.Mock;

import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;

import static org.mockito.ArgumentMatchers.any;

import static org.mockito.Mockito.*;



/**

* Lớp Unit Test kiểm thử tầng Service xử lý đăng ký tài khoản (AccountServiceImpl).

* Sử dụng thư viện Mockito để cô lập tầng Service và giả lập kết quả từ Repository.

*

* @author Senior QA Engineer

* @since 2026-07-01

*/

@ExtendWith(MockitoExtension.class)

class AccountServiceImplTest {



@Mock

private AccountRepository accountRepository;



@InjectMocks

private AccountServiceImpl accountService;



private AccountRegisterRequest validRequest;



@BeforeEach

void setUp() {

validRequest = AccountRegisterRequest.builder()

.fullName("Nguyen Van An")

.phone("0325412365")

.email("an.nguyen@gmail.com")

.citizenId("001095012345")

.build();

}



@Test

@DisplayName("TC-01: Đăng ký tài khoản hợp lệ - Thành công")

void registerBasicAccount_ValidRequest_Success() {

// GIVEN: Giả lập các điều kiện kiểm tra trùng lặp đều không tồn tại

when(accountRepository.existsByPhone(validRequest.getPhone())).thenReturn(false);

when(accountRepository.existsByEmail(validRequest.getEmail())).thenReturn(false);

when(accountRepository.existsByCitizenId(validRequest.getCitizenId())).thenReturn(false);

when(accountRepository.existsByAccountNumber(anyString())).thenReturn(false);



// Giả lập lưu thành công trả về thực thể Account có ID và UUID

Account mockSavedAccount = Account.builder()

.id(1L)

.accountId("550e8400-e29b-41d4-a716-446655440000")

.fullName(validRequest.getFullName())

.phone(validRequest.getPhone())

.email(validRequest.getEmail())

.citizenId(validRequest.getCitizenId())

.accountNumber("999123456789")

.status(AccountStatus.PENDING)

.build();

when(accountRepository.save(any(Account.class))).thenReturn(mockSavedAccount);



// WHEN: Thực hiện gọi hàm xử lý nghiệp vụ đăng ký

AccountRegisterResponse response = accountService.registerBasicAccount(validRequest);



// THEN: Kiểm chứng kết quả trả về khớp với mong đợi

assertNotNull(response);

assertEquals("550e8400-e29b-41d4-a716-446655440000", response.getAccountId());

assertEquals("999123456789", response.getAccountNumber());

assertEquals("PENDING", response.getStatus());



// Kiểm tra xem Repository có được gọi chính xác các phương thức cần thiết hay không

verify(accountRepository, times(1)).existsByPhone(validRequest.getPhone());

verify(accountRepository, times(1)).existsByEmail(validRequest.getEmail());

verify(accountRepository, times(1)).existsByCitizenId(validRequest.getCitizenId());

verify(accountRepository, times(1)).save(any(Account.class));

}



@Test

@DisplayName("TC-10: Đăng ký trùng Số điện thoại - Trả về lỗi DuplicateResourceException")

void registerBasicAccount_DuplicatePhone_ThrowsException() {

// GIVEN: Giả lập số điện thoại đã tồn tại trong database

when(accountRepository.existsByPhone(validRequest.getPhone())).thenReturn(true);



// WHEN & THEN: Gọi hàm và kiểm chứng ném ra ngoại lệ DuplicateResourceException

DuplicateResourceException exception = assertThrows(DuplicateResourceException.class, () -> {

accountService.registerBasicAccount(validRequest);

});



assertEquals("Số điện thoại này đã được đăng ký sử dụng.", exception.getMessage());


// Xác nhận không thực hiện các bước lưu hay kiểm tra trùng phía sau

verify(accountRepository, never()).existsByEmail(anyString());

verify(accountRepository, never()).existsByCitizenId(anyString());

verify(accountRepository, never()).save(any(Account.class));

}



@Test

@DisplayName("TC-13: Đăng ký trùng Email - Trả về lỗi DuplicateResourceException")

void registerBasicAccount_DuplicateEmail_ThrowsException() {

// GIVEN: SĐT không trùng nhưng Email trùng

when(accountRepository.existsByPhone(validRequest.getPhone())).thenReturn(false);

when(accountRepository.existsByEmail(validRequest.getEmail())).thenReturn(true);



// WHEN & THEN

DuplicateResourceException exception = assertThrows(DuplicateResourceException.class, () -> {

accountService.registerBasicAccount(validRequest);

});



assertEquals("Địa chỉ email này đã được đăng ký sử dụng.", exception.getMessage());



verify(accountRepository, times(1)).existsByPhone(validRequest.getPhone());

verify(accountRepository, times(1)).existsByEmail(validRequest.getEmail());

verify(accountRepository, never()).existsByCitizenId(anyString());

verify(accountRepository, never()).save(any(Account.class));

}



@Test

@DisplayName("TC-20: Đăng ký trùng Số CCCD - Trả về lỗi DuplicateResourceException")

void registerBasicAccount_DuplicateCitizenId_ThrowsException() {

// GIVEN: SĐT và Email không trùng nhưng CCCD trùng

when(accountRepository.existsByPhone(validRequest.getPhone())).thenReturn(false);

when(accountRepository.existsByEmail(validRequest.getEmail())).thenReturn(false);

when(accountRepository.existsByCitizenId(validRequest.getCitizenId())).thenReturn(true);



// WHEN & THEN

DuplicateResourceException exception = assertThrows(DuplicateResourceException.class, () -> {

accountService.registerBasicAccount(validRequest);

});



assertEquals("Số CCCD này đã được đăng ký cho một tài khoản khác.", exception.getMessage());



verify(accountRepository, times(1)).existsByPhone(validRequest.getPhone());

verify(accountRepository, times(1)).existsByEmail(validRequest.getEmail());

verify(accountRepository, times(1)).existsByCitizenId(validRequest.getCitizenId());

verify(accountRepository, never()).save(any(Account.class));

}

}

Từ 2 file test case trên sinh ra 2 file File 02: Bảng danh sách Test Case (Excel hoặc Markdown).

File 03: File dữ liệu kiểm thử mock_customers.csv.

Dưới đây là tài liệu đặc tả danh sách test case và file dữ liệu kiểm thử được trích xuất, chuẩn hóa từ 2 file Unit Test trên theo đúng chuẩn dự án nghiệp vụ.

File 02: Bảng danh sách Test Case (Markdown Format)
Bảng này hệ thống lại toàn bộ các kịch bản kiểm thử (Test Cases) ở cả 2 tầng Controller (API) và Service (Nghiệp vụ vụ/Database) dựa trên mã nguồn test bạn cung cấp.

Mã TC	Tầng Kiểm Thử	Chức Năng / API	Mô Tả Kịch Bản (Scenario)	Dữ Liệu Đầu Vào (Input)	Kết Quả Mong Đợi (Expected Outcome)
TC-01	Controller & Service	POST /api/v1/accounts/register	Đăng ký tài khoản với đầy đủ thông tin hợp lệ.	Đầy đủ dữ liệu chuẩn (Họ tên, SĐT VN, Email đúng định dạng, CCCD đúng tỉnh).	Trả về mã HTTP 201 Created. Phản hồi đầy đủ: accountId, accountNumber, status: "PENDING".
TC-04	Controller (Validation)	POST /api/v1/accounts/register	Kiểm tra tính bắt buộc của trường Họ và tên.	fullName: "" (Trống), các trường khác hợp lệ.	Trả về mã HTTP 400 Bad Request. Thông báo lỗi chi tiết: "Họ và tên không được để trống".
TC-07	Controller (Validation)	POST /api/v1/accounts/register	Kiểm tra định dạng số điện thoại nhà mạng Việt Nam.	phone: "0243123456" (Số máy bàn/Sai định dạng di động).	Trả về mã HTTP 400 Bad Request. Thông báo lỗi chi tiết: "Số điện thoại không đúng định dạng nhà mạng Việt Nam".
TC-10	Service (Business Rule)	Tầng Nghiệp Vụ	Kiểm tra trùng lặp Số điện thoại trong hệ thống.	phone trùng với một tài khoản đã tồn tại trong DB.	Hệ thống ném ra DuplicateResourceException kèm thông báo: "Số điện thoại này đã được đăng ký sử dụng." Không lưu DB.
TC-12	Controller (Validation)	POST /api/v1/accounts/register	Kiểm tra định dạng email hợp lệ.	email: "an.nguyen.gmail.com" (Thiếu ký tự @).	Trả về mã HTTP 400 Bad Request. Thông báo lỗi chi tiết: "Định dạng Email không hợp lệ".
TC-13	Service (Business Rule)	Tầng Nghiệp Vụ	Kiểm tra trùng lặp Địa chỉ Email trong hệ thống.	email trùng với một tài khoản đã tồn tại trong DB.	Hệ thống ném ra DuplicateResourceException kèm thông báo: "Địa chỉ email này đã được đăng ký sử dụng." Không lưu DB.
TC-18	Controller (Validation)	POST /api/v1/accounts/register	Kiểm tra cấu trúc 3 chữ số đầu (mã tỉnh) của số CCCD.	citizenId: "000095012345" (Mã tỉnh 000 không tồn tại).	Trả về mã HTTP 400 Bad Request. Thông báo lỗi chi tiết: "Mã tỉnh/thành phố (3 số đầu) trên CCCD không hợp lệ".
TC-20	Service (Business Rule)	Tầng Nghiệp Vụ	Kiểm tra trùng lặp số định danh cá nhân (CCCD).	citizenId trùng với một tài khoản đã tồn tại trong DB.	Hệ thống ném ra DuplicateResourceException kèm thông báo: "Số CCCD này đã được đăng ký cho một tài khoản khác." Không lưu DB.
File 03: File dữ liệu kiểm thử mock_customers.csv
File CSV này chứa tập dữ liệu đại diện tương ứng với các Test Cases phía trên để phục vụ cho việc thực hiện Data-Driven Testing (Kiểm thử hướng dữ liệu) hoặc import vào các công cụ như Postman / jMeter.

Đoạn mã
test_case_id,full_name,phone,email,citizen_id,expected_status,expected_error_field,expected_message
TC-01,Nguyen Van An,0325412365,an.nguyen@gmail.com,001095012345,210,,
TC-04,,0325412365,an.nguyen@gmail.com,001095012345,400,fullName,Họ và tên không được để trống
TC-07,Nguyen Van An,0243123456,an.nguyen@gmail.com,001095012345,400,phone,Số điện thoại không đúng định dạng nhà mạng Việt Nam
TC-10,Nguyen Van An,0325412365,an.nguyen@gmail.com,001095012345,400,phone,Số điện thoại này đã được đăng ký sử dụng.
TC-12,Nguyen Van An,0325412365,an.nguyen.gmail.com,001095012345,400,email,Định dạng Email không hợp lệ
TC-13,Nguyen Van An,0325412365,an.nguyen@gmail.com,001095012345,400,email,Địa chỉ email này đã được đăng ký sử dụng.
TC-18,Nguyen Van An,0325412365,an.nguyen@gmail.com,000095012345,400,citizenId,Mã tỉnh/thành phố (3 số đầu) trên CCCD không hợp lệ
TC-20,Nguyen Van An,0325412365,an.nguyen@gmail.com,001095012345,400,citizenId,Số CCCD này đã được đăng ký cho một tài khoản khác.
Lưu ý nghiệp vụ: > * Đối với TC-10, TC-13, TC-20, dữ liệu đầu vào nhìn giống hệt TC-01 nhưng kết quả mong đợi khác nhau là do trạng thái Database tại thời điểm test (giả định dữ liệu đã tồn tại trong DB, dẫn tới việc tầng Service ném lỗi trùng lặp).

Mã TCTầng Kiểm ThửChức Năng / APIMô Tả Kịch Bản (Scenario)Dữ Liệu Đầu Vào (Input)Kết Quả Mong Đợi (Expected Outcome)TC-01Controller & ServicePOST /api/v1/accounts/registerĐăng ký tài khoản với đầy đủ thông tin hợp lệ.Đầy đủ dữ liệu chuẩn (Họ tên, SĐT VN, Email đúng định dạng, CCCD đúng tỉnh).Trả về mã HTTP 201 Created. Phản hồi đầy đủ: accountId, accountNumber, status: "PENDING".TC-04Controller (Validation)POST /api/v1/accounts/registerKiểm tra tính bắt buộc của trường Họ và tên.fullName: "" (Trống), các trường khác hợp lệ.Trả về mã HTTP 400 Bad Request. Thông báo lỗi chi tiết: "Họ và tên không được để trống".TC-07Controller (Validation)POST /api/v1/accounts/registerKiểm tra định dạng số điện thoại nhà mạng Việt Nam.phone: "0243123456" (Số máy bàn/Sai định dạng di động).Trả về mã HTTP 400 Bad Request. Thông báo lỗi chi tiết: "Số điện thoại không đúng định dạng nhà mạng Việt Nam".TC-10Service (Business Rule)Tầng Nghiệp VụKiểm tra trùng lặp Số điện thoại trong hệ thống.phone trùng với một tài khoản đã tồn tại trong DB.Hệ thống ném ra DuplicateResourceException kèm thông báo: "Số điện thoại này đã được đăng ký sử dụng." Không lưu DB.TC-12Controller (Validation)POST /api/v1/accounts/registerKiểm tra định dạng email hợp lệ.email: "an.nguyen.gmail.com" (Thiếu ký tự @).Trả về mã HTTP 400 Bad Request. Thông báo lỗi chi tiết: "Định dạng Email không hợp lệ".TC-13Service (Business Rule)Tầng Nghiệp VụKiểm tra trùng lặp Địa chỉ Email trong hệ thống.email trùng với một tài khoản đã tồn tại trong DB.Hệ thống ném ra DuplicateResourceException kèm thông báo: "Địa chỉ email này đã được đăng ký sử dụng." Không lưu DB.TC-18Controller (Validation)POST /api/v1/accounts/registerKiểm tra cấu trúc 3 chữ số đầu (mã tỉnh) của số CCCD.citizenId: "000095012345" (Mã tỉnh 000 không tồn tại).Trả về mã HTTP 400 Bad Request. Thông báo lỗi chi tiết: "Mã tỉnh/thành phố (3 số đầu) trên CCCD không hợp lệ".TC-20Service (Business Rule)Tầng Nghiệp VụKiểm tra trùng lặp số định danh cá nhân (CCCD).citizenId trùng với một tài khoản đã tồn tại trong DB.Hệ thống ném ra DuplicateResourceException kèm thông báo: "Số CCCD này đã được đăng ký cho một tài khoản khác." Không lưu DB. chuyển doc trên sang file md

"""

Ghi nội dung ra file test_cases.md
file_path = "test_cases.md"
with open(file_path, "w", encoding="utf-8") as f:
f.write(markdown_content)

print(f"File created successfully at {file_path}")

File Markdown chứa bảng danh sách Test Case và dữ liệu kiểm thử của bạn đã được khởi tạo thành công.

Your Markdown file is ready
[file-tag: code-generated-file-0-1782869543638213488]

### Tóm tắt nội dung file `.md` vừa tạo:
* **Cấu trúc chuẩn hóa:** Chuyển đổi dữ liệu thô dạng text ghép của bạn thành định dạng bảng Markdown (`| Column |`) trực quan, dễ đọc, dễ tích hợp vào tài liệu dự án trên Gitlab/Github.
* **Bao gồm mã mẫu CSV:** Đã đính kèm phần mã nguồn dữ liệu `mock_customers.csv` ở cuối file dưới dạng khối mã lệnh (code block) để bạn có thể sao chép nhanh khi cấu hình công cụ test tự động.
chuyển file 2 ở trên sang định dạng md

Your Markdown file is ready
Biểu tượng MD
file_02_test_cases
 MD 
Đang mở

Dưới đây là nội dung chi tiết của File 02: Bảng danh sách Test Case đã được chuyển đổi sang định dạng Markdown chuẩn chỉnh để bạn có thể xem trực tiếp hoặc sao chép:

Bảng Danh Sách Test Case (Test Case Specification)
Tài liệu này hệ thống lại toàn bộ các kịch bản kiểm thử (Test Cases) ở cả tầng Controller (Kiểm thử dữ liệu Validation đầu vào) và tầng Service (Kiểm thử quy tắc nghiệp vụ Business Logic) được trích xuất từ mã nguồn kiểm thử của hệ thống đăng ký tài khoản (AccountController và AccountServiceImpl).

Danh Sách Các Kịch Bản Kiểm Thử
Mã TC	Tầng Kiểm Thử	Chức Năng / API	Mô Tả Kịch Bản (Scenario)	Dữ Liệu Đầu Vào (Input)	Kết Quả Mong Đợi (Expected Outcome)
TC-01	Controller & Service	POST /api/v1/accounts/register	Đăng ký tài khoản với đầy đủ thông tin hợp lệ.	Đầy đủ dữ liệu chuẩn (Họ tên, SĐT VN, Email đúng định dạng, CCCD đúng tỉnh).	Trả về mã HTTP 201 Created. Phản hồi đầy đủ cấu trúc JSON: accountId, accountNumber, status: "PENDING".
TC-04	Controller (Validation)	POST /api/v1/accounts/register	Kiểm tra tính bắt buộc của trường Họ và tên.	fullName: "" (Trống), các trường khác hợp lệ.	Trả về mã HTTP 400 Bad Request. Thông báo lỗi chi tiết: "Họ và tên không được để trống".
TC-07	Controller (Validation)	POST /api/v1/accounts/register	Kiểm tra định dạng số điện thoại nhà mạng Việt Nam.	phone: "0243123456" (Số máy bàn hoặc sai định dạng đầu số di động).	Trả về mã HTTP 400 Bad Request. Thông báo lỗi chi tiết: "Số điện thoại không đúng định dạng nhà mạng Việt Nam".
TC-10	Service (Business Rule)	Tầng Nghiệp Vụ (Service)	Kiểm tra quy tắc không cho phép trùng lặp Số điện thoại.	phone trùng với một tài khoản khách hàng đã tồn tại trong Database.	Hệ thống ném ra DuplicateResourceException kèm thông báo: "Số điện thoại này đã được đăng ký sử dụng." Không thực hiện lưu vào DB.
TC-12	Controller (Validation)	POST /api/v1/accounts/register	Kiểm tra định dạng email hợp lệ theo cấu trúc chuẩn.	email: "an.nguyen.gmail.com" (Chuỗi ký tự thiếu ký tự bắt buộc @).	Trả về mã HTTP 400 Bad Request. Thông báo lỗi chi tiết: "Định dạng Email không hợp lệ".
TC-13	Service (Business Rule)	Tầng Nghiệp Vụ (Service)	Kiểm tra quy tắc không cho phép trùng lặp Địa chỉ Email.	email trùng với một tài khoản khách hàng đã tồn tại trong Database.	Hệ thống ném ra DuplicateResourceException kèm thông báo: "Địa chỉ email này đã được đăng ký sử dụng." Không thực hiện lưu vào DB.
TC-18	Controller (Validation)	POST /api/v1/accounts/register	Kiểm tra cấu trúc pháp lý 3 chữ số đầu (mã tỉnh) của số CCCD.	citizenId: "000095012345" (Cấu trúc hợp lệ nhưng mã tỉnh 000 không tồn tại).	Trả về mã HTTP 400 Bad Request. Thông báo lỗi chi tiết: "Mã tỉnh/thành phố (3 số đầu) trên CCCD không hợp lệ".
TC-20	Service (Business Rule)	Tầng Nghiệp Vụ (Service)	Kiểm tra quy tắc không cho phép trùng lặp số định danh (CCCD).	citizenId trùng với một số định danh đã tồn tại trong Database.	Hệ thống ném ra DuplicateResourceException kèm thông báo: "Số CCCD này đã được đăng ký cho một tài khoản khác." Không thực hiện lưu vào DB.



Gemini là AI và có thể mắc sai sót.

# Bảng Danh Sách Test Case (Test Case Specification)

Tài liệu này hệ thống lại toàn bộ các kịch bản kiểm thử (Test Cases) ở cả tầng Controller (Kiểm thử dữ liệu Validation đầu vào) và tầng Service (Kiểm thử quy tắc nghiệp vụ Business Logic) được trích xuất từ mã nguồn kiểm thử của hệ thống đăng ký tài khoản (`AccountController` và `AccountServiceImpl`).

## Danh Sách Các Kịch Bản Kiểm Thử

| Mã TC | Tầng Kiểm Thử | Chức Năng / API | Mô Tả Kịch Bản (Scenario) | Dữ Liệu Đầu Vào (Input) | Kết Quả Mong Đợi (Expected Outcome) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **TC-01** | Controller & Service | `POST /api/v1/accounts/register` | Đăng ký tài khoản với đầy đủ thông tin hợp lệ. | Đầy đủ dữ liệu chuẩn (Họ tên, SĐT VN, Email đúng định dạng, CCCD đúng tỉnh). | Trả về mã HTTP `201 Created`. Phản hồi đầy đủ cấu trúc JSON: `accountId`, `accountNumber`, `status: "PENDING"`. |
| **TC-04** | Controller (Validation) | `POST /api/v1/accounts/register` | Kiểm tra tính bắt buộc của trường Họ và tên. | `fullName: ""` (Trống), các trường khác hợp lệ. | Trả về mã HTTP `400 Bad Request`. Thông báo lỗi chi tiết: `"Họ và tên không được để trống"`. |
| **TC-07** | Controller (Validation) | `POST /api/v1/accounts/register` | Kiểm tra định dạng số điện thoại nhà mạng Việt Nam. | `phone: "0243123456"` (Số máy bàn hoặc sai định dạng đầu số di động). | Trả về mã HTTP `400 Bad Request`. Thông báo lỗi chi tiết: `"Số điện thoại không đúng định dạng nhà mạng Việt Nam"`. |
| **TC-10** | Service (Business Rule) | Tầng Nghiệp Vụ (Service) | Kiểm tra quy tắc không cho phép trùng lặp Số điện thoại. | `phone` trùng với một tài khoản khách hàng đã tồn tại trong Database. | Hệ thống ném ra `DuplicateResourceException` kèm thông báo: `"Số điện thoại này đã được đăng ký sử dụng."` Không thực hiện lưu vào DB. |
| **TC-12** | Controller (Validation) | `POST /api/v1/accounts/register` | Kiểm tra định dạng email hợp lệ theo cấu trúc chuẩn. | `email: "an.nguyen.gmail.com"` (Chuỗi ký tự thiếu ký tự bắt buộc `@`). | Trả về mã HTTP `400 Bad Request`. Thông báo lỗi chi tiết: `"Định dạng Email không hợp lệ"`. |
| **TC-13** | Service (Business Rule) | Tầng Nghiệp Vụ (Service) | Kiểm tra quy tắc không cho phép trùng lặp Địa chỉ Email. | `email` trùng với một tài khoản khách hàng đã tồn tại trong Database. | Hệ thống ném ra `DuplicateResourceException` kèm thông báo: `"Địa chỉ email này đã được đăng ký sử dụng."` Không thực hiện lưu vào DB. |
| **TC-18** | Controller (Validation) | `POST /api/v1/accounts/register` | Kiểm tra cấu trúc pháp lý 3 chữ số đầu (mã tỉnh) của số CCCD. | `citizenId: "000095012345"` (Cấu trúc hợp lệ nhưng mã tỉnh `000` không tồn tại). | Trả về mã HTTP `400 Bad Request`. Thông báo lỗi chi tiết: `"Mã tỉnh/thành phố (3 số đầu) trên CCCD không hợp lệ"`. |
| **TC-20** | Service (Business Rule) | Tầng Nghiệp Vụ (Service) | Kiểm tra quy tắc không cho phép trùng lặp số định danh (CCCD). | `citizenId` trùng với một số định danh đã tồn tại trong Database. | Hệ thống ném ra `DuplicateResourceException` kèm thông báo: `"Số CCCD này đã được đăng ký cho một tài khoản khác."` Không thực hiện lưu vào DB. |
file_02_test_cases.md
Đang hiển thị file_02_test_cases.md.
