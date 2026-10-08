# LAB THỰC HÀNH: Tấn Công Mạng LAN Với Nmap – Yersinia – Ettercap

Một trong những cách hiệu quả nhất để hiểu về bảo mật mạng là... thử tấn công nó (trong môi trường lab an toàn) trước khi học cách phòng thủ. Bài lab hôm nay của trung tâm sẽ đưa các bạn đi qua 3 công cụ kinh điển trong giới Pentest & Network Security.

# GIAI ĐOẠN 1 – TRINH SÁT VỚI NMAP

Trước khi tấn công, kẻ xấu luôn cần bản đồ mạng. Trong lab, sau khi dựng hạ tầng cơ bản (switch/router có NAT + DHCP pool), Nmap được dùng theo nhiều kỹ thuật khác nhau:
nmap -sT – TCP Connect Scan: quét toàn bộ dải IP để xem port nào mở. Điểm thú vị là bài lab cho thấy rõ trước và sau khi bật SSH/Telnet, kết quả scan thay đổi ra sao – một bài học trực quan về việc mỗi dịch vụ bật thêm đồng nghĩa với một bề mặt tấn công (attack surface) mới.
nmap -O – OS Fingerprinting: đoán hệ điều hành của router/switch dựa trên đặc điểm TCP/IP stack.
nmap -p- – quét toàn bộ 65535 port, hữu ích khi kiểm tra máy Windows có dịch vụ ẩn nào đang chạy.
nmap -sn – Ping sweep, chỉ xác định host còn sống, không quét port.

>Insight: Đây chính là bước mà mọi pentest thực tế đều bắt đầu. Việc phòng thủ ở giai đoạn này chủ yếu là giảm thiểu dịch vụ không cần thiết và dùng firewall/IDS để phát hiện các pattern quét bất thường.

# GIAI ĐOẠN 2 – KHAI THÁC GIAO THỨC LỚP 2 VỚI YERSINIA
Đây là phần đáng chú ý nhất vì tấn công lớp 2 thường bị xem nhẹ so với lớp 3 trở lên, trong khi hậu quả lại rất nghiêm trọng do các giao thức này vốn được thiết kế không có cơ chế xác thực.
a) DHCP Starvation Attack
Yersinia gửi hàng loạt gói DHCP DISCOVER giả với MAC nguồn ngẫu nhiên, khiến DHCP server cấp phát cạn kiệt toàn bộ dải IP hợp lệ chỉ trong vài giây. Hậu quả: client thật không xin được IP → mất kết nối mạng (DoS). Lab minh chứng bằng lệnh show ip dhcp binding cho thấy hàng loạt địa chỉ "ma" xuất hiện.
→ Phòng chống: DHCP Snooping (giới hạn rate + chỉ tin cậy port hướng lên uplink) kết hợp Port Security để giới hạn số MAC học được trên mỗi port truy cập.
b) Root Bridge Attack (STP Manipulation)
Bằng cách gửi BPDU với Bridge ID thấp hơn Root Bridge hiện tại, kẻ tấn công có thể "cướp" vai trò Root Bridge trong cây Spanning Tree. Khi máy tấn công (thường có băng thông/tài nguyên yếu hơn switch thật) trở thành Root, toàn bộ traffic buộc phải đi qua nó → vừa là điểm nghẽn, vừa là vị trí lý tưởng để sniffing.
→ Phòng chống: Root Guard (chặn port không cho trở thành Root Port) kết hợp BPDU Guard (tự động shutdown port truy cập nếu nhận BPDU – vì thiết bị đầu cuối không bao giờ nên gửi BPDU).
c) CDP Flooding Attack
CDP là giao thức Cisco dùng để trao đổi thông tin thiết bị lân cận, nhưng hoàn toàn không mã hóa và không xác thực. Yersinia lợi dụng điều này để gửi ồ ạt gói CDP giả, làm bảng neighbor table phình to bất thường, tiêu tốn CPU/RAM của switch/router đến mức có thể treo thiết bị – một dạng tấn công DoS ở Layer 2.
→ Phòng chống: Tắt CDP (no cdp enable) trên toàn bộ port hướng về endpoint, chỉ giữ lại giữa các thiết bị hạ tầng cần quản lý lẫn nhau.
>Insight chung của phần Yersinia: Cả ba kiểu tấn công đều khai thác cùng một điểm yếu triết học: các giao thức Layer 2 (DHCP, STP, CDP) được thiết kế cho một mạng "đáng tin cậy", không có khái niệm xác thực nguồn gửi. Đây là lý do vì sao các tính năng "Guard" (Root Guard, BPDU Guard, DHCP Snooping) đều xoay quanh nguyên tắc: không tin bất kỳ thứ gì đến từ port truy cập (access port).
# GIAI ĐOẠN 3 – MAN-IN-THE-MIDDLE VỚI ETTERCAP
Đây là phần thể hiện rõ nhất hậu quả thực tế của một cuộc tấn công thành công.
Cơ chế ARP Poisoning:
Kali chuyển card mạng sang chế độ promiscuous, gửi ARP Request để dò toàn bộ host trong subnet.
Chọn 2 mục tiêu: nạn nhân (Windows 7) và default gateway.
Ettercap liên tục gửi gói ARP Reply giả, khiến:
Nạn nhân tin rằng IP gateway ứng với MAC của Kali.
Gateway tin rằng IP nạn nhân cũng ứng với MAC của Kali.
Kết quả: mọi traffic giữa hai bên đều phải "ghé qua" Kali trước – xác minh được ngay trên bảng ARP của router khi thấy 2 IP khác nhau trỏ về cùng 1 MAC.
Sniffing & khai thác:
Khi nạn nhân đăng nhập vào một trang web dùng HTTP (không mã hóa) như altoromutual.com, Wireshark trên máy Kali bắt trọn gói tin HTTP POST, và trong phần HTML Form URL Encoded – username/password hiện nguyên dạng plaintext. Ettercap thậm chí còn có sẵn cơ chế tự động bóc tách credential cho hàng loạt giao thức cũ như FTP, Telnet, POP, IMAP, SNMP...
>Insight: Đây là minh chứng rõ ràng nhất cho việc vì sao HTTPS/TLS không phải là tùy chọn mà là bắt buộc. ARP Poisoning về bản chất không thể ngăn hoàn toàn ở lớp 2 (trừ khi dùng Dynamic ARP Inspection kết hợp DHCP Snooping), nhưng nếu dữ liệu đã được mã hóa end-to-end thì dù kẻ tấn công có đứng giữa cũng không đọc được nội dung.
# KẾT LUẬN
Bài lab này là một case study rất đầy đủ để hiểu vì sao bảo mật mạng LAN cần tiếp cận theo nhiều lớp (defense-in-depth):
Lớp 2: Port Security, DHCP Snooping, Dynamic ARP Inspection, Root Guard, BPDU Guard
Lớp ứng dụng: bắt buộc mã hóa (HTTPS/TLS) thay vì các giao thức plaintext cũ
Giám sát: theo dõi log/CDP table/ARP table bất thường để phát hiện sớm