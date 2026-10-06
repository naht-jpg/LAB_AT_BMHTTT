# LAB 5 - THIẾT LẬP MÔ HÌNH TƯỜNG LỬA pfSense

## Thông tin sinh viên

-   **Họ và tên:** Trần Minh Nhật
-   **MSSV:** 1150080068
-   **Lớp:** 11CNPM1
-   **Ngày thực hiện:** 06/10/2026

------------------------------------------------------------------------

## 1. Mục tiêu

- Xây dựng mô hình mạng có tường lửa pfSense bảo vệ mạng LAN và vùng DMZ.
- Cấu hình các interface WAN, LAN, DMZ, Outbound NAT và Firewall Rules.
- Sử dụng Windows Server làm Domain Controller trong mạng LAN để kiểm thử lưu lượng.
- Thực hành các tình huống firewall: chặn ICMP, giới hạn host được truy cập Internet, cô lập DMZ khỏi LAN, Port Forward WAN vào DMZ và kiểm tra Firewall Log.

## 2. Môi trường thực hành

- **Phần mềm ảo hóa:** VMware Workstation.
- **Firewall:** pfSense CE 2.7.2-RELEASE (amd64).
- **Domain Controller:** Windows Server 2022.
- **LAN-Test:** Ubuntu Server.
- **DMZ-Web:** Windows Server + IIS.

### Mô hình địa chỉ IP

| Thiết bị | Interface | Địa chỉ IP | Gateway | DNS |
|---|---|---|---|---|
| pfSense | WAN | DHCP | Upstream | Upstream |
| pfSense | LAN | 10.0.0.1/8 | - | - |
| pfSense | DMZ | 172.16.0.1/16 | - | - |
| Domain Controller | LAN | 10.0.0.2/8 | 10.0.0.1 | 10.0.0.2 |
| LAN-Test | LAN | 10.0.0.3/8 | 10.0.0.1 | - |
| DMZ-Web | DMZ | 172.16.0.2/16 | 172.16.0.1 | 8.8.8.8 |

## 3. Nội dung thực hiện

### 3.1. Cài đặt và cấu hình pfSense

- Tạo máy ảo pfSense trên VMware.
- Cấu hình 3 card mạng:
  - WAN: Bridged.
  - LAN: Host-only, mạng 10.0.0.0/8.
  - DMZ: mạng riêng 172.16.0.0/16.
- Cấu hình LAN của pfSense là `10.0.0.1/8`.
- Cấu hình DMZ là `172.16.0.1/16`.
- Kiểm tra Dashboard và trạng thái các interface.
- Kiểm tra Automatic/Hybrid Outbound NAT cho LAN và DMZ.

### 3.2. Cấu hình Domain Controller

- Đặt IP tĩnh `10.0.0.2/8`.
- Default Gateway: `10.0.0.1`.
- Preferred DNS: `10.0.0.2`.
- Cài đặt Active Directory Domain Services và DNS.
- Cấu hình DNS Forwarder sử dụng `8.8.8.8`.
- Kiểm tra kết nối tới pfSense, Internet và phân giải DNS.

### 3.3. Chuẩn hóa Firewall Rules trên LAN

- Disable các rule `Default allow LAN to any` IPv4 và IPv6.
- Giữ nguyên `Anti-Lockout Rule`.
- Tạo rule nền tảng `Pass LAN subnets -> Any`.
- Thực hiện Reset States sau khi thay đổi ruleset để bảo đảm kết quả kiểm thử không bị ảnh hưởng bởi state cũ.

### 3.4. Tình huống 1 - Chặn ICMP nhưng vẫn cho phép Web/DNS

- Tạo rule Block ICMP từ LAN.
- Tạo rule Pass DNS TCP/UDP port 53.
- Tạo rule Pass HTTP/HTTPS TCP port 80/443.
- Kết quả: ping ra Internet bị chặn nhưng truy vấn DNS và truy cập HTTPS vẫn hoạt động.

### 3.5. Tình huống 2 - Chỉ cho một host cụ thể ra Internet

- Tạo LAN-Test với IP `10.0.0.3/8`.
- Tạo rule `Pass 10.0.0.2 -> Any`.
- Tạo rule `Block LAN subnets -> Any`.
- Kết quả: Domain Controller `10.0.0.2` truy cập Internet được, trong khi LAN-Test `10.0.0.3` bị chặn.

### 3.6. Tình huống 3 - Cô lập DMZ khỏi LAN

- Tạo DMZ-Web với IP `172.16.0.2/16`.
- Tạo baseline bằng rule `Pass DMZ subnets -> Any` và xác nhận DMZ-Web truy cập được Domain Controller.
- Thêm rule `Block DMZ subnets -> LAN subnets` phía trên rule Pass.
- Kết quả: DMZ-Web không còn truy cập được LAN nhưng vẫn có thể truy cập Internet.

### 3.7. Tình huống 4 - Port Forward WAN vào DMZ

- Cài IIS trên DMZ-Web.
- Kiểm tra dịch vụ Web tại `172.16.0.2:80`.
- Tạo Port Forward:
  - WAN port `8080`.
  - Redirect tới `172.16.0.2:80`.
- Kết quả: từ máy thật có thể truy cập trang IIS qua địa chỉ WAN của pfSense tại port `8080`.

### 3.8. Tình huống 5 - Firewall Logging

- Bật logging cho rule Block DMZ -> LAN.
- Tạo lưu lượng bị chặn.
- Kiểm tra tại `Status -> System Logs -> Firewall`.
- Kết quả: log hiển thị các gói tin bị chặn và rule tương ứng.

## 4. Kết quả thực hiện

- Cấu hình thành công mô hình pfSense với WAN, LAN và DMZ.
- Domain Controller truy cập Internet và phân giải DNS thông qua pfSense.
- Outbound NAT hoạt động cho LAN và DMZ.
- Các tình huống firewall cho kết quả đúng theo ruleset đã cấu hình.
- Port Forward WAN -> DMZ hoạt động và truy cập được IIS từ máy thật.
- Firewall Log ghi nhận được các gói tin bị chặn.

## 5. Lưu ý

- pfSense CE 2.7.2 được sử dụng cho môi trường lab, không dùng cho hệ thống production.
- Sau mỗi lần thay đổi Firewall Rule hoặc NAT cần Apply Changes và Reset States trước khi kiểm thử lại.
- Rule trong pfSense được xử lý theo thứ tự từ trên xuống, vì vậy các rule Block cụ thể phải được đặt trước các rule Pass tổng quát.
- Các máy ảo phải được gắn đúng VMnet tương ứng với LAN hoặc DMZ để tránh sai kết quả kiểm thử.
