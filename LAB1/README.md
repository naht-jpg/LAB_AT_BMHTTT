## Họ và tên sinh viên: Trần Minh Nhật
## MSSV: 1150080068
## Lab 1: Bắt gói tin Telnet - SSH

## Nội dung thực hiện:
Đã thực hiện kết nối cả 3 máy với nhau, sử dụng kali linux làm server (10.0.0.1), windows 11 trong vmware làm client (10.0.0.2) và máy thật có wireshark làm attacker (10.0.0.3) cùng kết nối trong mạng nội bộ 10.0.0.0/24 thông qua VMnet3 sau đó tạo 1 user mới trên server, và tiến hành bắt gói tin khi thực hiện kết tối telnet và ssh từ máy client đến server. Qua wireshark từ máy attacker thu thập thông tin gói tin telnet và ssh, sau đó phân tích các gói tin này để tìm hiểu về cách thức hoạt động của các giao thức này, cũng như các so sánh khả năng bảo vệ dữ liệu của Telnet và SSH thông qua kết quả bắt gói bằng Wireshark.

## Kết quả thực hiện:
- Đã thiết lập thành công mô hình mạng gồm:
  + Kali Linux Server: 10.0.0.1
  + Windows 11 Client: 10.0.0.2
  + Máy Attacker: 10.0.0.3
- Các máy có thể kết nối và ping qua lại với nhau trong mạng 10.0.0.0/24.
- Đã cài đặt và cấu hình Telnet Server trên Kali Linux, dịch vụ hoạt động tại TCP port 23.
- Đã kết nối Telnet thành công từ Windows 11 Client đến Kali Linux Server bằng PuTTY.
- Đã sử dụng Wireshark để bắt và phân tích lưu lượng Telnet. Qua Follow TCP Stream có thể quan sát được dữ liệu trao đổi của phiên Telnet ở dạng có thể đọc được.
- Đã thử nghiệm lại Telnet với mật khẩu phức tạp và nhận thấy việc tăng độ phức tạp của mật khẩu không làm cho kênh truyền Telnet được mã hóa.
- Đã cài đặt và cấu hình SSH Server trên Kali Linux, dịch vụ hoạt động tại TCP port 22.
- Đã kết nối SSH thành công từ Windows 11 Client đến Kali Linux Server.
- Đã sử dụng Wireshark để bắt và phân tích lưu lượng SSH. Wireshark vẫn quan sát được địa chỉ IP, port, thời gian và kích thước gói tin nhưng không thể đọc trực tiếp nội dung phiên như Telnet do dữ liệu SSH đã được mã hóa.
- Đã tìm hiểu và thực hiện xác thực SSH bằng Public Key Authentication.
- Qua bài thực hành có thể thấy SSH an toàn hơn Telnet trong việc bảo vệ dữ liệu và thông tin xác thực trên đường truyền.

## Các lưu ý cần thiết để giảng viên có thể kiểm tra hoặc chạy lại bài làm.
- Mô hình mạng sử dụng VMware VMnet3 với dải mạng 10.0.0.0/24.
- Server sử dụng Kali Linux tại địa chỉ 10.0.0.1.
- Client sử dụng Windows 11 chạy trên VMware tại địa chỉ 10.0.0.2.
- Máy Attacker sử dụng địa chỉ 10.0.0.3 và cài đặt Wireshark.
- Telnet sử dụng TCP port 23, SSH sử dụng TCP port 22.
- Client sử dụng PuTTY để thực hiện kết nối Telnet và SSH tới Server.
- Do cơ chế hoạt động của VMware virtual switch, máy Attacker có thể không quan sát đầy đủ lưu lượng unicast giữa Client và Server. Trong trường hợp này Wireshark được chạy trực tiếp trên Client để thu được đầy đủ phiên kết nối.
- Tài khoản thử nghiệm trên Server là `uitlab1`.
- Không sử dụng mật khẩu hoặc private key thật trong các file được đưa lên Repository.
- Public key có thể được lưu trong `/home/uitlab1/.ssh/authorized_keys` để kiểm tra chức năng SSH Public Key Authentication.