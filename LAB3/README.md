# LAB 3 - Nhận diện và ứng phó các mối đe dọa đến an toàn thông tin

## Thông tin sinh viên
- Họ và tên: Trần Minh Nhật
- MSSV: 1150080068
- Mã lớp: 11CNPM1
- Tên lab: Lab 3 - Nhận diện và ứng phó các mối đe dọa đến an toàn thông tin

## Môi trường thực hành
- VMware Workstation Pro 26H1u1
- Windows 11 25H2 x64 - OS Build 26200.8037
- Microsoft Defender Antivirus - Real-time Protection: Enabled
- Python 3.14.7
- Wireshark 4.6.8 + Npcap
- Sysmon 15.22
- Autoruns 14.3
- Process Explorer 17.14
- Mạng VM: Host-only là cấu hình mặc định; NAT chỉ được bật tạm thời khi cần tải công cụ hoặc tạo traffic HTTPS, sau đó chuyển lại Host-only.

## Cách dựng môi trường
1. Tạo máy ảo Windows 11 trên VMware Workstation Pro và cấu hình Network Adapter ở chế độ Host-only.
2. Tạo cấu trúc thư mục `C:\LAB3` gồm `Evidence`, `Tools`, `Downloads` và `Assets`.
3. Kiểm tra SHA-256 của gói `LAB3_Threats_Assets.zip`, sau đó giải nén dữ liệu LAB.
4. Tạm chuyển VM sang NAT khi cần Internet để cài Python 3.14.7, Wireshark 4.6.8 và tải Sysinternals từ nguồn Microsoft chính thức.
5. Kiểm tra phiên bản Sysmon, Autoruns, Process Explorer, Python và Wireshark trước khi thực hành.
6. Chuyển VM về Host-only để thực hiện các tình huống cục bộ; chỉ bật NAT tạm thời ở bước HTTPS khi cần Internet.
7. Cài Sysmon với cấu hình của LAB và thu baseline hệ thống/Autoruns trước khi tạo các artifact thử nghiệm.

## Các tình huống đã thực hiện
| Tình huống | Nội dung | Kết quả |
|---|---|---|
| TH1 | Xác định Asset, Vulnerability, Threat, Risk, Control và phân loại 5 nhóm nguồn đe dọa | PASS |
| TH2 | Kiểm tra chu trình phát hiện/quarantine EICAR bằng Microsoft Defender | PASS |
| TH3 | Tạo sự kiện xác thực 4624/4625/4648, đổi mật khẩu `lab3user` và kiểm chứng credential | PASS |
| TH4 | Tạo persistence lành tính, kiểm tra bằng Sysmon/Autoruns và ánh xạ listener 127.0.0.1:8080 tới tiến trình Python | PASS |
| TH5 | Capture HTTP loopback và so sánh với traffic HTTPS/TLS | PASS |
| TH6 | Local load test giới hạn trên 127.0.0.1:8080, phân tích dataset DDoS và mail bombing offline | PASS |
| TH7 | Phân tích phishing và phân loại các tình huống Social Engineering offline | PASS |
| Cleanup | Xóa artifact LAB, dừng listener, xóa tài khoản thử nghiệm, kiểm tra Defender và tính SHA-256 Evidence | PASS |

## Kết quả chính
- Microsoft Defender vẫn được giữ bật trong quá trình thực hành và phát hiện/cách ly mẫu EICAR.
- Windows Security Log ghi nhận các sự kiện xác thực liên quan đến tài khoản thử nghiệm `lab3user`.
- Sysmon, Autoruns và Process Explorer được sử dụng để tương quan persistence, tiến trình và listener cục bộ.
- HTTP cho phép quan sát Request URI dạng plaintext; HTTPS/TLS che nội dung ứng dụng nhưng vẫn để lộ một số metadata mạng.
- `local_load_test.py` chỉ chạy với target `127.0.0.1:8080`; không thực hiện DDoS, mail bomb, spoofing hoặc MITM chủ động trên mạng bên ngoài VM.
- Sau cleanup, các artifact thử nghiệm được loại bỏ và các file Evidence được tính SHA-256.

## Lỗi gặp phải và cách khắc phục
1. **Winget không mở được source khi VM ở Host-only**  
   Nguyên nhân: Host-only không có Internet.  
   Khắc phục: tạm chuyển Network Adapter sang NAT để cập nhật/cài công cụ, sau đó chuyển lại Host-only.

2. **Một số lệnh PowerShell bị lỗi khi copy trực tiếp từ PDF**  
   Nguyên nhân: lệnh bị xuống dòng, ghép nhiều lệnh trên cùng một dòng hoặc xuất hiện khoảng trắng/ký tự đặc biệt.  
   Khắc phục: nhập từng lệnh riêng, dùng đúng tham số như `-OutFile`, và chỉ dùng backtick khi thực sự cần nối dòng.

3. **Đổi mật khẩu `lab3user` không có hiệu lực ở lần thử đầu**  
   Nguyên nhân: `Read-Host` và `Set-LocalUser` bị nhập trên cùng một dòng nên `Set-LocalUser` không được thực thi.  
   Khắc phục: chạy `Read-Host ... -AsSecureString` và `Set-LocalUser ... -Password $newPw` thành hai lệnh riêng.

4. **Không thấy traffic HTTPS trong Wireshark ở lần capture đầu**  
   Nguyên nhân: đang capture trên Adapter for loopback traffic trong khi request HTTPS đi qua NIC NAT.  
   Khắc phục: chọn NIC đang hoạt động/NAT và dùng display filter `tls || tcp.port == 443`.

5. **Tạo `evidence_sha256.csv` bị lỗi file đang được sử dụng**  
   Nguyên nhân: file `evidence_sha256.csv` cũ vừa nằm trong tập file cần hash vừa là file output.  
   Khắc phục: xóa file cũ hoặc loại `evidence_sha256.csv` khỏi danh sách đầu vào trước khi tạo lại.

## Cấu trúc nộp bài
```text
LAB3/
├── README.md
├── 11CNPM1-LAB3_1150080068-TranMinhNhat.docx
├── evidence_sha256.csv
└── Evidence/
    ├── các file output/log đã làm sạch
    └── ...
```

## Lưu ý
- Không upload installer hoặc executable của Sysinternals/Wireshark/Python.
- Không upload file EICAR bị Defender quarantine.
- Không upload mật khẩu, token/API key, cookie/session, email thật hoặc dữ liệu cá nhân.
- Các log trước khi đưa lên GitHub phải được kiểm tra và làm sạch thông tin không cần thiết có thể định danh hệ thống thật.
- Repository `LAB_AT_BMHTTT` được đặt ở chế độ Public và cần kiểm tra lại bằng cửa sổ trình duyệt không đăng nhập trước khi nộp URL lên Google Classroom.
