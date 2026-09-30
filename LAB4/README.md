# Lab 4 — Khảo sát và đánh giá bề mặt mạng bằng Nmap

Lab thuộc môn **An toàn hệ thống thông tin**, thực hành phát hiện host, khảo sát cổng TCP/UDP, nhận diện dịch vụ và hệ điều hành, kiểm tra thông tin SMB và lưu bằng chứng để viết báo cáo.

README này mô tả môi trường thực tế và các kết quả đã được xác nhận trong quá trình thực hành ngày **30/09/2026**. Những bước mới chỉ có thông báo đã chạy nhưng chưa kiểm tra output được ghi riêng để tránh coi là đã nghiệm thu.

## 1. Tài liệu và phạm vi

- [Tài liệu hướng dẫn Lab 4](LAB4_Nmap_HuongDan_2026.docx).
- [Mẫu báo cáo Word để chèn ảnh](LAB4_BaoCao_ChenAnh.docx).
- Chỉ quét các máy thuộc phạm vi lab được phép trong mạng Host-Only.
- Tài liệu gốc minh họa bằng VirtualBox; môi trường thực tế dùng **VMware VMnet1**. Cần đối chiếu yêu cầu phần mềm của giảng viên khi nộp bài.
- Kali có thể dùng NAT tạm thời để cài gói; ngắt NAT trước khi quét theo yêu cầu tài liệu. Metasploitable 2 chỉ nối Host-Only.
- Tài liệu yêu cầu sinh viên **tự gõ lệnh, chụp màn hình và tự phân tích kết quả**. Các lệnh bên dưới dùng để tham khảo và đối chiếu cú pháp.

## 2. Mục tiêu và yêu cầu

1. Cài và kiểm tra Nmap trên Windows và Kali; hiểu vai trò Npcap.
2. Dựng mạng Host-Only, xác định IP thật, tạo snapshot `Before-LAB4` và kiểm tra kết nối.
3. Phát hiện host bằng `-sn`; ghi IP, MAC/vendor và vai trò đã đối chiếu.
4. So sánh TCP Connect, SYN, FIN, Xmas, NULL và ACK scan.
5. Quét 20 cổng UDP phổ biến và giải thích trạng thái thu được.
6. Nhận diện phiên bản dịch vụ bằng `-sV`, hệ điều hành bằng `-O`, thông tin tổng hợp bằng `-A`.
7. Chạy NSE thu thập thông tin SMB và kiểm tra dấu hiệu MS17-010; kết luận theo output.
8. Xuất normal text, XML, grepable; lọc kết quả và chuyển XML sang HTML.
9. So sánh kết quả trước/sau hardening trên Windows VM hoặc dịch vụ test do giảng viên cung cấp.
10. Hoàn thiện ảnh minh chứng, bảng phân tích, câu hỏi và phần bổ sung được giao.

## 3. Môi trường thực hành

| Thành phần | Thông tin đã ghi nhận |
|---|---|
| Windows thật | Nmap 7.991, Npcap 1.88 |
| Kali | Nmap 7.99; interface `eth0`; cấu hình NetworkManager tên `LAB1` |
| Mạng lab | VMware VMnet1, `172.16.16.0/24` |
| Windows VMnet1 | `172.16.16.1/24` |
| Metasploitable 2 | `172.16.16.128/24`; MAC `00:0C:29:EF:1B:0C` |
| Kali sau khi sửa mạng | Chưa có output IP mới để xác nhận; `172.16.16.129` là ứng viên từ discovery |
| Windows VM phục vụ hardening | Chưa xác nhận có hay không |
| Snapshot Before-LAB4 | Chưa có bằng chứng xác nhận |

Ban đầu Kali dùng `192.168.50.10/24`, khác subnet với Metasploitable và báo `Network is unreachable`. Sau khi chỉnh cấu hình kết nối, người thực hành xác nhận đã ping được máy đích. Cần lưu ảnh IP Kali sau khi sửa và kết quả ping vào báo cáo.

Kiểm tra IP hiện tại trước mỗi buổi thực hành, vì DHCP có thể cấp địa chỉ khác:

```bash
# Kali
nmap --version
ip -br addr
ip route
ping -c 4 172.16.16.128
```

```bash
# Metasploitable 2
ifconfig
```

```powershell
# Windows thật
nmap --version
ipconfig
```

Trên Windows, thư mục Nmap là `C:\Program Files (x86)\Nmap`. Thêm thư mục này vào biến `Path` có sẵn nếu cần; giữ nguyên các đường dẫn Python và phần mềm khác.

## 4. Tổ chức bằng chứng

Trên Kali:

```text
~/LAB4/
├── results/       # Output Nmap, XML, grepable và HTML
└── screenshots/   # Ảnh câu lệnh và kết quả thực tế
```

```bash
mkdir -p ~/LAB4/results ~/LAB4/screenshots
ls -lh ~/LAB4/results
```

Thư mục này nằm trong Kali, khác với thư mục dự án trên Windows chứa README và tài liệu Word. Tệp kết quả do lệnh có `sudo` tạo có thể thuộc sở hữu `root`; quyền đọc đang hiển thị vẫn cho phép người dùng xem các tệp đã cung cấp.

Chạy lại với cùng tên output có thể ghi đè bằng chứng cũ. Đổi tên tệp nếu cần giữ nhiều lần đo.

## 5. Phát hiện host

```bash
sudo nmap -sn 172.16.16.0/24 -oN ~/LAB4/results/05_host_discovery.txt
```

Kết quả đã xác nhận: **256 địa chỉ được kiểm tra, 4 host up, 10,93 giây**.

| IP | MAC / Vendor | Vai trò |
|---|---|---|
| `172.16.16.1` | `00:50:56:C0:00:01` / VMware | Adapter VMnet1 trên Windows; đã đối chiếu |
| `172.16.16.128` | `00:0C:29:EF:1B:0C` / VMware | Metasploitable 2; MAC khớp `ifconfig` |
| `172.16.16.129` | Output không hiển thị | Có thể là Kali; cần đối chiếu `ip -br addr` |
| `172.16.16.254` | `00:50:56:E2:72:73` / VMware | Có thể là thành phần DHCP của VMware; chưa xác nhận cấu hình |

Không suy ra vai trò chắc chắn chỉ từ hậu tố IP hoặc vendor. Số host up có thể gồm máy thật và thành phần mạng ảo.

## 6. So sánh các kỹ thuật TCP

Các lệnh dưới đây đều đã có kết quả đối chiếu. Mỗi lượt quét dùng 1.000 cổng TCP phổ biến mặc định.

```bash
nmap -sT 172.16.16.128 -oN ~/LAB4/results/06_tcp_connect.txt
sudo nmap -sS 172.16.16.128 -oN ~/LAB4/results/06_tcp_syn.txt
sudo nmap -sF 172.16.16.128 -oN ~/LAB4/results/06_tcp_fin.txt
sudo nmap -sX 172.16.16.128 -oN ~/LAB4/results/06_tcp_xmas.txt
sudo nmap -sN 172.16.16.128 -oN ~/LAB4/results/06_tcp_null.txt
sudo nmap -sA 172.16.16.128 -oN ~/LAB4/results/06_tcp_ack.txt
```

| Kỹ thuật | Open | Closed | Filtered | Open\|filtered | Unfiltered | Thời gian |
|---|---:|---:|---:|---:|---:|---|
| TCP Connect `-sT` | 23 | 977 | 0 | — | — | 4,63 giây |
| SYN `-sS` | 23 | 977 | 0 | — | — | 4,71 giây |
| FIN `-sF` | — | 977 | 0 | 23 | — | 6,95 giây |
| Xmas `-sX` | — | 977 | 0 | 23 | — | 7,37 giây |
| NULL `-sN` | — | 977 | 0 | 23 | — | 5,86 giây |
| ACK `-sA` | — | — | 0 | — | 1.000 | Cần đọc lại dòng cuối tệp |

Dấu `—` chỉ trạng thái không được kỹ thuật đó xác định trong kết quả này; không dùng để thay thế số 0.

Các cổng TCP được xác nhận mở bằng `-sT` và `-sS`:

```text
21, 22, 23, 25, 53, 80, 111, 139, 445, 512, 513, 514,
1099, 1524, 2049, 2121, 3306, 5432, 5900, 6000, 6667, 8009, 8180
```

### Nhận xét để đối chiếu báo cáo

- `-sT` dùng lời gọi kết nối của hệ điều hành và hoàn tất bắt tay TCP khi kết nối thành công; trên Kali không cần quyền gửi gói raw như `-sS`.
- `-sS` thường gửi RST sau SYN/ACK để không hoàn tất kết nối. Hai lần quét cho cùng số cổng mở/đóng; chênh lệch 0,08 giây không chứng minh kỹ thuật nào luôn nhanh hơn.
- FIN/Xmas/NULL nhận RST từ 977 cổng nên xác định `closed`. 23 cổng còn lại là `open|filtered`: không đủ thông tin để phân biệt mở với bị lọc. Không ghi riêng kết quả này thành `open`.
- ACK nhận RST từ cả 1.000 cổng nên ghi `unfiltered`. Kết quả không phân biệt mở/đóng và không chứng minh máy hoàn toàn không có firewall.
- Nhãn dịch vụ trong lượt quét không có `-sV` chủ yếu dựa vào số cổng. Cần nhận diện phiên bản để kiểm tra dịch vụ thực tế.

## 7. Quét UDP có kiểm soát

```bash
sudo nmap -sU --top-ports 20 172.16.16.128 -oN ~/LAB4/results/07_udp_top20.txt
cat ~/LAB4/results/07_udp_top20.txt
```

Đã thấy tệp kết quả trong danh sách thư mục, nhưng chưa có nội dung để thống kê trạng thái. Báo cáo cần ghi cổng UDP, trạng thái, dịch vụ và giải thích. UDP có thể chậm và xuất hiện `open|filtered`; không tự quy đổi thành cổng mở.

## 8. Dịch vụ và hệ điều hành

```bash
nmap -sV 172.16.16.128 -oN ~/LAB4/results/08_service_version.txt
sudo nmap -O 172.16.16.128 -oN ~/LAB4/results/08_os_detection.txt
sudo nmap -A 172.16.16.128 -oN ~/LAB4/results/08_aggressive.txt
```

- Đã thấy tệp `08_service_version.txt` và `08_os_detection.txt`; chưa đối chiếu nội dung.
- Người thực hành thông báo đã chạy lượt `-A`; cần kiểm tra output và dòng `Nmap done`.
- Ghi phiên bản tại các cổng 21, 22, 80, 445, 3306 nếu phát hiện; không điền phiên bản dựa trên tên cổng.
- Ghi độ tin cậy/cảnh báo của OS detection; không coi dự đoán là tuyệt đối.
- `-A` bật nhận diện phiên bản, OS detection, default scripts và traceroute. So sánh thông tin và thời gian với `-sV`, `-O` riêng lẻ.

## 9. NSE và SMB

```bash
sudo nmap -p 445 --script smb-os-discovery 172.16.16.128 -oN ~/LAB4/results/09_smb_info.txt
sudo nmap -p 445 --script smb-vuln-ms17-010 172.16.16.128 -oN ~/LAB4/results/09_ms17_010.txt
```

### Thông tin SMB đã xác nhận

| Trường | Kết quả |
|---|---|
| Cổng | `445/tcp open` |
| OS / dịch vụ trả về | `Unix (Samba 3.0.20-Debian)` |
| Computer name | `metasploitable` |
| NetBIOS computer name | Trường output để trống |
| Domain | `localdomain` |
| FQDN | `metasploitable.localdomain` |
| Thời gian smb-os-discovery | 4,69 giây |
| Thời gian kiểm tra MS17-010 | 4,72 giây |

### Kết luận MS17-010

Lượt kiểm tra hoàn tất nhưng không xuất kết luận của script và không có dòng `VULNERABLE`. Kết quả chưa cung cấp bằng chứng mục tiêu bị ảnh hưởng bởi MS17-010; **không dùng để khẳng định máy đã được vá hoặc không có lỗ hổng SMB**.

Mục tiêu được nhận diện là Unix/Samba. MS17-010 liên quan SMB trên Windows; chỉ có cổng 445 mở không chứng minh mục tiêu bị ảnh hưởng. Nếu bài yêu cầu so sánh với Windows đã cập nhật, cần thêm máy Windows VM và output thực tế.

## 10. Xuất kết quả và HTML

```bash
nmap -sV 172.16.16.128 -oA ~/LAB4/results/10_export
ls -lh ~/LAB4/results/10_export.*
grep "445/open" ~/LAB4/results/10_export.gnmap
```

Đã xác nhận tồn tại ba tệp:

| Tệp | Dung lượng hiển thị | Mục đích |
|---|---|---|
| `10_export.nmap` | 1,8 KB | Văn bản đọc trực tiếp |
| `10_export.xml` | 15 KB | Dữ liệu có cấu trúc, chuyển đổi báo cáo |
| `10_export.gnmap` | 1,4 KB | Lọc nhanh bằng grep |

Tạo HTML khi đã có `xsltproc` và stylesheet Nmap:

```bash
xsltproc -o ~/LAB4/results/10_report.html /usr/share/nmap/nmap.xsl ~/LAB4/results/10_export.xml
xdg-open ~/LAB4/results/10_report.html
```

Chưa có output xác nhận bước lọc grep và tạo HTML. Nếu báo thiếu `xsltproc`, cần cài gói trong giai đoạn Kali được nối NAT tạm thời rồi ngắt NAT trước khi tiếp tục quét; không nối Metasploitable ra NAT để cài gói.

## 11. Hardening trước và sau

Phần này chưa thực hiện/đối chiếu. Theo đề, chọn **Windows VM của sinh viên hoặc dịch vụ test do giảng viên cung cấp** trong Host-Only.

1. Ghi IP máy test, dịch vụ và trạng thái ban đầu; chuẩn bị snapshot.
2. Quét `-sV`, lưu kết quả `before`.
3. Thực hiện một thay đổi phòng thủ: tắt dịch vụ test, chỉnh rule firewall hoặc cấu hình mạng phù hợp.
4. Quét lại cùng IP, cổng và tùy chọn, chỉ đổi tên output thành `after`.
5. So sánh trạng thái cổng, số cổng mở/bị lọc và dịch vụ. Gắn kết luận với thay đổi đã thực hiện.
6. Khôi phục snapshot nếu thay đổi gây hỏng môi trường.

Không tạo trước kết quả `after` khi chưa thực hiện thay đổi. Hiện chưa xác nhận có Windows VM hoặc dịch vụ test phù hợp.

## 12. Ảnh và nội dung báo cáo

| Ảnh bắt buộc | Nội dung |
|---|---|
| 1 | `ip -br addr` trên Kali sau khi cấu hình đúng |
| 2 | `ifconfig`/`ip` trên Metasploitable 2 |
| 3 | Host discovery `-sn` |
| 4 | TCP `-sS` hoặc `-sT` |
| 5 | Nhận diện dịch vụ `-sV` |
| 6 | Nhận diện OS `-O` hoặc `-A` |
| 7 | Một NSE script và kết luận dựa trên output |
| 8 | Tệp .txt/.xml và/hoặc HTML |

Ngoài danh mục tối thiểu, lưu ảnh các kỹ thuật so sánh, UDP và before/after hardening để chứng minh các mục đã thực hiện. Ảnh cần đọc được lệnh, IP đích và output; chia nhiều ảnh khi kết quả dài.

Trong Word, thay dòng **[CHÈN ẢNH TẠI ĐÂY]** bằng ảnh qua **Insert → Pictures**, chọn **In Line with Text**, giữ tỉ lệ ảnh. Điền tên, MSSV, lớp, IP thực tế và nhận xét; xóa các dòng hướng dẫn khi nộp.

### Câu hỏi phân tích

1. Phân biệt open, closed, filtered; nêu tình huống cho mỗi trạng thái.
2. So sánh quyền và cơ chế kết nối của `-sT` với `-sS`.
3. Vì sao FIN/Xmas/NULL phụ thuộc hệ điều hành và firewall?
4. ACK scan trả lời câu hỏi nào khác SYN scan?
5. Vì sao UDP chậm và dễ có open|filtered?
6. Vì sao cần `-sV` khi quản lý lỗ hổng, thay vì chỉ biết port 80?
7. Giới hạn của OS fingerprinting là gì?
8. NSE timeout có đồng nghĩa không có lỗ hổng không?
9. Thay đổi port state nào chứng minh tác động của hardening?
10. Nêu ba cấu hình phòng thủ giúp giảm bề mặt tấn công.

## 14. Bài tập bổ sung theo đề

Tài liệu gốc đánh số từ mục 12 sang 14; README giữ cách đánh số này để đối chiếu.

- So sánh `-sT`, `-sS`, `-sA` trên cùng Metasploitable 2. Đã có số liệu ở mục 6, cần hoàn thiện bảng/nhận xét.
- Quét toàn bộ 65.535 cổng TCP, so sánh với lần quét mặc định.
- Tra cứu ít nhất một dịch vụ có phiên bản cũ và CVE liên quan nếu có; ghi nguồn, ngày truy cập và điều kiện ảnh hưởng.
- So sánh kiểm tra MS17-010 trên Metasploitable với Windows đã cập nhật nếu có.
- So sánh quét thường với decoy trong lab, đối chiếu log và nêu ý nghĩa phòng thủ.
- Dùng `-oA`, mở ba định dạng để so sánh. Đã có ba tệp ở mục 10; còn phần nhận xét.
- Lập bản đồ host, cổng mở, dịch vụ; chọn ba dịch vụ rủi ro nhất và đề xuất khắc phục.
- Trả lời thêm: chi phí quét đủ cổng; decoy/fragmentation và phòng thủ; lợi ích `-oA`; ý nghĩa khi mọi cổng đều filtered.

## Tiến độ cần hoàn thiện

- [x] Cài và kiểm tra Nmap trên Windows/Kali.
- [x] Khắc phục khác subnet; người thực hành xác nhận ping được máy đích.
- [x] Tạo thư mục results và screenshots.
- [x] Xác nhận host discovery và các kỹ thuật TCP.
- [x] Đọc và phân tích hai output NSE SMB.
- [x] Xác nhận đã xuất ba tệp bằng `-oA`.
- [ ] Đối chiếu IP Kali mới, vai trò host .254 và snapshot Before-LAB4.
- [ ] Đọc output UDP, `-sV`, `-O`, `-A` để điền số liệu chính xác.
- [ ] Xác nhận lọc grep và tạo/mở báo cáo HTML.
- [ ] Thực hiện hardening với bằng chứng before/after.
- [ ] Hoàn thiện ảnh trong Word, câu hỏi phân tích và phần bổ sung được giao.

## Lỗi thường gặp trong quá trình thực hành

| Lỗi | Nguyên nhân và cách xử lý |
|---|---|
| Windows không nhận `nmap` | Thêm thư mục Nmap vào `Path` hiện có rồi mở terminal mới; không tạo biến `Path1` |
| `Network is unreachable` | Kiểm tra VMnet1, IP/subnet và route; Kali từng còn IP tĩnh ở dải 192.168.50.0/24 |
| `ls: invalid option -- '~'` | Cần dấu cách: `ls -lahR ~/LAB4` |
| `Failed to resolve "oN"` | Thiếu dấu gạch ngang; phải dùng `-oN` trước tên tệp |
| Không tìm thấy tệp kết quả | Đối chiếu `ls -lh ~/LAB4/results`; `-oN` dùng đúng tên đã đặt, `-oA` sinh .nmap/.xml/.gnmap |
| Dòng `ignored states` trong ACK | Nmap gộp cổng cùng trạng thái; xem dòng `Not shown` để biết số lượng và trạng thái |
| Không có kết luận MS17-010 | Ghi đúng việc script không trả kết luận; không tự suy diễn đã vá |
