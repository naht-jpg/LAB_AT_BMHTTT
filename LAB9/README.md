# LAB 9 - REMOTE DNS CACHE POISONING TRONG MÔI TRƯỜNG pfSense CÔ LẬP

## Thông tin sinh viên

-   **Họ và tên:** Trần Minh Nhật
-   **MSSV:** 1150080068
-   **Lớp:** 11CNPM1
-   **Ngày thực hiện:** 06/11/2026

------------------------------------------------------------------------

## 1. Mục tiêu

- Hiểu sự khác nhau giữa Local DNS spoofing (LAB trước) và Remote DNS cache poisoning/Kaminsky: attacker không còn nằm cùng mạng với resolver.
- Tách ATTACKER (Kali) khỏi DNS-LAB sang subnet riêng `10.10.0.0/24` bằng interface ATTACKNET của pfSense, để attacker không thể sniff query/transaction ID/source port của resolver.
- Quan sát vai trò của Transaction ID, UDP source port, cache và bản ghi NS/glue trong quá trình poisoning.
- Thực hiện cache poisoning trong mô hình mô phỏng an toàn bằng domain `.test` và các IP private đã hard-code; dùng resolver mô phỏng có chủ đích (`vuln_resolver_lab10.py`) để tái hiện điều kiện yếu mà không cài resolver lỗi thời.
- Kiểm chứng hậu quả bằng DNS query và trang web mô phỏng; sau đó khôi phục BIND9 hiện đại và áp dụng các biện pháp hardening.
- Đối chiếu bằng chứng Wireshark/tcpdump (cặp IP/port `10.9.0.10:33333 ↔ 172.16.6.20:53`, forged referral) và trả lời 12 câu hỏi phân tích của đề.

## 2. Môi trường thực hành

- **Phần mềm ảo hóa:** VMware Workstation.
- **Firewall:** pfSense CE (kế thừa từ LAB trước, bổ sung interface **ATTACKNET** `10.10.0.1/24`, không gán Upstream Gateway).
- **DNS-SERVER:** Ubuntu Server, BIND9 (`named`) — baseline hiện đại: recursive/caching cho `10.9.0.0/24`, forward `example.lab10.test` tới AUTH-DNS; trong attack phase tạm dừng BIND và chạy `vuln_resolver_lab10.py` (resolver mô phỏng, cố định source port 33333).
- **VICTIM:** Ubuntu 24.04, DNS duy nhất trỏ về DNS-SERVER `10.9.0.10`, giữ nguyên IP/DNS từ LAB trước.
- **ATTACKER:** Kali Linux trên subnet ATTACKNET `10.10.0.200` — chạy `evil_dns_lab10.py` (malicious authoritative DNS), web giả lập `fake_index.html` và `kaminsky_remote_lab10.py` (remote TXID guessing).
- **AUTH-DNS/RELAY:** APP-TEST `172.16.6.20` — `auth_dns_relay_lab10.py`: authoritative hợp lệ cho `example.lab10.test` (delay 3s tạo race window) kiêm race harness nhận batch TXID qua UDP/5300 **chỉ từ** `10.10.0.200`.
- **DMZ-WEB:** `172.16.6.10` — website hợp lệ (dùng lại từ LAB 7/8).

### Mô hình địa chỉ IP

| Thành phần | Vai trò | Địa chỉ IP | Kế thừa / mới |
|---|---|---|---|
| pfSense DNSLAB | Gateway DNS-LAB | 10.9.0.1/24 | Kế thừa |
| DNS-SERVER | BIND9 / resolver lab | 10.9.0.10/24 | Kế thừa |
| VICTIM | DNS client | 10.9.0.100/24 | Kế thừa |
| pfSense ATTACKNET | Gateway attacker subnet | 10.10.0.1/24 | **Mới** |
| ATTACKER (Kali) | Remote attacker + evil DNS/web | 10.10.0.200/24 | Chuyển mạng |
| AUTH-DNS/RELAY | DNS hợp lệ + race harness | 172.16.6.20/24 | Kế thừa (APP-TEST) |
| DMZ-WEB | Website hợp lệ | 172.16.6.10/24 | Kế thừa |
| HOST MGMT | Quản trị WebGUI | 192.168.56.1/24 | Kế thừa |

**Luồng DNS (baseline):** VICTIM `10.9.0.100` → DNS-SERVER `10.9.0.10` (recursive) → AUTH-DNS `172.16.6.20` (authoritative `example.lab10.test`) → trả A = `172.16.6.10`.
**Luồng sau poison:** resolver `10.9.0.10` cache delegation giả → chuyển mọi truy vấn dưới `example.lab10.test` sang evil DNS `10.10.0.200`.

## 3. Nội dung thực hiện

### 3.1. Chuẩn bị môi trường

- Tạo snapshot trước khi thay đổi: pfSense (`LAB10_START_FROM_LAB9`), Kali (trước khi tháo card DNSLAB). Không thay đổi LAB7/8 LAN-DOMAIN.
- Cài sẵn các gói cần Internet khi card còn NAT; sau khi vào attack phase không bật NAT.
- Kiểm tra toàn vẹn gói tài nguyên bằng `SHA256SUMS.txt`.

### 3.2. Thêm interface ATTACKNET cho pfSense

- VMware: thêm card mạng cho pfSense, gắn **LAN segment `attack-net`** (host-only, cô lập khỏi DNSLAB).
- WebGUI: Interfaces → Assignments → Add → Enable; Description `ATTACKNET`; Static IPv4 `10.10.0.1/24`; **KHÔNG chọn Upstream Gateway** (tránh pfSense coi interface nội bộ là kiểu WAN).
- Save → Apply Changes.

### 3.3. Chuyển Kali khỏi DNSLAB

- Tắt Kali, tháo/disable card Internal Network `dns-lab`; gắn card mới `attack-net`.
- **Promiscuous Mode = Deny** (LAB 10 không sniff trên DNSLAB).
- Đặt IPv4 `10.10.0.200/24`, gateway `10.10.0.1`; không bật NAT trong attack phase.

### 3.4. Cấu hình ma trận cô lập trên pfSense

**Interface DNSLAB** (first-match, từ trên xuống):

1. Pass `10.9.0.100 → 10.9.0.10` TCP/UDP 53 (Victim → resolver).
2. Pass `10.9.0.10 → 172.16.6.20` UDP 53 (resolver → AUTH-DNS hợp lệ).
3. Pass `10.9.0.10 → 10.10.0.200` UDP 53 (sau poison: resolver → malicious NS).
4. Pass `10.9.0.0/24 → 10.10.0.200` TCP 80 (Victim xem web bị chuyển hướng).
5. Block + Log `10.9.0.0/24 → 10.6.0.0/24`, `→ 192.168.56.0/24`, `→ any` (bảo vệ LAN7/8, MGMT; default isolation).

**Interface ATTACKNET:**

1. Pass `10.10.0.200 → 10.9.0.10` TCP/UDP 53 (remote attacker kích hoạt resolver).
2. Pass `10.10.0.200 → 172.16.6.20` UDP 5300 (lab-only TXID race relay).
3. Block + Log `10.10.0.0/24 → 10.6.0.0/24`, `→ 192.168.56.0/24`, `→ any` (không WAN, không DMZ ngoài phạm vi).

> Sau khi tạo rule: Apply Changes rồi Reset States. Rule UDP/5300 chỉ phục vụ race harness — sau lab phải disable/xóa.

### 3.5. Chuẩn bị DNS-SERVER và AUTH-DNS

- **DNS-SERVER:** sao lưu cấu hình LAB trước; nạp baseline BIND9 hiện đại — chỉ cho `10.9.0.0/24` recursion, forward `example.lab10.test → 172.16.6.20`, `dnssec-validation auto` (riêng `example.lab10.test` đặt `validate-except` vì là namespace `.test` nội bộ).
- **AUTH-DNS/RELAY:** trên APP-TEST `172.16.6.20` chạy `auth_dns_relay_lab10.py` — trả `www.example.lab10.test → 172.16.6.10`, chờ 3s trước NXDOMAIN với tên ngẫu nhiên (tạo race window), nhận batch TXID qua UDP/5300 chỉ từ `10.10.0.200`.

### 3.6. Baseline trước tấn công

- Từ VICTIM: `resolvectl query www.example.lab10.test` → `172.16.6.10`; `curl http://www.example.lab10.test` trả về trang hợp lệ.
- **Không tiếp tục nếu baseline sai** — kiểm tra forward zone, rule DNSLAB → AUTH-DNS:53, DNS của VICTIM và service `172.16.6.20`.

### 3.7. Xác nhận tính "remote"

- Chạy `tcpdump` trên Kali (ATTACKNET) lọc `10.9.0.10 ↔ 172.16.6.20`: kết quả **0 packet** — Kali không bắt được Transaction ID lẫn source port của resolver.
- Muốn quan sát race, capture trên DNS-SERVER hoặc APP-TEST (không phải trên Kali); dùng filter `udp.port == 53` hoặc `ip.addr == 10.9.0.10 && udp.port == 33333`.

### 3.8. Kiểm thử theo kịch bản

| Kịch bản | Đích kiểm thử | Kết quả |
|---|---|---|
| Baseline | `www.example.lab10.test` | Phân giải về `172.16.6.10` từ cả resolver lẫn Victim |
| Xác nhận remote | `tcpdump` trên Kali | `10.9.0.10 ↔ 172.16.6.20` → 0 packet; attacker không sniff được DNSLAB |
| Task 1 – Remote cache poisoning / Kaminsky | `www.example.lab10.test` | Gửi 16 batch TXID (mỗi batch 4096 giá trị) qua race relay; khi trúng: AUTH-DNS `[HIT]`, resolver `[CACHE]`, script `[VERIFY] = 10.10.0.200` |
| Task 2 – Phân tích cache | Bản ghi NS / glue | Forged referral: NS = `ns.evil.example.lab10.test`, glue A = `10.10.0.200`; so sánh upstream trước/sau |
| Khôi phục + hardening | `www.example.lab10.test` | Khởi động lại BIND9; recursion ACL → Kali bị `REFUSED`; source port upstream trở lại **ngẫu nhiên** (khác 33333) |

## 4. Kết quả thực hiện

- Dựng thành công subnet ATTACKNET `10.10.0.0/24` cô lập; pfSense chỉ mở đúng các luồng của bài lab, chặn và log mọi lưu lượng sang LAN7/8, MGMT và ra ngoài.
- Chứng minh được tính "remote": Kali trên ATTACKNET **không sniff** được lưu lượng `10.9.0.10 ↔ 172.16.6.20` (0 packet), trong khi Victim vẫn phân giải baseline chuẩn `172.16.6.10`.
- Task 1 thành công: race relay giúp batch TXID trúng Transaction ID thật → AUTH-DNS log `[PENDING] → [HIT]`, resolver log `[CACHE] delegation poisoned`, script xác nhận `[VERIFY] www.example.lab10.test = 10.10.0.200`.
- Task 2: bắt gói tại DNS-SERVER cho thấy truy vấn upstream `10.9.0.10:33333 → 172.16.6.20:53` và forged referral `(0/1/1)` chèn NS giả + glue `10.10.0.200`; sau poison resolver chuyển mọi truy vấn sang evil DNS. Victim phân giải `www → 10.10.0.200` và `curl` ra trang **LAB10 Fake Site**.
- Khôi phục BIND9 hiện đại (service `named` active); kiểm chứng 2 lớp phòng thủ: recursion ACL khiến Kali bị `REFUSED`, và source-port randomization (cổng nguồn upstream ngẫu nhiên thay vì 33333).
- Hoàn thành bảng so sánh Trước attack / Sau poison / Sau khôi phục và trả lời đủ 12 câu hỏi phân tích.

## 5. Lưu ý

- Gói tài nguyên và đề đánh số nội bộ dùng domain `example.lab10.test`, phạm vi mạng `10.9.0.0/24`, `10.10.0.0/24`, `172.16.6.0/24`; file nộp đặt tên **`11CNPM1-Lab10_1150080068-TranMinhNhat.docx`**.
- Đề viết theo **VirtualBox**, bài làm chạy **VMware Workstation**: dùng **LAN segment `attack-net`** thay cho "Internal Network". Khác LAB trước, **Promiscuous Mode = Deny** vì mô hình remote không sniff.
- Khác biệt mấu chốt với LAB trước (local spoofing): attacker bị tách subnet nên **thiếu Transaction ID (2¹⁶) và UDP source port** của resolver → buộc phải **đoán (brute-force)** trong race window thay vì đọc trộm.
- `vuln_resolver_lab10.py` là resolver mô phỏng **có chủ đích**, cố định source port 33333, chỉ tồn tại trong attack phase; BIND9 thật (9.18) đã random hoá source port theo mặc định.
- **Race relay chỉ là mô phỏng:** nó "biết trước" TXID thật và rút gọn không gian `2¹⁶` còn 16 batch để minh hoạ mà không flood — không phải điều kiện tấn công Internet thật.
- Malicious NS được đặt **ngay trong** `example.lab10.test` nên glue `10.10.0.200` nằm trong **bailiwick** → mô phỏng được; không suy rộng rằng resolver thật chấp nhận glue ngoài phạm vi.
- `example.lab10.test` thuộc TLD `.test` (RFC 2606/6761) không bao giờ phân giải ra Internet → tấn công hoàn toàn cô lập; web giả trên Kali chỉ mô phỏng, không thu thập dữ liệu.
- Các rule pfSense xét theo thứ tự từ trên xuống; khi đổi rule phải **Apply Changes** và **Reset States** trước khi kiểm thử lại.
- Sau buổi thực hành: dừng resolver mô phỏng và khởi động lại BIND9; thu hẹp `allow-recursion` còn `10.9.0.0/24`; disable/xóa rule UDP/5300 và interface ATTACKNET nếu lab sau không cần; giữ Kali Promiscuous = Deny; khôi phục snapshot nếu muốn trả pfSense về trạng thái LAB trước.
- Không đưa private key, credential hay `config.xml` của pfSense lên repository Public.
