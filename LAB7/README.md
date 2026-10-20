# LAB 7 - XÂY DỰNG CHÍNH SÁCH WEB PROTECTION TRÊN MÔ HÌNH pfSense + WEB PROXY

## Thông tin sinh viên

-   **Họ và tên:** Trần Minh Nhật
-   **MSSV:** 1150080068
-   **Lớp:** 11CNPM1
-   **Ngày thực hiện:** 20/10/2026

------------------------------------------------------------------------

## 1. Mục tiêu

- Kế thừa mô hình LAB 6 và bổ sung máy WEB-PROXY chạy Squid, E2Guardian, ClamAV; không cài Squid/SquidGuard trực tiếp trên pfSense.
- Dùng pfSense ép toàn bộ lưu lượng web của Client-A/B đi qua proxy và chặn truy cập web trực tiếp để chống bypass.
- Xây dựng chính sách Web Protection theo tên miền: hard block, cảnh báo kèm bypass tạm, exception, giới hạn theo khung giờ.
- Quét virus khi tải file bằng ClamAV và giới hạn kích thước tải về lớn hơn 100 MB.
- Kiểm thử 10 kịch bản và đối chiếu log E2Guardian, Squid và Firewall Log của pfSense.

## 2. Môi trường thực hành

- **Phần mềm ảo hóa:** VMware Workstation.
- **Firewall:** pfSense CE (máy pfSenseLab6 dùng lại từ LAB 6).
- **Domain Controller/DNS:** Windows Server 2022, domain `lab6.test`, bổ sung zone `lab7.test`.
- **Client-A, Client-B:** Windows 11, dùng proxy `10.6.0.30:8080`.
- **WEB-PROXY:** Ubuntu Server 26.04, 2 GB RAM, 2 vCPU, 20 GB disk; E2Guardian 5.5.9, Squid 7.2, ClamAV 1.5.4.
- **DMZ-Web:** Ubuntu Server 26.04, Nginx với 13 website mô phỏng `*.lab7.test`.

### Mô hình địa chỉ IP

| Thiết bị | Interface / mạng | Địa chỉ IP | Gateway | DNS |
|---|---|---|---|---|
| pfSense | WAN / VMnet5 | DHCP 192.168.250.0/24 | Upstream NAT | Upstream |
| pfSense | MGMT / VMnet2 | 192.168.56.254/24 | - | - |
| Máy thật quản trị | VMnet2 | 192.168.56.1/24 | - | - |
| pfSense | LAN / VMnet3 | 10.6.0.1/24 | - | - |
| DC01 | LAN / VMnet3 | 10.6.0.10/24 | 10.6.0.1 | 10.6.0.10 |
| Client-A | LAN / VMnet3 | 10.6.0.21/24 | 10.6.0.1 | 10.6.0.10 |
| Client-B | LAN / VMnet3 | 10.6.0.22/24 | 10.6.0.1 | 10.6.0.10 |
| WEB-PROXY | LAN / VMnet3 | 10.6.0.30/24 | 10.6.0.1 | 10.6.0.10 |
| pfSense | DMZ / VMnet4 | 172.16.6.1/24 | - | - |
| Ubuntu DMZ-Web | DMZ / VMnet4 | 172.16.6.10/24 (thêm .20, .30, .40 từ LAB 6 và .50 cho LAB 7) | 172.16.6.1 | Theo cấu hình thử nghiệm |

**Luồng web:** Client → E2Guardian `10.6.0.30:8080` → Squid `127.0.0.1:3128` → pfSense → DMZ-Web/Internet.

## 3. Nội dung thực hiện

### 3.1. Chuẩn bị môi trường

- Backup cấu hình pfSense và tạo snapshot `LAB7_START` trước khi thay đổi.
- Tắt Captive Portal `LAB6_USERS` vì portal chặn WEB-PROXY và DC01 ra Internet.
- Disable (không xóa) các rule LAB 6 và rule Default allow LAN trên interface LAN.
- Tạo VM WEB-PROXY trên VMnet3, đặt IP tĩnh `10.6.0.30/24`.

### 3.2. Cấu hình Squid

- Đổi `http_port` thành `127.0.0.1:3128` để client không thể đi thẳng vào Squid.
- Tạo `/etc/squid/conf.d/lab7.conf` với `reply_body_max_size 100 MB all`.
- Thêm `dns_nameservers 10.6.0.10` để khắc phục lỗi `ERR_DNS_FAIL` của Squid trên Ubuntu 26.04.

### 3.3. Cấu hình E2Guardian và ClamAV

- Trong `e2guardian.conf`: bật `proxyip = 127.0.0.1`, `proxyport = 3128`, `loglocation`, `contentscanner` (clamdscan); `filterip`/`filterports` để mặc định (nghe `0.0.0.0:8080`).
- Trong `e2guardianf1.conf`: `bypass = 300`, `contentscanexceptions = on`, `htmltemplate = 'lab7_block.html'`.
- Danh sách trong `/etc/e2guardian/lists/example.group/`:
  - `bannedsitelist`: china, mideast, jobs, blockedgame, weapons, politics, religion, streaming.
  - `domainsnobypass`: china, mideast, jobs, blockedgame.
  - `exceptionsitelist`: normal, education, download.
- Bổ sung các biến `-URL-`, `-REASONGIVEN-`, `-BYPASS-` vào trang chặn tùy biến.
- Tải chữ ký virus bằng `freshclam`, thêm user `clamav` vào nhóm `e2guardian` để clamd đọc được file tạm.

### 3.4. Chuẩn bị website mô phỏng và DNS

- Chạy `setup_lab7_dmz_web.sh` trên DMZ-Web để tạo 13 website và các file kiểm thử `eicar.com.txt`, `10MB.bin`, `100MB.bin`, `110MB.bin`.
- Gán thêm IP `172.16.6.50` cho `entertainment` và `games` để pfSense áp được Schedule riêng.
- Tạo zone `lab7.test` trên DC01 với 13 bản ghi A: 11 bản ghi trỏ `172.16.6.10`, `entertainment` và `games` trỏ `172.16.6.50`.

### 3.5. Cấu hình pfSense ép lưu lượng web qua proxy

- Tạo các Alias: `LAB7_CLIENTS`, `LAB7_PROXY`, `LAB7_DC01`, `LAB7_ENT_SITE`, `MAIL_PORTS`; dùng lại `WEB_PORTS`.
- Tạo Schedule `LAB7_ENT_60M` (khung 60 phút).
- Ruleset trên LAN, xét theo thứ tự từ trên xuống:
  1. Pass DNS về DC01, Pass DNS từ DC01 ra ngoài.
  2. Pass `LAB7_CLIENTS → LAB7_PROXY` TCP/8080; Pass email `MAIL_PORTS`.
  3. Block `LAB7_CLIENTS → any` `WEB_PORTS` (chống bypass, có log).
  4. Pass `LAB7_PROXY → LAB7_ENT_SITE` TCP/80 với Schedule `LAB7_ENT_60M`; Reject cùng luồng ngay phía dưới.
  5. Pass `LAB7_PROXY → any` `WEB_PORTS` và `LAB7_PROXY → 172.16.6.10` TCP/80.
  6. Block `LAN net → any` (explicit deny, có log).

### 3.6. Kiểm thử theo kịch bản

| Kịch bản | Đích kiểm thử | Kết quả |
|---|---|---|
| 1. Baseline | `normal.lab7.test` | Qua proxy truy cập được; tắt proxy thì bị rule chống bypass chặn (Client-A và Client-B) |
| 2. Geo-Sim | `china`, `mideast` | Hard block, không bypass được |
| 3. Job Search | `jobs` | Hard block |
| 4. Warning | `weapons`, `politics`, `religion` | Trang cảnh báo, bypass tạm 5 phút, hết hạn bị cảnh báo lại |
| 5. Education | `education` | Truy cập bình thường, không cảnh báo |
| 6. Entertainment/Game | `entertainment`, `games` | Trong khung giờ truy cập được; ngoài khung bị Reject |
| 7. Streaming | `streaming` | Cảnh báo, bypass tạm được |
| 8. Specific game | `blockedgame` | Hard block |
| 9. Antivirus | `download.lab7.test/files/eicar.com.txt` | ClamAV phát hiện `Eicar-Signature`, bị chặn |
| 10. Size | `100MB.bin`, `110MB.bin` | 100MB tải được; 110MB bị Squid từ chối 403 |

## 4. Kết quả thực hiện

- Hoàn thành WEB-PROXY với luồng E2Guardian → Squid → pfSense; Squid chỉ nghe localhost.
- pfSense chặn truy cập web trực tiếp từ Client-A/B; chỉ WEB-PROXY được phép ra HTTP/HTTPS.
- Cả 10 kịch bản đều cho kết quả đúng như yêu cầu, có ảnh minh chứng và log E2Guardian, Squid, Firewall Log trong báo cáo.
- Trả lời đủ 4 câu hỏi của mẫu báo cáo và 10 câu hỏi phân tích của đề.
- **HTTPS inspection:** phần mở rộng, không thực hiện.

## 5. Lưu ý

- Một số điểm khác tài liệu hướng dẫn do môi trường thực tế:
  - Ubuntu 26.04 dùng E2Guardian 5.5.9: danh sách nằm trong `lists/example.group/`, không cần sửa `site.story`.
  - Đặt `contentscanexceptions = on` (tài liệu ghi `off`), nếu không file EICAR trên `download.lab7.test` sẽ không bị quét.
  - Dòng `reply_body_max_size 104857600 allow all` trong file mẫu sai cú pháp; dùng `reply_body_max_size 100 MB all`.
  - `custom_block_template.html` của bộ tài nguyên thiếu biến `-BYPASS-`, cần bổ sung thì trang cảnh báo mới có link bypass.
- Schedule 60 phút là time-window policy, không tương đương daily category quota của Sophos.
- Firewall Log của pfSense hiển thị giờ UTC, lệch 7 giờ so với giờ Việt Nam trên client.
- Các rule pfSense xét theo thứ tự từ trên xuống (first-match); khi thay rule/Schedule phải Apply Changes và Reset States trước khi kiểm thử lại.
- Báo cáo Word, video minh chứng và bản sao cấu hình pfSense được quản lý riêng trong thư mục LAB7.
- Không đưa private key của CA hoặc credential trong `config.xml` lên repository Public.
