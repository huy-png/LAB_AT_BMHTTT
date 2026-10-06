# LAB5 — Cấu hình tường lửa pfSense

Bài thực hành xây dựng mạng WAN–LAN–DMZ trên VMware, cấu hình NAT và kiểm thử các chính sách firewall bằng Windows Server và Ubuntu.

## 1. Tài liệu trong thư mục

| Tài liệu | Nội dung |
|---|---|
| [LAB5_BaoCao_pfSense.docx](LAB5_BaoCao_pfSense.docx) | Báo cáo Word: cấu hình, ảnh minh chứng, các tình huống C1–C5 và câu hỏi phân tích |
| [Tài liệu đề bài](../LAB5-pfSense/Lab3-pfSense_SAMPLE_STYLE_SPLIT_FIGURES_v6_FINAL_TEXT_ONLY.docx) | Yêu cầu thực hành và nộp bài |
| [README của bộ tài liệu](../LAB5-pfSense/README.txt) | Hướng dẫn môi trường, bộ cài và các lưu ý kiểm thử |

Tài liệu nguồn còn dùng nhãn **Lab3** ở một số vị trí; thư mục và báo cáo thực hành này dùng **Lab5**. Kiểm tra mẫu tên file với giảng viên trước khi nộp.

## 2. Mục tiêu

- Thiết lập pfSense với ba interface WAN, LAN và DMZ.
- Cấu hình Windows Server làm Domain Controller/DNS cho domain `vietnam.local`.
- Dùng Windows Server riêng chạy IIS trong DMZ và Ubuntu làm máy LAN-Test.
- Phân biệt chức năng dịch địa chỉ của NAT với chức năng cho phép/chặn của firewall rule.
- Kiểm thử ICMP, DNS, HTTP, HTTPS, giới hạn theo IP nguồn, cô lập DMZ và Port Forward.
- Đọc firewall log và lưu bằng chứng thực hành.

## 3. Mô hình và địa chỉ IP

```text
Internet / router mạng vật lý
            |
    VMware VMnet0 (Bridged)
            |
      pfSense WAN em0 — DHCP
            |
            +-- LAN em1: 10.0.0.1/8 — VMnet2
            |       +-- Windows thật: 10.0.0.100/8 (quản trị)
            |       +-- Windows Server DC/DNS: 10.0.0.2/8
            |       +-- Ubuntu LAN-Test: 10.0.0.3/8
            |
            +-- DMZ em2: 172.16.0.1/16 — LAN segment dmz-net
                    +-- Windows Server DMZ-Web/IIS: 172.16.0.2/16
```

| Thiết bị | IP / mask | Gateway | DNS / ghi chú |
|---|---|---|---|
| pfSense WAN | DHCP; từng ghi nhận `192.168.1.190/24` | Nhận qua DHCP | Xác nhận lại IP trước khi thử Port Forward |
| pfSense LAN | `10.0.0.1/8` | Không đặt upstream gateway | Quản trị tại `https://10.0.0.1` |
| pfSense DMZ | `172.16.0.1/16` | None | OPT1 đổi tên thành DMZ |
| Windows thật, VMnet2 | `10.0.0.100/8` | Để trống | Không đặt DNS trên adapter Host-only |
| Windows Server DC | `10.0.0.2/8` | `10.0.0.1` | NIC dùng `10.0.0.2`; DNS Forwarder cấu hình trong DNS Manager |
| Ubuntu LAN-Test | `10.0.0.3/8` | `10.0.0.1` | DNS thử nghiệm `8.8.8.8`; card đã ghi nhận là `ens33` |
| Windows Server DMZ-Web | `172.16.0.2/16` | `172.16.0.1` | DNS `8.8.8.8` khi rule DMZ cho phép |

`/8` tương ứng `255.0.0.0`; `/16` tương ứng `255.255.0.0`.

DC và DMZ-Web là **hai máy ảo riêng** để có thể kiểm thử lưu lượng qua pfSense. Không clone trực tiếp DC đã promote để làm máy thử nghiệm. VMnet2 là Host-only; mạng DMZ phải nối cùng LAN segment trên pfSense và DMZ-Web. Tránh dùng mạng khác trùng với LAN hoặc DMZ.

## 4. Môi trường thực hành

- VMware Workstation trên máy Windows thật.
- pfSense CE 2.7.2 theo bộ tài liệu lab; phiên bản này dành cho môi trường thực hành theo đề.
- Windows Server 2022 Standard Evaluation cho DC; Windows Server riêng cho DMZ-Web.
- Ubuntu Server cho LAN-Test.
- Gợi ý RAM: pfSense 2 GB, DC 2–4 GB, DMZ-Web 2 GB, LAN-Test 1 GB. Chỉ bật các máy cần cho phép thử.

## 5. Các bước cấu hình nền tảng

1. Cài pfSense, gán WAN/LAN/DMZ đúng card mạng và đặt địa chỉ theo bảng trên.
2. Truy cập WebGUI từ Windows thật, hoàn tất wizard và đặt mật khẩu quản trị.
3. Bật interface DMZ; chọn Static IPv4, địa chỉ `172.16.0.1/16`, upstream gateway None.
4. Kiểm tra Outbound NAT tự động có mạng nguồn `10.0.0.0/8` và `172.16.0.0/16`, dịch sang WAN address.
5. Cấu hình DC `10.0.0.2`, cài AD DS/DNS, tạo forest `vietnam.local`; đặt DNS Forwarder trong DNS Manager.
6. Tạo rule LAN nền tảng Pass IPv4 Any, Source LAN subnets, Destination Any để kiểm tra Internet ban đầu.
7. Cấu hình Ubuntu LAN-Test `10.0.0.3/8`; kiểm tra bằng `ip -br addr` và `ip route`.
8. Cấu hình DMZ-Web `172.16.0.2/16`, cài IIS và kiểm tra `http://localhost`.

Lệnh cài và kiểm tra IIS, chạy trong PowerShell Administrator trên DMZ-Web:

```powershell
Install-WindowsFeature Web-Server -IncludeManagementTools
ipconfig
Get-WindowsFeature Web-Server
Get-Service W3SVC
curl.exe -I http://localhost
```

## 6. Nguyên tắc khi kiểm thử rule

- Giữ **Anti-Lockout Rule** bật để duy trì quản trị từ LAN.
- Tắt Default allow LAN IPv4/IPv6 và rule Pass LAN rộng khi thử các chính sách hạn chế.
- Tạm tắt các rule của tình huống trước nếu chúng ảnh hưởng tình huống đang thử.
- Các rule interface thông thường khớp từ trên xuống; đặt rule cụ thể trước rule tổng quát.
- Sau khi thay đổi: **Save → Apply Changes → Diagnostics → States → Reset States**, rồi tạo phép thử mới. Reset States ngắt các kết nối hiện có đi qua pfSense.
- Kiểm thử từ đúng máy nguồn, không dùng ping trong Diagnostics của pfSense thay cho ping từ máy khách.
- NAT không tự cấp quyền đi qua firewall. DMZ cần rule Pass phù hợp dù đã có Outbound NAT.

## 7. Các tình huống thực hành

Các kết quả dưới đây là **tiêu chí mong đợi**, không phải xác nhận rằng mọi tình huống đã hoàn thành.

### C1 — Chặn ICMP, cho phép DNS và Web

Trên tab LAN tạo theo thứ tự, đều là IPv4, Source LAN subnets, Destination Any:

1. Block ICMP.
2. Pass TCP/UDP port 53.
3. Pass TCP port 80.
4. Pass TCP port 443.

Không dùng dải port 80–443. Chụp bảng rule làm **C1a**; chạy trên DC để lấy **C1b**:

```powershell
ping -4 -n 4 -w 2000 8.8.8.8
Resolve-DnsName example.com -Server 8.8.8.8 -Type A -DnsOnly
curl.exe -4 -I --connect-timeout 10 --max-time 15 http://example.com
curl.exe -4 -I --connect-timeout 10 --max-time 15 https://example.com
```

Mong đợi: ping mất 100% gói; DNS trả về IP; HTTP/HTTPS có phản hồi. Lỗi `curl: (6)` là lỗi phân giải tên, chưa chứng minh riêng TCP/443 bị chặn.

### C2 — Chỉ cho DC ra Internet

Trên tab LAN, tắt các rule C1 và tạo:

1. Pass IPv4 Any, Source `10.0.0.2`, Destination Any.
2. Block IPv4 Any, Source LAN subnets, Destination Any.

**C2a:** ảnh rule Pass DC nằm trên Block LAN.

**C2b:** ảnh hai phép thử:

```powershell
# Windows Server DC
ipconfig
ping -4 -n 4 -w 2000 8.8.8.8
```

```bash
# Ubuntu LAN-Test
ip -br addr
ip route
ping -4 -c 4 -W 2 8.8.8.8
```

Mong đợi: DC thành công, LAN-Test thất bại. Không dùng Windows thật làm máy nguồn thứ hai vì máy thật có đường Internet riêng qua Wi-Fi.

### C3 — Cô lập DMZ khỏi LAN

1. Trên DC, cho phép tạm ICMP Echo từ DMZ bằng PowerShell Administrator:

```powershell
New-NetFirewallRule -DisplayName "LAB5-C3-Allow-Ping-DMZ" -Direction Inbound -Protocol ICMPv4 -IcmpType 8 -RemoteAddress 172.16.0.0/16 -Action Allow
```

2. Trên tab DMZ, ban đầu chỉ bật Pass IPv4 Any, DMZ subnets → Any. Apply Changes và Reset States.
3. Từ DMZ-Web, ping DC `10.0.0.2` và Internet `8.8.8.8`. **Phải có phản hồi trước khi thêm Block**; lưu ảnh baseline.
4. Thêm Block IPv4 Any, DMZ subnets → LAN subnets ở **trên** Pass DMZ → Any. Apply Changes và Reset States.
5. Trên DMZ-Web chạy:

```powershell
ping -4 -n 4 -w 2000 10.0.0.2
ping -4 -n 4 -w 2000 8.8.8.8
Resolve-DnsName example.com -Server 8.8.8.8 -Type A -DnsOnly
```

Mong đợi: DC không còn phản hồi; Internet và DNS vẫn hoạt động. **C3a:** rule DMZ; **C3b:** ảnh trước/sau và kiểm tra Internet. Nếu ping DC thất bại ngay từ baseline thì chưa thể kết luận pfSense cô lập thành công.

Sau khi chụp đủ bằng chứng, xóa rule tạm trên DC:

```powershell
Remove-NetFirewallRule -DisplayName "LAB5-C3-Allow-Ping-DMZ"
```

### C4 — Port Forward WAN vào Web trong DMZ

- Kiểm tra IIS hoạt động trên DMZ-Web bằng `curl.exe -I http://localhost`.
- Firewall → NAT → Port Forward: WAN, TCP, Destination WAN address, port 8080 → `172.16.0.2`, port 80.
- Chọn **Add associated filter rule**, Save và Apply Changes.
- Với WAN private của mô hình lab, cấu hình tùy chọn chặn mạng private/bogon theo đề để máy thật có thể kiểm thử vào WAN.
- Từ Windows thật truy cập `http://<IP-WAN-pfSense>:8080`; dùng IP WAN hiện tại, không mặc định IP cũ luôn đúng.
- **C4a:** NAT và rule WAN đi kèm; **C4b:** trang IIS qua WAN:8080.

### C5 — Logging và phân tích firewall log

- Bật **Log packets that are handled by this rule** trên rule Block cần kiểm tra.
- Apply Changes, Reset States nếu cần và tạo lưu lượng mới khớp rule đó.
- Vào **Status → System Logs → Firewall**.
- Đối chiếu thời gian, interface, action, protocol, IP nguồn/đích và rule/tracker.
- **C5a:** rule bật logging; **C5b:** bản ghi tương ứng phép thử.

## 8. Ảnh minh chứng trong báo cáo

| Hình | Nội dung cần có | Nơi lấy ảnh |
|---|---|---|
| B1–B2 | Ba card mạng; console pfSense sau đặt LAN | VMware / console pfSense |
| B3–B6 | Dashboard, gán interface, bật và đặt IP DMZ | pfSense WebGUI |
| B7 | IP, DNS NIC và thông tin domain/DC | Windows Server DC |
| B8 | DNS Forwarders | DNS Manager trên DC |
| B9 | Outbound NAT cho LAN và DMZ | pfSense |
| B10–B11 | Rule LAN, Apply Changes và Reset States | pfSense |
| B12–B13 | Kiểm thử trước/sau tắt rule LAN | Windows Server DC |
| B14 | IP DMZ-Web, IIS đã cài và trang localhost | Windows Server DMZ-Web |
| B15 | IP `10.0.0.3/8`, default route và kết quả kiểm tra kết nối | Ubuntu LAN-Test |
| C1a–C5b | Rule, phép thử và giải thích tương ứng | pfSense và đúng máy nguồn |

## 9. Tiến độ đã có bằng chứng trong cuộc trao đổi

- pfSense đã cài, có LAN/DMZ và Outbound NAT; truy cập WebGUI được từ máy thật.
- DC chạy Windows Server 2022; DNS nội bộ `vietnam.local` trả về `10.0.0.2`; đã có kết quả DNS Internet và HTTPS 200 OK.
- Khi tắt các rule cho phép LAN: ping Internet mất 100% gói và curl báo lỗi phân giải tên.
- Ubuntu đã đặt `10.0.0.3/8`, gateway `10.0.0.1`; SSH đã được bật. Ảnh ping được gửi chưa có thống kê kết thúc nên chưa xác nhận phép thử đó thành công.
- Đã có ảnh cài IIS thành công, không cần restart; cần đối chiếu ảnh IP DMZ-Web và HTTP localhost để đủ B14.
- Đã có ảnh bảng rule C1 đúng cấu trúc. Chưa có kết quả C1b, C2, C3, C4, C5 trong cuộc trao đổi để xác nhận đạt.

Phần tiến độ này chỉ phản ánh bằng chứng đã gửi, không suy đoán kết quả các bước mới được hướng dẫn. Cập nhật README và báo cáo sau khi hoàn thành từng phép thử.

## 10. Hoàn thiện và nộp bài

- [ ] Điền họ tên, MSSV, lớp, giảng viên và ngày nộp.
- [ ] Bổ sung ảnh cấu hình và kiểm thử nền tảng B1–B15.
- [ ] Hoàn thành ít nhất **4 tình huống firewall** theo đề, mỗi tình huống có ảnh rule, ảnh kiểm thử và giải thích ngắn.
- [ ] Có baseline DMZ → LAN thành công và kết quả sau Block thất bại cho phần cô lập DMZ.
- [ ] Bổ sung bằng chứng các rule mặc định đã tắt và Reset States khi kiểm thử tắt LAN.
- [ ] Đối chiếu 6 câu hỏi phân tích trong phần D với kết quả thực tế.
- [ ] Thay các chỗ `[Điền]`, `[CHÈN ẢNH...]`; không ghi kết quả mong đợi thành kết quả đã đo.
- [ ] Kiểm tra ảnh rõ chữ, ngắt trang và tên file theo yêu cầu giảng viên.

Không đưa mật khẩu quản trị hoặc DSRM vào ảnh và báo cáo.
