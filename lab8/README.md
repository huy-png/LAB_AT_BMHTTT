# LAB 8 — Local DNS Attack với pfSense

Bài thực hành mô phỏng chuyển hướng tên miền bằng file hosts, giả mạo phản hồi DNS và đầu độc cache DNS trong mạng ảo cô lập. Môi trường sử dụng pfSense, Ubuntu 24.04, Kali Linux, BIND9, Wireshark và Scapy trên VirtualBox.

Tài liệu nguồn: [LAB8_Local_DNS_Attack_pfSense_2026.docx](../LAB8_Local_DNS_Attack_pfSense_2026.docx).

**Lưu ý về số hiệu:** tên file và trang bìa ghi LAB 8, nhưng phần lớn nội dung ghi LAB 9 và kế thừa LAB 8. README đặt trong `lab8` theo tên tài liệu/yêu cầu tổ chức dự án, đồng thời giữ nguyên các IP, domain, tên script và snapshot trong nội dung gốc. Cần xác nhận số hiệu bài và tên báo cáo với giảng viên trước khi nộp.

**Trạng thái:** đây là mô tả yêu cầu và tiêu chí kiểm thử từ tài liệu, chưa phải báo cáo kết quả thực hành. Việc tạo README không đồng nghĩa đã dựng VM hoặc thực hiện các task.

## 1. Mục tiêu

- Dựng BIND9 làm DNS authoritative cho zone nội bộ và recursive/caching resolver cho mạng DNS-LAB.
- Cấu hình zone thuận/nghịch; kiểm tra phân giải bằng `dig`, `nslookup` và `resolvectl`.
- Quan sát DNS transaction ID, UDP port, query name và các phản hồi cạnh tranh trong Wireshark.
- Phân biệt chuyển hướng cục bộ bằng hosts, giả mạo phản hồi tới client và đầu độc cache resolver.
- Khôi phục hệ thống sau thử nghiệm và đánh giá các lớp phòng vệ DNS.

## 2. Phạm vi thực hành

Theo tài liệu, các thử nghiệm chỉ được chạy trong mạng riêng `10.9.0.0/24`, với các IP của lab và tên miền `.test`. Không sử dụng mạng Wi-Fi, mạng trường, mạng doanh nghiệp, Internet hoặc thiết bị của người khác làm mục tiêu.

NAT chỉ dùng tạm để cài gói. Trước pha tấn công, tắt card NAT trên DNS-SERVER, VICTIM và Kali; ruleset pfSense phải cô lập DNSLAB khỏi WAN, LAN-DOMAIN và MGMT theo ma trận bên dưới. Trang web mô phỏng chỉ hiển thị thông báo, không có form đăng nhập hay thu thập dữ liệu.

## 3. Mô hình và địa chỉ

| Thành phần | Hệ điều hành / vai trò | Địa chỉ | RAM gợi ý |
| --- | --- | --- | --- |
| pfSense | Gateway/firewall, thêm interface DNSLAB | `10.9.0.1/24` | 2 GB |
| DNS-SERVER | Ubuntu Server 24.04, BIND9 recursive/cache | `10.9.0.10/24` | 1,5–2 GB |
| VICTIM | Ubuntu 24.04 Desktop/Server, client kiểm thử | `10.9.0.100/24` | 2 GB |
| ATTACKER | Kali Linux, Scapy, Wireshark, web mô phỏng | `10.9.0.200/24` | 2–4 GB |
| APP-TEST / AUTH-DNS | Authoritative DNS mô phỏng, kế thừa môi trường trước | `172.16.6.20` | Đã có theo tài liệu |
| DMZ-WEB | Web hợp lệ, kế thừa môi trường trước | `172.16.6.10` | Đã có theo tài liệu |
| HOST MGMT | Máy thật quản trị | `192.168.56.1` | — |

Luồng phân giải tên miền thử nghiệm:

```text
VICTIM 10.9.0.100
    -> DNS-SERVER 10.9.0.10 (BIND9 recursive/cache)
        -> pfSense DNSLAB 10.9.0.1
            -> APP-TEST/AUTH-DNS 172.16.6.20
                trả www.external.lab9.test = 172.16.6.10

Web hợp lệ:  DMZ-WEB 172.16.6.10
Web mô phỏng của attacker: Kali 10.9.0.200
```

DNS-SERVER, VICTIM và ATTACKER cùng Internal Network tên `dns-lab`. Lưu lượng giữa các máy cùng subnet không mặc nhiên đi qua pfSense; ruleset trên gateway kiểm soát lưu lượng được định tuyến ra khỏi subnet này.

## 4. Tài nguyên được tài liệu đề cập

Các mục sau là cấu trúc bộ tài nguyên mô tả trong DOCX, không phải các file đã được tạo trong thư mục `lab8`:

| Thư mục / file | Nội dung |
| --- | --- |
| `01_DNS_SERVER` | `named.conf.options`, `named.conf.local`, zone thuận/nghịch mẫu |
| `02_VICTIM` | Netplan mẫu cho VICTIM và DNS `10.9.0.10` |
| `03_ATTACKER` | `dns_spoof_lab9.py`, `fake_index.html` |
| `04_APP_TEST_AUTH_DNS` | `auth_dns_delayed.py` |
| `05_PFSENSE` | `firewall_policy_matrix.csv` |
| `06_WIRESHARK` | Bộ display filter |
| `07_REPORT` | Mẫu báo cáo Word |
| `SHA256SUMS.txt` | Kiểm tra tính toàn vẹn bộ tài nguyên |

## 5. Chuẩn bị môi trường

1. Tạo snapshot pfSense `LAB9_START_FROM_LAB8` theo tài liệu; tạo snapshot Kali trước khi thay đổi card mạng.
2. Bổ sung card Internal Network `dns-lab` cho pfSense. Tài liệu gọi đây là Adapter 5; đối chiếu số card và cách cấu hình với VM thực tế.
3. Gán interface mới là `DNSLAB`, IP tĩnh `10.9.0.1/24`, **không chọn Upstream Gateway**.
4. Thêm card `dns-lab` cho ba máy DNS-SERVER, VICTIM và Kali, cấu hình IP đúng bảng.
5. Cài các gói cần thiết khi còn NAT, sau đó tắt NAT trước thử nghiệm.
6. Trên card `dns-lab` của Kali, đặt Promiscuous Mode là `Allow VMs` theo thiết kế bài để quan sát lưu lượng giữa các VM.

### Ruleset pfSense trên DNSLAB

Giữ thứ tự từ trên xuống như tài liệu:

| Thứ tự | Action | Source | Destination | Port | Mục đích |
| --- | --- | --- | --- | --- | --- |
| 1 | Pass | `10.9.0.10` | `172.16.6.20` | UDP/TCP 53 | Resolver hỏi AUTH-DNS |
| 2 | Pass | `10.9.0.0/24` | `172.16.6.10` | TCP 80 | Truy cập web hợp lệ |
| 3 | Block + Log | `10.9.0.0/24` | `10.6.0.0/24` | Any | Bảo vệ LAN-DOMAIN |
| 4 | Block + Log | `10.9.0.0/24` | `192.168.56.0/24` | Any | Bảo vệ MGMT |
| 5 | Block + Log | `10.9.0.0/24` | Any | Any | Cô lập các lưu lượng còn lại |

Sau khi cấu hình, Apply Changes, Reset States và chụp ruleset làm minh chứng. Giữ đường quản trị pfSense qua `192.168.56.254` từ HOST MGMT; không sửa Anti-Lockout rule.

### DNS và web

- BIND9 authoritative cho `lab9.test`, đồng thời cho phép recursive/cache trong `10.9.0.0/24`.
- Zone `external.lab9.test` được forward tới `172.16.6.20`.
- Tài liệu chỉ tắt DNS COOKIE bằng `send-cookie no` cho AUTH-DNS mô phỏng để phục vụ thử nghiệm; đây không phải cấu hình production.
- Kiểm tra cấu hình và zone bằng `named-checkconf` và `named-checkzone` trước khi khởi động lại BIND9.
- VICTIM chỉ dùng DNS `10.9.0.10` trong pha thử nghiệm. Cấu hình qua Netplan hoặc Network Settings, không sửa tay `/etc/resolv.conf` trên Ubuntu dùng systemd-resolved.
- AUTH-DNS mô phỏng trả `www.external.lab9.test` về `172.16.6.10`, với độ trễ chủ đích 1,5 giây để quan sát cuộc đua phản hồi.
- Kali phục vụ trang web mô phỏng tại `10.9.0.200`.

## 6. Baseline và ba nhiệm vụ

### Baseline bắt buộc

Trước Task 2/3, xác nhận `www.external.lab9.test` phân giải về `172.16.6.10`. Nếu sai, sửa DNS, routing, ruleset hoặc AUTH-DNS trước khi tiếp tục.

Trên Wireshark, có thể dùng display filter `dns` hoặc `dns.qry.name == "www.external.lab9.test"`. Xác nhận quan sát được cả query VICTIM → DNS-SERVER và DNS-SERVER → AUTH-DNS.

| Nhiệm vụ | Cơ chế cần quan sát | Tiêu chí kết quả mong đợi |
| --- | --- | --- |
| Task 1 — Hosts-file redirection | Thay đổi ánh xạ tên trên chính VICTIM | `getent`/`curl` với `www.lab9.test` tới `10.9.0.200`, nhưng `dig @10.9.0.10 www.lab9.test` vẫn trả `172.16.6.10` |
| Task 2 — Direct DNS spoofing | Phản hồi giả gửi trực tiếp tới VICTIM, giả địa chỉ nguồn DNS-SERVER | Khi phản hồi giả thắng cuộc đua, VICTIM nhận `10.9.0.200`; Wireshark cho thấy query và forged response khớp transaction ID, query name và UDP port |
| Task 3 — DNS cache poisoning | Phản hồi giả gửi tới DNS-SERVER, giả nguồn AUTH-DNS | Nếu BIND9 chấp nhận, cache chứa `10.9.0.200`; query tiếp theo vẫn trả IP này sau khi dừng script, trong thời gian TTL còn hiệu lực |

Task 2/3 dùng tên `www.external.lab9.test`. TTL của bản ghi giả trong Task 3 được tài liệu mô tả là **120 giây**. Khôi phục hosts sau Task 1 và làm sạch cache trước từng thử nghiệm để tránh nhầm nguyên nhân chuyển hướng.

Đối với Task 3, cần kiểm tra cache BIND9 bằng `rndc dumpdb`; file thường là `/var/cache/bind/named_dump.db`, nhưng phải đối chiếu cấu hình `directory` thực tế. Chỉ thấy trang web mô phỏng chưa đủ chứng minh cache resolver đã bị đầu độc.

## 7. Khôi phục và phòng thủ

- Dừng script giả mạo DNS và HTTP server mô phỏng trên Kali.
- Flush cache BIND9 và cache resolver của VICTIM; khôi phục `/etc/hosts` từ bản sao lưu.
- Khi AUTH-DNS còn hoạt động, xác nhận tên miền thử nghiệm đã trở về `172.16.6.10`; sau đó dừng AUTH-DNS khi kết thúc bài.
- Trả Promiscuous Mode của Kali về `Deny`.
- Bật lại DNS COOKIE mặc định bằng cách bỏ ngoại lệ `send-cookie no` cho server mô phỏng, kiểm tra cấu hình rồi restart BIND9.
- Thực hiện lại Task 3 theo bài tập phòng thủ và ghi nhận khác biệt thực tế, không mặc định rằng một thay đổi đơn lẻ chắc chắn ngăn được mọi phản hồi giả.
- Giữ DNSLAB nếu bài sau cần; nếu không, disable đúng interface DNSLAB đã gán hoặc lưu snapshot `LAB9_DONE` theo tài liệu.

Các biện pháp cần phân tích trong báo cáo gồm phân đoạn mạng, kiểm soát DNS egress, giới hạn promiscuous mode, DNS COOKIE, ngẫu nhiên hóa source port, giám sát phản hồi bất thường và DNSSEC validation. DNSSEC xác thực dữ liệu của zone có ký; zone `.test` unsigned trong bài không tự được bảo vệ chỉ vì bật validation.

## 8. Xử lý sự cố thường gặp

| Hiện tượng | Điểm cần kiểm tra |
| --- | --- |
| VICTIM không phân giải được | DNS client, địa chỉ BIND9 lắng nghe, kết nối mạng |
| DNS-SERVER không hỏi AUTH-DNS | Forward zone, cache còn hiệu lực, rule UDP/TCP 53 tới `172.16.6.20` |
| Kali không thấy query | Card capture, Internal Network, Promiscuous Mode, bộ lọc |
| Script không ghi nhận phản hồi giả | Domain/mode/interface sai hoặc client/resolver đang dùng cache |
| VICTIM luôn nhận IP hợp lệ | Thứ tự phản hồi, khả năng quan sát query, độ trễ AUTH-DNS mô phỏng |
| Task 2 đạt nhưng Task 3 chưa đạt | Các trường khớp query/response, cookie, phản hồi có được resolver chấp nhận hay không |
| BIND9 không khởi động | `named-checkconf`, `named-checkzone`, log dịch vụ |
| Không thấy cache dump | Đường dẫn `directory` của BIND9 và file `*dump*` trong `/var/cache/bind` |

## 9. Minh chứng và yêu cầu báo cáo

Các ô bên dưới là checklist cần hoàn thành, không phải xác nhận đã thực hiện:

- [ ] Interface DNSLAB `10.9.0.1/24` và ruleset cô lập trên pfSense.
- [ ] IP DNS-SERVER, VICTIM và Kali đúng mô hình.
- [ ] `named-checkconf` và `named-checkzone` thành công.
- [ ] Baseline `www.external.lab9.test` trả `172.16.6.10`.
- [ ] Task 1: ứng dụng tới IP mô phỏng trong khi truy vấn DNS trực tiếp vẫn trả IP hợp lệ.
- [ ] Task 2: query và forged response trong Wireshark; VICTIM nhận `10.9.0.200`.
- [ ] Task 3: phản hồi giả tới resolver và cache chứa `10.9.0.200`.
- [ ] Sau flush cache, tên miền trở lại `172.16.6.10`.
- [ ] Promiscuous Mode đã trả về `Deny`.

Báo cáo cần trả lời 10 câu hỏi trong DOCX về hosts và DNS, trường khớp phản hồi, phạm vi ảnh hưởng Task 2/3, TTL, promiscuous mode, lý do tách DNSLAB, vai trò các loại DNS server, DNSSEC, hạn chế của việc chỉ chặn UDP/53 và ít nhất bốn biện pháp hardening.

Thực hiện cá nhân, dùng ảnh từ chính VM của sinh viên, phân biệt rõ **baseline**, **attack result** và **restore result**. Không dùng ảnh minh họa trong DOCX làm kết quả thực nghiệm. Tên nộp trong nội dung gốc là `[Mã lớp]-Lab9_MSSV-HoTen.docx`; xác nhận lại số hiệu do tài liệu mang tên LAB 8.
