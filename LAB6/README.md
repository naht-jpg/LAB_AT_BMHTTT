# LAB 6 - XÁC THỰC ACTIVE DIRECTORY VÀ CHÍNH SÁCH TRUY CẬP TRÊN pfSense

## Thông tin sinh viên

-   **Họ và tên:** Trần Minh Nhật
-   **MSSV:** 1150080068
-   **Lớp:** 11CNPM1
-   **Ngày thực hiện:** 13/10/2026

------------------------------------------------------------------------

## 1. Mục tiêu

- Xây dựng mô hình mạng riêng cho LAB 6 bằng pfSense với bốn interface WAN, MGMT, LAN và DMZ.
- Cấu hình Windows Server 2022 làm Domain Controller và DNS cho domain `lab6.test`.
- Tích hợp Active Directory với pfSense qua LDAP và xác thực các tài khoản `sv_a`, `sv_b`.
- Thiết lập Captive Portal trên LAN, quản trị pfSense qua mạng MGMT riêng.
- Tạo Firewall Alias và Schedule, áp dụng và kiểm thử các chính sách truy cập theo kịch bản 1–5.
- Kiểm tra khả năng cô lập DMZ khỏi LAN và ghi nhận Firewall Log.

## 2. Môi trường thực hành

- **Phần mềm ảo hóa:** VMware Workstation.
- **Firewall:** pfSense CE 2.7.2-RELEASE (amd64).
- **Domain Controller/DNS:** Windows Server 2022, domain `lab6.test`.
- **Client-A:** Windows 11, tài khoản LDAP kiểm thử `sv_a`.
- **Client-B:** Windows 11, tài khoản LDAP kiểm thử `sv_b`.
- **DMZ-Web:** Ubuntu Server 26.04, dịch vụ Nginx.

### Mô hình địa chỉ IP

| Thiết bị | Interface / mạng | Địa chỉ IP | Gateway | DNS |
|---|---|---|---|---|
| pfSense | WAN / VMnet5 | 192.168.250.128/24 (DHCP tại thời điểm thực hành) | Upstream NAT | Upstream |
| pfSense | MGMT / VMnet2 | 192.168.56.254/24 | - | - |
| Máy thật quản trị | VMnet2 | 192.168.56.1/24 | - | - |
| pfSense | LAN / VMnet3 | 10.6.0.1/24 | - | - |
| DC01 | LAN / VMnet3 | 10.6.0.10/24 | 10.6.0.1 | 10.6.0.10 |
| Client-A | LAN / VMnet3 | 10.6.0.21/24 | 10.6.0.1 | 10.6.0.10 |
| Client-B | LAN / VMnet3 | 10.6.0.22/24 | 10.6.0.1 | 10.6.0.10 |
| pfSense | DMZ / VMnet4 | 172.16.6.1/24 | - | - |
| Ubuntu DMZ-Web | DMZ / VMnet4 | 172.16.6.10/24 (thêm .20, .30, .40) | 172.16.6.1 | Theo cấu hình thử nghiệm |

## 3. Nội dung thực hiện

### 3.1. Thiết lập mô hình mạng và pfSense

- Tạo các mạng ảo VMnet5 (WAN/NAT), VMnet2 (MGMT), VMnet3 (LAN), VMnet4 (DMZ).
- Gán bốn Network Adapter cho pfSense và đặt IP cho từng interface.
- Truy cập WebGUI qua `https://192.168.56.254` từ máy thật.
- Sử dụng Automatic Outbound NAT cho các mạng nội bộ.

### 3.2. Cấu hình Active Directory và DNS

- Đặt IP tĩnh `10.6.0.10/24` cho Windows Server 2022.
- Cài đặt AD DS và DNS, tạo domain `lab6.test`, OU `LAB6-Users`.
- Tạo các tài khoản `sv_a`, `sv_b` và tài khoản dịch vụ LDAP `svc_pfsense_ldap`.
- Cấu hình DNS Forwarders và kiểm tra phân giải tên miền nội bộ.

### 3.3. Chuẩn bị website mô phỏng trong DMZ

- Cài đặt Nginx trên Ubuntu Server tại `172.16.6.10`.
- Gán ba IP phụ `172.16.6.20`, `172.16.6.30`, `172.16.6.40`.
- Kiểm tra phản hồi HTTP 200 từ cả bốn địa chỉ IP.
- Sử dụng các tên logic: `GENERAL_SITE`, `JOB_SITE`, `SOCIAL_SITE`, `RESTRICTED_SITE`.

### 3.4. Tích hợp LDAP và Captive Portal

- Tạo LDAP Authentication Server `LAB6-AD`, trỏ đến DC01 `10.6.0.10:389`.
- Kiểm thử xác thực các tài khoản `sv_a`, `sv_b` bằng **Diagnostics → Authentication**.
- Tạo Captive Portal Zone `LAB6_USERS` trên interface LAN và sử dụng nguồn xác thực LDAP.
- Kiểm tra giao diện đăng nhập Captive Portal từ client.

### 3.5. Tạo Firewall Alias và Schedule

- Tạo các Alias: `CLIENT_A`, `CLIENT_B`, `POLICY_CLIENTS`, `GENERAL_SITE`, `JOB_SITE`, `SOCIAL_SITE`, `RESTRICTED_SITE`.
- Tạo Port Alias `WEB_PORTS` gồm TCP/80 và TCP/443.
- Tạo các Schedule `LUNCH_WINDOW`, `WORK_HOURS` và `TEST_NOW` để kiểm thử chính sách theo thời gian.

### 3.6. Kịch bản 1 - Chặn Client-A truy cập JOB_SITE

- Tạo rule Block `CLIENT_A → JOB_SITE` trên TCP/80.
- Cho phép Client-A truy cập `GENERAL_SITE`.
- Kết quả trong báo cáo: `GENERAL_SITE` trả HTTP 200; `JOB_SITE` bị timeout.

### 3.7. Kịch bản 2 - Chặn RESTRICTED_SITE cho hai client

- Tạo rule Block `POLICY_CLIENTS → RESTRICTED_SITE` trên TCP/80.
- Giữ rule Pass đến `GENERAL_SITE`.
- Kiểm thử truy cập bị chặn tại `172.16.6.40`, trong khi `172.16.6.10` vẫn phản hồi HTTP 200.

### 3.8. Kịch bản 3 - Truy cập SOCIAL_SITE theo giờ nghỉ

- Tạo rule Pass `CLIENT_B → SOCIAL_SITE` với Schedule `LUNCH_WINDOW`.
- Đặt rule Block tương ứng ngay phía dưới.
- Sử dụng tạm `TEST_NOW` để thử khi lịch có hiệu lực và không có hiệu lực, sau đó khôi phục `LUNCH_WINDOW`.

### 3.9. Kịch bản 4 - Chỉ cho phép HTTP/HTTPS theo giờ hành chính

- Tạo rule Pass `POLICY_CLIENTS → Any`, cổng `WEB_PORTS`, Schedule `WORK_HOURS`.
- Đặt rule Block các lưu lượng còn lại phía dưới.
- Thử bằng `TEST_NOW` và `Test-NetConnection` tới địa chỉ IP `1.1.1.1` trên TCP/80, TCP/443; ghi nhận kết quả trước và sau khi rule cho phép không còn hiệu lực.
- **Giới hạn kết quả:** DNS ngoài Internet qua DC01 bị lỗi timeout khi phân giải `example.com`, do đó phép thử TCP qua IP không được diễn giải là truy cập website theo tên miền thành công.

### 3.10. Kịch bản 5 - Cô lập DMZ khỏi LAN

- Tạo rule Block `OPT2 net → LAN net` trên interface DMZ/OPT2.
- Cho phép LAN truy cập `GENERAL_SITE` tại `172.16.6.10:80`.
- Kết quả: HTTP từ LAN đến web server thành công; Ubuntu DMZ-WEB ping DC01 `10.6.0.10` mất 100% gói.
- Đối chiếu Firewall Log của rule Block DMZ → LAN.

## 4. Kết quả thực hiện

- Hoàn thành mô hình pfSense với bốn interface và mạng MGMT quản trị riêng.
- Cấu hình Active Directory/DNS nội bộ và xác thực LDAP trên pfSense.
- Triển khai Captive Portal, hệ thống Alias, Schedule và các Firewall Rules tương ứng kịch bản 1–5.
- Nginx trên Ubuntu phục vụ được bốn địa chỉ web mô phỏng; các phép thử Allow/Block chính có ảnh minh chứng trong báo cáo.
- Ghi nhận việc cô lập DMZ và các bản ghi Firewall Log.
- DNS nội bộ `lab6.test` phân giải được, nhưng DNS ngoài Internet qua DC01 còn gặp timeout trong quá trình kiểm thử kịch bản 4.
- **Kịch bản 6 và 7:** không yêu cầu thực hiện theo hướng dẫn của giảng viên.

## 5. Lưu ý

- Các rule pfSense xét theo thứ tự từ trên xuống (first-match); đặt rule Block cụ thể trên rule Pass rộng.
- Khi thay rule/Schedule, phải Apply Changes và tạo kết nối kiểm thử mới; chỉ xóa state liên quan khi cần.
- LDAP TCP/389 chỉ phù hợp với môi trường LAB cô lập; khi triển khai thực tế nên dùng kết nối mã hóa.
- Báo cáo Word, video minh chứng và bản sao cấu hình pfSense được quản lý riêng trong thư mục LAB6.
- Kiểm tra, loại bỏ thông tin nhạy cảm trong `config.xml` trước khi đưa lên repository Public; bản backup cấu hình đầy đủ nên được giữ ở nơi riêng tư nếu chứa credential/secret.
