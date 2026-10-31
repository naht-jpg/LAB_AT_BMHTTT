# LAB 8 - LOCAL DNS ATTACK: DNS SPOOFING VÀ CACHE POISONING TRONG MẠNG ẢO CÔ LẬP

## Thông tin sinh viên

-   **Họ và tên:** Trần Minh Nhật
-   **MSSV:** 1150080068
-   **Lớp:** 11CNPM1
-   **Ngày thực hiện:** 31/10/2026

------------------------------------------------------------------------

## 1. Mục tiêu

- Dựng một mạng DNS-LAB `10.9.0.0/24` cô lập hoàn toàn trên VMnet6, tách khỏi mạng LAN-DOMAIN và MGMT của các lab trước; pfSense đóng vai gateway và ép ma trận cô lập bằng firewall rule.
- Triển khai DNS-SERVER chạy BIND9 vừa authoritative cho zone `lab9.test`, vừa recursive/caching cho mạng `10.9.0.0/24`; AUTH-DNS trên DMZ đóng vai authoritative cho `external.lab9.test` với độ trễ phản hồi mô phỏng.
- Thiết lập baseline DNS trước tấn công và bắt gói bằng Wireshark trên máy Kali.
- Thực hiện 3 kỹ thuật tấn công DNS cục bộ: thay đổi `/etc/hosts`, direct DNS spoofing (giả mạo phản hồi tới Victim), và DNS cache poisoning (đầu độc cache của resolver).
- Khôi phục hệ thống về trạng thái sạch và rút ra các biện pháp hardening.
- Đối chiếu bằng chứng Wireshark (Transaction ID, cặp IP/port, cuộc đua phản hồi) và trả lời 10 câu hỏi phân tích của đề.

## 2. Môi trường thực hành

- **Phần mềm ảo hóa:** VMware Workstation.
- **Firewall:** pfSense CE (máy `pfSenseLab6` dùng lại từ LAB 6/7, bổ sung card DNSLAB).
- **DNS-SERVER:** Ubuntu Server 24.04, BIND9 (`named`) — authoritative `lab9.test` + recursive/caching `10.9.0.0/24`.
- **VICTIM:** Ubuntu 24.04, DNS duy nhất trỏ về DNS-SERVER, cấu hình bằng Netplan.
- **ATTACKER:** Kali Linux — `python3-scapy`, `wireshark`, `tcpdump`; chạy web giả lập và script giả mạo DNS.
- **DMZ-WEB:** Ubuntu (dùng lại từ LAB 6/7) — website hợp lệ `172.16.6.10` và AUTH-DNS trên APP-TEST `172.16.6.20` (`auth_dns_delayed.py`, delay 1.5s).

### Mô hình địa chỉ IP

| Thiết bị | Interface / mạng | Địa chỉ IP | Gateway | DNS |
|---|---|---|---|---|
| pfSense | MGMT / VMnet2 | 192.168.56.254/24 | - | - |
| Máy thật quản trị | VMnet2 | 192.168.56.1/24 | - | - |
| pfSense | DNSLAB / VMnet6 | 10.9.0.1/24 (không gán gateway) | - | - |
| DNS-SERVER | DNSLAB / VMnet6 | 10.9.0.10/24 | 10.9.0.1 | 10.9.0.10 |
| VICTIM | DNSLAB / VMnet6 | 10.9.0.100/24 | 10.9.0.1 | 10.9.0.10 |
| ATTACKER (Kali) | DNSLAB / VMnet6 | 10.9.0.200/24 | 10.9.0.1 | 10.9.0.10 |
| pfSense | DMZ / VMnet4 | 172.16.6.1/24 | - | - |
| DMZ-WEB (web hợp lệ) | DMZ / VMnet4 | 172.16.6.10/24 | 172.16.6.1 | - |
| APP-TEST (AUTH-DNS) | DMZ / VMnet4 | 172.16.6.20/24 | 172.16.6.1 | - |

**Luồng DNS:** VICTIM `10.9.0.100` → DNS-SERVER `10.9.0.10` (recursive) → AUTH-DNS `172.16.6.20` (authoritative `external.lab9.test`) → trả A = `172.16.6.10`.

## 3. Nội dung thực hiện

### 3.1. Chuẩn bị môi trường

- Cài sẵn các gói cần Internet khi card còn NAT: DNS-SERVER (`bind9 bind9-utils dnsutils`), VICTIM (`dnsutils`), Kali (`python3-scapy wireshark tcpdump`); sau khi vào mạng cô lập sẽ không ra Internet được.
- Kiểm tra toàn vẹn gói tài nguyên bằng `SHA256SUMS` (dùng PowerShell trên máy Windows thật).
- Backup cấu hình pfSense và tạo snapshot `LAB8_START` trước khi thay đổi; tắt các máy không liên quan (Windows Client A/B, DC01, các firewall khác) để nhẹ RAM.

### 3.2. Tạo mạng VMnet6 và gán interface pfSense

- Trong **Virtual Network Editor**: Add Network → **VMnet6** → **Host-only**, **bỏ tick** "Use local DHCP"; Subnet `10.9.0.0 / 255.255.255.0`.
- Tắt hẳn pfSense, thêm Network Adapter **Custom: VMnet6**, bật lại pfSense.
- Trên WebGUI: Interfaces → Assignments → gán card mới, Description `DNSLAB`, Static IPv4 `10.9.0.1/24`, **không chọn Upstream Gateway**.

### 3.3. Cấu hình ma trận cô lập DNSLAB trên pfSense

Ruleset trên interface DNSLAB, xét từ trên xuống (first-match):

1. Pass `10.9.0.10 → 172.16.6.20` UDP/TCP 53 (resolver forward tới AUTH-DNS).
2. Pass `10.9.0.0/24 → 172.16.6.10` TCP 80 (client xem web hợp lệ trong DMZ).
3. Block + Log `DNSLAB net → 10.6.0.0/24` (LAN-DOMAIN của lab trước).
4. Block + Log `DNSLAB net → 192.168.56.0/24` (MGMT).
5. Block + Log `DNSLAB net → any` (explicit deny ra ngoài).

### 3.4. Cấu hình DNS-SERVER (BIND9)

- Đặt IP tĩnh `10.9.0.10/24` trên card VMnet6, gateway `10.9.0.1`.
- Tạo zone thuận `lab9.test` và zone nghịch `0.9.10.in-addr.arpa`; cấu hình recursion cho `10.9.0.0/24` và forward `external.lab9.test` tới AUTH-DNS `172.16.6.20`.
- Validate: `named-checkconf` và `named-checkzone` đều OK, dịch vụ `named` active (running).
- Kiểm tra zone cục bộ: `dig @10.9.0.10 www.lab9.test +short` → `172.16.6.10`; `dig @10.9.0.10 -x 10.9.0.10 +short` → `ns.lab9.test.`

### 3.5. Cấu hình VICTIM và ATTACKER

- **VICTIM:** đặt IP tĩnh `10.9.0.100/24` bằng Netplan, DNS **duy nhất** `10.9.0.10` (không sửa `/etc/resolv.conf` thủ công vì systemd-resolved sẽ ghi đè); kiểm tra `resolvectl status` → DNS = 10.9.0.10, `resolvectl query www.lab9.test` → `172.16.6.10`.
- **ATTACKER (Kali):** đặt IP tĩnh `10.9.0.200/24` bằng `nmcli`; dựng web giả lập tại `http://10.9.0.200` (trang mô phỏng, không có form thu thập dữ liệu); mở Wireshark bắt gói trên card DNS-LAB.

### 3.6. Baseline DNS trước tấn công

- Trên APP-TEST chạy `auth_dns_delayed.py`: AUTH-DNS listening `172.16.6.20:53`, delay 1.5s, `www.external.lab9.test → 172.16.6.10`.
- Từ DNS-SERVER: `dig @10.9.0.10 www.external.lab9.test` → `172.16.6.10`; `curl http://www.external.lab9.test` trả về trang hợp lệ.
- Trên Kali/Wireshark (filter `dns.qry.name == "www.external.lab9.test"`): thấy query Victim `10.9.0.100 → 10.9.0.10` và query đệ quy `10.9.0.10 → 172.16.6.20`, các phản hồi hợp lệ đều trả A = `172.16.6.10`.

### 3.7. Kiểm thử theo kịch bản

| Kịch bản | Đích kiểm thử | Kết quả |
|---|---|---|
| Baseline | `www.external.lab9.test` | Phân giải về `172.16.6.10` từ cả DNS-SERVER lẫn Victim |
| Task 1 – HOSTS file | `www.lab9.test` trên Victim | `/etc/hosts` ưu tiên hơn DNS: `curl` → `10.9.0.200`, nhưng `dig @10.9.0.10` vẫn → `172.16.6.10` |
| Task 2 – Direct DNS spoofing | `www.external.lab9.test` | Phản hồi giả (cùng Transaction ID `0xfe1a`, khớp cổng) thắng cuộc đua → Victim nhận `10.9.0.200` (TTL 120) |
| Task 3 – Cache poisoning | `www.external.lab9.test` | Resolver chấp nhận & cache bản ghi giả → mọi client dùng chung DNS-SERVER đều nhận `10.9.0.200` suốt TTL, kể cả sau khi dừng script |
| Khôi phục | `www.external.lab9.test` | Sau `rndc flush` + `resolvectl flush-caches` → phân giải trở lại `172.16.6.10` |

## 4. Kết quả thực hiện

- Dựng thành công mạng DNS-LAB `10.9.0.0/24` cô lập; pfSense chặn và log mọi lưu lượng từ DNSLAB sang LAN-DOMAIN, MGMT và ra ngoài.
- BIND9 trên DNS-SERVER hoạt động đúng vai trò authoritative + recursive/caching; baseline phân giải chuẩn trước tấn công.
- Cả 3 Task đều cho kết quả đúng kỳ vọng, có ảnh minh chứng và bằng chứng Wireshark:
  - Task 1: redirect cục bộ qua `/etc/hosts`, `dig` vẫn trả bản ghi thật.
  - Task 2: direct spoofing thành công nhờ trùng Transaction ID + cặp (IP, port) và thắng cuộc đua thời gian.
  - Task 3: cache poisoning thành công, bản ghi giả tồn tại trong cache resolver suốt TTL (120s); `rndc dumpdb` xác nhận bản ghi `www.external.lab9.test A 10.9.0.200`.
- Khôi phục hệ thống về trạng thái sạch; trả lời đủ 10 câu hỏi phân tích.

## 5. Lưu ý

- Gói tài nguyên và đề đánh số nội bộ là "Lab 9" (`LAB9_Resources`, domain `lab9.test`, mạng `10.9.0.0/24`, script `dns_spoof_lab9.py`), nhưng môn gọi là **Lab 8**: giữ nguyên tên kỹ thuật `lab9`, chỉ tên bài và file nộp là Lab 8 (`11CNPM1-Lab8_1150080068-TranMinhNhat.docx`).
- Đề viết theo **VirtualBox**, bài làm chạy **VMware Workstation**: dùng **VMnet6 (Host-only, tắt DHCP)** thay cho "Internal Network"; Promiscuous Mode bật trong Wireshark để Kali nghe lén được truy vấn unicast.
- `dig` gửi truy vấn thẳng tới DNS-SERVER nên bỏ qua `/etc/hosts` và NSS — dùng `dig` để phát hiện redirect cục bộ; `curl`/trình duyệt tuân theo `/etc/hosts`.
- Task 2 chỉ ảnh hưởng Victim đang hỏi (không lưu dùng chung); Task 3 ảnh hưởng mọi client dùng chung resolver trong suốt TTL — khác biệt mấu chốt giữa spoofing và cache poisoning.
- Trên Ubuntu, cấu hình DNS cho Victim bằng Netplan/Network Settings, không sửa `/etc/resolv.conf` thủ công (systemd-resolved sẽ ghi đè).
- Các rule pfSense xét theo thứ tự từ trên xuống; khi đổi rule phải Apply Changes và Reset States trước khi kiểm thử lại.
- Toàn bộ thao tác giới hạn trong mạng lab cô lập, domain `.test`; web giả trên Kali chỉ mô phỏng, không thu thập dữ liệu.
- Sau buổi thực hành: flush cache, khôi phục `/etc/hosts`, dừng script giả mạo, trả Promiscuous Mode của Kali về **Deny**.
- Không đưa private key, credential hay `config.xml` của pfSense lên repository Public.