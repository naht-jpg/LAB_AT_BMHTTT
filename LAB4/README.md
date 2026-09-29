# LAB 4 - KHẢO SÁT VÀ ĐÁNH GIÁ BỀ MẶT MẠNG BẰNG NMAP

## Thông tin sinh viên

-   **Họ và tên:** Trần Minh Nhật
-   **MSSV:** 1150080068
-   **Lớp:** 11CNPM1
-   **Ngày thực hiện:** 29/09/2026

------------------------------------------------------------------------

## 1. Mục tiêu

-   Làm quen và sử dụng Nmap trên Windows 11 và Kali Linux.
-   Xây dựng môi trường thực hành an toàn bằng VMware Host-Only.
-   Phát hiện các host đang hoạt động trong mạng nội bộ.
-   Khảo sát các cổng TCP/UDP và so sánh các kỹ thuật quét.
-   Nhận diện dịch vụ, phiên bản dịch vụ và hệ điều hành.
-   Sử dụng NSE để thu thập thông tin SMB và kiểm tra MS17-010.
-   Xuất kết quả Nmap để lưu trữ bằng chứng.
-   Quan sát sự thay đổi trước và sau khi hardening.

## 2. Môi trường thực hành

Môi trường được xây dựng hoàn toàn trên VMware và sử dụng mạng
Host-Only.

  -----------------------------------------------------------------------
  Thiết bị          IP                Subnet Mask       Vai trò
  ----------------- ----------------- ----------------- -----------------
  Windows Host      192.168.40.1      255.255.255.0     VMware Host-Only
                                                        Adapter

  Kali Linux        192.168.40.129    255.255.255.0     Máy quét

  Metasploitable 2  192.168.40.130    255.255.255.0     Máy đích

  Windows 11 VM     192.168.40.131    255.255.255.0     Máy đích đối
                                                        chiếu
  -----------------------------------------------------------------------

Metasploitable 2 chỉ được kết nối với mạng Host-Only để bảo đảm máy ảo
có lỗ hổng không tiếp xúc trực tiếp với mạng bên ngoài.

## 3. Nội dung thực hiện

### 3.1. Kiểm tra Nmap và kết nối

``` bash
nmap --version
ip -br addr
ping -c 4 192.168.40.130
```

Trên Metasploitable 2:

``` bash
ifconfig
```

### 3.2. Host Discovery

``` bash
sudo nmap -sn 192.168.40.0/24
```

Các host chính phát hiện được:

  IP               Vai trò
  ---------------- -----------------------------
  192.168.40.1     VMware Host-Only / máy thật
  192.168.40.130   Metasploitable 2
  192.168.40.131   Windows 11 VM
  192.168.40.254   VMware DHCP

### 3.3. Khảo sát cổng TCP

TCP Connect Scan:

``` bash
nmap -sT 192.168.40.130
```

SYN Scan:

``` bash
sudo nmap -sS 192.168.40.130
```

  Tiêu chí         -sT      -sS
  ----------- -------- --------
  Open              23       23
  Closed           977      977
  Filtered           0        0
  Thời gian     5.39 s   4.99 s

Hai kỹ thuật phát hiện cùng 23 cổng TCP đang mở trên Metasploitable 2.

### 3.4. FIN / Xmas / NULL Scan

``` bash
sudo nmap -sF 192.168.40.130
sudo nmap -sX 192.168.40.130
sudo nmap -sN 192.168.40.130
```

Nhiều cổng xuất hiện ở trạng thái `open|filtered`. Trạng thái này không
có nghĩa chắc chắn cổng đang mở; Nmap không thể phân biệt giữa cổng mở
và trường hợp bộ lọc làm gói tin không nhận được phản hồi.

### 3.5. ACK Scan

``` bash
sudo nmap -sA 192.168.40.130
```

ACK scan được sử dụng để quan sát chính sách lọc mạng và phân biệt
`filtered` với `unfiltered`, không dùng để khẳng định một cổng đang mở.

### 3.6. UDP Scan

``` bash
sudo nmap -sU --top-ports 20 192.168.40.130
```

Kết quả xuất hiện các trạng thái `open`, `closed` và `open|filtered`.
UDP scan thường khó xác định hơn TCP do UDP không có quá trình bắt tay.

### 3.7. Version Detection

``` bash
sudo nmap -sV 192.168.40.130
```

        Port Service   Version phát hiện
  ---------- --------- --------------------
      21/tcp FTP       vsftpd 2.3.4
      22/tcp SSH       OpenSSH 4.7p1
      80/tcp HTTP      Apache httpd 2.2.8
     445/tcp SMB       Samba smbd
    3306/tcp MySQL     MySQL 5.0.51a

### 3.8. OS Detection

``` bash
sudo nmap -O 192.168.40.130
```

Nmap nhận diện máy đích thuộc họ Linux. OS fingerprinting là kết quả
nhận diện dựa trên phản ứng của TCP/IP stack nên không được xem là kết
luận tuyệt đối.

### 3.9. Aggressive Scan

``` bash
sudo nmap -A 192.168.40.130
```

Aggressive Scan cung cấp nhiều thông tin trong một lần quét, bao gồm
service/version detection, OS detection, NSE default scripts và
traceroute.

## 4. NSE - SMB

### SMB OS Discovery

``` bash
sudo nmap -p 445 --script smb-os-discovery 192.168.40.130
```

### Kiểm tra MS17-010

Metasploitable 2:

``` bash
sudo nmap -p 445 --script smb-vuln-ms17-010 192.168.40.130
```

Windows 11:

``` bash
sudo nmap -p 445 --script smb-vuln-ms17-010 192.168.40.131
```

Trong kết quả thực hành hiện tại không xuất hiện kết luận `VULNERABLE`,
do đó không kết luận mục tiêu tồn tại MS17-010 chỉ dựa trên lần kiểm tra
này. Trên Windows VM, cổng 445 ở trạng thái `filtered`, vì vậy script
không thu thập đủ thông tin để đưa ra kết luận về lỗ hổng.

## 5. Xuất kết quả

Normal text:

``` bash
sudo nmap -sV -O 192.168.40.130 -oN ket_qua.txt
```

XML:

``` bash
sudo nmap -sV -O 192.168.40.130 -oX ket_qua.xml
```

Grepable:

``` bash
sudo nmap -p 445 192.168.40.0/24 -oG smb.txt
grep "445/open" smb.txt
```

Các file output được lưu lại để phục vụ việc kiểm tra và làm bằng chứng
cho quá trình thực hành.

## 6. Before / After Hardening

Thực hiện trên Windows 11 VM `192.168.40.131` với TCP port 8000.

### Before

``` bash
sudo nmap -sV -p 8000 192.168.40.131 -oN windows_before.txt
```

Kết quả:

``` text
8000/tcp open
```

Sau đó tạo Windows Firewall Inbound Rule để chặn TCP port 8000.

### After

``` bash
sudo nmap -sV -p 8000 192.168.40.131 -oN windows_after.txt
```

Kết quả:

``` text
8000/tcp filtered
```

  Chỉ tiêu   Before   After
  ---------- -------- ----------
  Open       1        0
  Filtered   0        1
  TCP/8000   open     filtered

Sau khi thêm firewall rule, Kali không còn truy cập trực tiếp được đến
TCP/8000. Kết quả chứng minh biện pháp firewall đã làm giảm bề mặt dịch
vụ có thể tiếp cận từ mạng.

## 7. Kết quả đạt được

-   Xây dựng được môi trường VMware Host-Only an toàn.
-   Xác định được các host đang hoạt động trong mạng.
-   Thực hiện được TCP Connect, SYN, FIN, Xmas, NULL và ACK scan.
-   Thực hiện UDP scan.
-   Nhận diện được service/version và hệ điều hành.
-   Sử dụng NSE để thu thập thông tin SMB và kiểm tra MS17-010.
-   Phân biệt được `open`, `closed`, `filtered`, `open|filtered` và
    `unfiltered`.
-   Xuất được kết quả Nmap để lưu bằng chứng.
-   Thực hiện và chứng minh được sự thay đổi trước/sau khi hardening
    Windows Firewall.

## 8. Lưu ý

-   Toàn bộ quá trình quét được thực hiện trên các máy ảo thuộc môi
    trường thực hành cá nhân.
-   Metasploitable 2 chỉ kết nối với mạng VMware Host-Only.
-   Không sử dụng các kỹ thuật quét trong LAB để quét hệ thống bên ngoài
    khi chưa được cho phép.
-   Các địa chỉ IP và kết quả trong README là kết quả của môi trường
    thực hành tại thời điểm thực hiện LAB.
