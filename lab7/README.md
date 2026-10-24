# LAB 7 — Xây dựng chính sách Web Protection với pfSense và Web Proxy

Dự án thực hành môn An toàn hệ thống thông tin, kế thừa môi trường LAB 6 và bổ sung một máy chủ Web Proxy riêng để kiểm soát truy cập web. pfSense đảm nhiệm firewall/gateway và chặn truy cập web trực tiếp; E2Guardian, Squid và ClamAV trên Ubuntu đảm nhiệm lọc nội dung, cảnh báo và quét tệp tải xuống.

README mô tả yêu cầu và tài nguyên triển khai theo tài liệu trong `LAB7 - reading`. Các kết quả bên dưới là **tiêu chí cần kiểm thử**, chưa phải kết quả thực nghiệm đã hoàn thành.

## 1. Mục tiêu

- Phân biệt vai trò firewall L3/L4 và proxy lọc nội dung L7.
- Buộc Client-A/B dùng proxy để truy cập web qua gateway.
- Xây dựng chính sách cho phép, chặn hoàn toàn, cảnh báo và mở tạm thời.
- Kiểm thử truy cập theo khung thời gian 60 phút.
- Quét tệp tải xuống bằng ClamAV với mẫu kiểm thử EICAR.
- Đối chiếu kết quả trên trình duyệt với log firewall và proxy.
- Mở rộng tùy yêu cầu: giới hạn kích thước tải xuống và HTTPS inspection.

Theo thiết kế của bài lab, **không cài Squid/SquidGuard trực tiếp trên pfSense**. Các dịch vụ proxy chạy trên VM Ubuntu riêng.

## 2. Kiến trúc và tài nguyên

| Thành phần | Địa chỉ/cấu hình | Vai trò |
| --- | --- | --- |
| pfSense-LAB6 | MGMT `192.168.56.254`, LAN `10.6.0.1`, DMZ `172.16.6.1` | Gateway, firewall, chống bypass |
| Máy thật quản trị | `192.168.56.1/24` | Quản trị môi trường lab |
| DC01 | `10.6.0.10/24` | Windows Server AD/DNS, zone `lab7.test` |
| Client-A | `10.6.0.21/24` | Máy kiểm thử chính sách |
| Client-B | `10.6.0.22/24` | Máy kiểm thử chính sách |
| DMZ-WEB | `172.16.6.10/24` | Nginx, 13 website mô phỏng và tệp kiểm thử |
| WEB-PROXY | `10.6.0.30/24` | Ubuntu Server 24.04 LTS, E2Guardian, Squid, ClamAV |

WEB-PROXY cần **2 vCPU, 2 GB RAM, 20 GB đĩa**, một card mạng cùng `lab6-lan`, gateway `10.6.0.1` và DNS `10.6.0.10`. Các thành phần còn lại được kế thừa từ LAB 6. Tạo snapshot `LAB7_START` trước khi thay đổi chính sách mạng.

Luồng truy cập web:

```text
Client-A/B (manual proxy)
    -> WEB-PROXY: E2Guardian 10.6.0.30:8080
        -> Squid 127.0.0.1:3128
            -> pfSense
                -> DMZ-WEB / Internet

E2Guardian -> ClamAV daemon để quét nội dung tải xuống
DC01      -> phân giải các tên miền *.lab7.test
```

Squid chỉ lắng nghe trên localhost để client không thể truy cập trực tiếp cổng `3128` và bỏ qua E2Guardian. Client và WEB-PROXY cùng subnet nên kết nối giữa chúng không mặc nhiên đi qua pfSense; cần kiểm tra cấu hình dịch vụ trên chính WEB-PROXY.

## 3. Tài liệu và bộ tài nguyên

Tài liệu gốc nằm ở thư mục `../LAB7 - reading/`:

- `LAB7_pfSense_Web_Protection_2026.docx`: hướng dẫn, mô hình, chính sách, kịch bản kiểm thử và yêu cầu báo cáo.
- `LAB7_pfSense_Web_Protection_2026.zip`: bộ tài nguyên, có thư mục gốc `LAB7_Resources/` bên trong.

Cấu trúc chính **trong gói ZIP**, chưa được giải nén vào thư mục này:

```text
LAB7_Resources/
├── README_FIRST.txt
├── SHA256SUMS.txt
├── 01_DMZ_WEB/          # Script dựng Nginx, website và tệp kiểm thử
├── 02_DNS/              # Script thêm/xóa zone lab7.test trên DC01
├── 03_WEB_PROXY/        # Script cài gói, ghi chú cấu hình, policy lists
│   └── lists/          # Allow, hard block, warning, entertainment, no MITM
├── 04_PFSENSE/          # Alias và ma trận firewall rule
├── 05_REPORT/          # LAB7_Report_Template.docx
└── 06_HTTPS_OPTIONAL/   # Script tạo CA và hướng dẫn HTTPS inspection
```

Tài liệu Word gọi bộ tài nguyên là `LAB7_Resources_Student.zip`; file thực tế được cung cấp có tên `LAB7_pfSense_Web_Protection_2026.zip`.

## 4. Trình tự triển khai

1. **Kiểm tra LAB 6:** IP, gateway, DNS, kết nối LAN/DMZ và snapshot.
2. **Chuẩn bị WEB-PROXY:** tạo VM Ubuntu, đặt mạng tĩnh; chạy `03_WEB_PROXY/install_lab7_proxy.sh` trên VM này để cài các gói cơ bản.
3. **Dựng website mô phỏng:** chạy `01_DMZ_WEB/setup_lab7_dmz_web.sh` trên DMZ-WEB. Script tạo 13 virtual host Nginx và các tệp `10MB.bin`, `100MB.bin`, `110MB.bin`, `eicar.com.txt`.
4. **Cấu hình DNS:** chạy `02_DNS/Add-LAB7-DNS.ps1` trong PowerShell quyền quản trị trên DC01, hoặc tạo thủ công theo yêu cầu giảng viên. Cả 13 bản ghi A đều trỏ về `172.16.6.10`.
5. **Cấu hình proxy:** sao lưu cấu hình hiện tại, ghép nội dung mẫu Squid và áp dụng các thay đổi E2Guardian. Script cài gói không tự hoàn thành toàn bộ chính sách.
6. **Cấu hình client:** đặt manual proxy HTTP/HTTPS là `10.6.0.30:8080`; kiểm tra `http://normal.lab7.test` trước khi ép chính sách firewall.
7. **Cấu hình pfSense:** tạo alias, đặt ruleset đúng thứ tự và Apply Changes. Reset States khi cần kiểm thử lại kết nối đã tồn tại.
8. **Áp dụng chính sách và kiểm thử:** thực hiện từng kịch bản, thu ảnh và log, sau đó hoàn thiện báo cáo.

### Cấu hình proxy trọng tâm

| Thành phần | Thiết lập theo tài nguyên |
| --- | --- |
| Squid | Nghe `127.0.0.1:3128`, chỉ cho localhost dùng upstream proxy |
| E2Guardian main config | `filterip = 10.6.0.30`, `filterports = 8080`, `proxyip = 127.0.0.1`, `proxyport = 3128` |
| Filter group | `reportinglevel = 3`, `bypass = 300`, `contentscanexceptions = off` |
| Antivirus | Bật content scanner `clamdscan.conf`, kiểm tra `clamav-daemon` |
| HTTPS ở phần cơ bản | `enablessl = off`; inspection là phần mở rộng |

Đọc `README_PROXY_SETUP.txt`, hai file `e2guardian*_lab7_changes.txt` và `squid_lab7.conf.fragment` trước khi sửa cấu hình. Các file này là hướng dẫn/đoạn cấu hình mẫu, không phải bản cấu hình đầy đủ để thay thế nguyên file.

### Firewall trên pfSense

Alias cần tạo:

| Alias | Giá trị |
| --- | --- |
| `LAB7_CLIENTS` | `10.6.0.21`, `10.6.0.22` |
| `LAB7_PROXY` | `10.6.0.30` |
| `WEB_PORTS` | `80`, `443` |
| `MAIL_PORTS` | `25`, `465`, `587`, `110`, `995`, `143`, `993` |

Ruleset LAN theo ma trận tài nguyên:

| Thứ tự | Action | Nguồn | Đích | Giao thức/cổng |
| --- | --- | --- | --- | --- |
| 1 | Pass | LAN net | DC01 | TCP/UDP 53 |
| 2 | Pass | LAB7_CLIENTS | LAB7_PROXY | TCP 8080 |
| 3 | Pass | LAN net | Any | TCP MAIL_PORTS |
| 4 | Block | LAB7_CLIENTS | Any | TCP WEB_PORTS |
| 5 | Pass | LAB7_PROXY | Any | TCP WEB_PORTS |
| 6 | Pass | LAB7_PROXY | DMZ-WEB | TCP 80 |
| 7 | Block | LAN net | Any | Any |

Rule 6 đã nằm trong phạm vi cho phép của rule 5; ma trận nguồn vẫn liệt kê riêng để mô tả luồng vào DMZ. Rule cho client đến proxy cùng subnet không thay thế việc giới hạn cổng nghe trên WEB-PROXY.

## 5. Chính sách website

Tất cả tên miền dưới đây thuộc zone `lab7.test` và cùng trỏ về DMZ-WEB.

| Nhóm | Tên miền | Kết quả mong đợi |
| --- | --- | --- |
| Bình thường, giáo dục | `normal.lab7.test`, `education.lab7.test` | Cho phép, không cảnh báo |
| Geo-Sim | `china.lab7.test`, `mideast.lab7.test` | Chặn hoàn toàn |
| Tìm việc | `jobs.lab7.test` | Chặn hoàn toàn |
| Game cụ thể | `blockedgame.lab7.test` | Chặn hoàn toàn |
| Nội dung cảnh báo | `weapons.lab7.test`, `politics.lab7.test`, `religion.lab7.test` | Cảnh báo, cho phép mở tạm 300 giây |
| Streaming | `streaming.lab7.test` | Cảnh báo, cho phép mở tạm 300 giây |
| Giải trí/game | `entertainment.lab7.test`, `games.lab7.test` | Truy cập trong khung 60 phút đã quy định |
| Tải xuống | `download.lab7.test` | Cho phép website, kiểm tra AV và giới hạn kích thước nếu bật |

Hard-block và warning được đưa vào danh sách chặn; hard-block còn cần cơ chế non-bypass phù hợp với package thực tế. Các site cho phép được đưa vào exception list, nhưng phải kiểm thử riêng để bảo đảm tệp tải xuống vẫn được quét AV.

## 6. Phạm vi và các điểm cần kiểm chứng

- **Geo-Sim là mô phỏng theo domain**, không phải bằng chứng đã triển khai GeoIP thật. pfBlockerNG GeoIP là phần mở rộng tùy chọn trong tài liệu.
- **Khung 60 phút không phải quota tích lũy theo ngày.** Tài liệu đề xuất pfSense Schedule hoặc Captive Portal hard timeout. Tuy nhiên, cả 13 website cùng dùng một IP và luồng ra firewall có nguồn là WEB-PROXY: rule IP/port trên pfSense không tự phân biệt riêng hai hostname giải trí. Cần bổ sung cơ chế theo hostname ở proxy hoặc điều chỉnh mô hình nếu muốn giới hạn riêng nhóm này; bộ tài nguyên chưa cung cấp cấu hình hoàn chỉnh cho việc đó.
- **Danh sách non-bypass cần kiểm tra lại trên bản cài đặt.** `README_PROXY_SETUP.txt` nhắc tên `bannedsitelistwithbypass`, trong khi file hướng dẫn filter group chỉ mô tả “non-bypass list”. Không kết luận hard block thành công trước khi xác nhận site thực sự không có khả năng mở tạm.
- **Giới hạn tải xuống là phần mở rộng phụ thuộc build.** `maxcontentfilecachescansize` không mặc nhiên là hard download limit. Đoạn `reply_body_max_size` được cung cấp ở dạng comment; phải kiểm tra cú pháp/cấu hình thực tế và thử cả hai tệp 100/110 MB trước khi ghi nhận đạt.
- **Đơn vị kích thước:** script tạo tệp bằng `dd bs=1M`, tương ứng MiB; ngưỡng mẫu `104857600` byte là 100 MiB, dù tên tệp/tài liệu ghi MB.
- **HTTPS inspection là tùy chọn.** Bộ dựng Nginx hiện tạo website HTTP cổng 80; kiểm thử HTTPS cần chuẩn bị thêm endpoint HTTPS. Chỉ tạo và tin cậy CA trên VM lab do người thực hiện kiểm soát, không nộp private key CA.

## 7. Kiểm thử và minh chứng

| Kịch bản | Tiêu chí kiểm tra |
| --- | --- |
| Baseline | `normal.lab7.test` truy cập được qua proxy |
| Chống bypass | Tắt proxy trên client thì truy cập web trực tiếp qua gateway bị chặn; bật lại proxy thì hoạt động |
| Hard block | Geo-Sim, jobs và blockedgame bị chặn, không mở tạm được |
| Warning | Weapons/politics/religion và streaming có cảnh báo, mở tạm và hết hạn đúng chính sách |
| Education | Truy cập bình thường, không cảnh báo |
| Time window | Kiểm tra trong/ngoài khung 60 phút, xác nhận chính sách chỉ tác động đúng phạm vi dự kiến |
| Antivirus | `/files/eicar.com.txt` bị AV chặn hoặc có log scanner chứng minh phát hiện |
| Kích thước, nếu triển khai | Kiểm tra `/files/100MB.bin` và `/files/110MB.bin`; không ghi đạt nếu cả hai đều tải được |
| HTTPS inspection, nếu triển khai | Chứng minh hành vi chứng chỉ trước/sau khi client tin cậy CA lab |

EICAR là mẫu kiểm thử antivirus, không phải mã độc thực. Mỗi kịch bản cần ảnh cấu hình policy/list/rule, ảnh kết quả từ trình duyệt hoặc lệnh kiểm thử, cùng 1–2 câu giải thích. Dùng ảnh từ môi trường thực hiện, không dùng ảnh mẫu trong tài liệu.

### Log và xử lý sự cố

| Nguồn | Vị trí |
| --- | --- |
| pfSense | `Status -> System Logs -> Firewall` |
| E2Guardian | `/var/log/e2guardian/access.log` (kiểm tra lại theo package) |
| Squid | `/var/log/squid/access.log` |
| ClamAV | `journalctl -u clamav-daemon` hoặc log của package |

Nếu không truy cập được web, kiểm tra lần lượt proxy trên client, kết nối tới `10.6.0.30:8080`, dịch vụ E2Guardian/Squid, DNS và firewall. Nếu trình duyệt lỗi mà E2Guardian không có log, trước tiên xác nhận client thực sự dùng proxy. Nếu tắt proxy vẫn vào web được qua gateway, kiểm tra thứ tự rule và state cũ. Nếu EICAR tải được, kiểm tra scanner, exception list và việc lưu lượng là HTTP hay HTTPS chưa được inspection.

## 8. Sản phẩm cần hoàn thành

- Môi trường lab và các chính sách đã được kiểm thử, kèm ma trận kết quả thực tế.
- Báo cáo Word theo `05_REPORT/LAB7_Report_Template.docx` hoặc định dạng tương đương.
- Tên báo cáo: `[Mã lớp]-Lab7_MSSV-HoTen.docx`.
- Ảnh topology/IP, alias/rule, allow/block/warning, khung thời gian, AV và log tương ứng.
- Trả lời các câu hỏi phân tích trong tài liệu Word về firewall/proxy, bypass, HTTPS/CA, giới hạn kích thước và time-window policy.
- Backup cấu hình pfSense nếu giảng viên yêu cầu; không đưa private key CA vào bài nộp.

Việc chạy script thành công chỉ là bước chuẩn bị. Báo cáo cần giải thích từng chính sách và chứng minh bằng kết quả kiểm thử thực tế.
