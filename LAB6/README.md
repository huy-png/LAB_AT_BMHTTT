# LAB6 — Xác thực Active Directory và chính sách truy cập trên pfSense

Lab6 xây dựng môi trường riêng gồm pfSense, Active Directory, hai máy khách và máy chủ Web trong DMZ. Nội dung tập trung vào xác thực LDAP, Captive Portal và kiểm thử chính sách theo địa chỉ IP, dịch vụ và thời gian.

**Tài liệu đề bài:** [LAB6_pfSense_AD_Access_Policies_2026.pdf](../LAB6_pfSense_AD_Access_Policies_2026.pdf).

**Cập nhật tiến độ:** 20/10/2026. Các kết quả mong đợi trong hướng dẫn không được coi là kết quả đã thực hiện.

## 1. Mục tiêu

- Dựng pfSense-LAB6 với bốn interface WAN, MGMT, LAN và DMZ.
- Dùng Active Directory làm nguồn xác thực LDAP cho pfSense và Captive Portal.
- Tạo Alias cho máy khách, nhóm máy khách, website mô phỏng và cổng Web.
- Thiết lập Schedule và rule theo nguyên tắc first-match.
- Kiểm thử chặn site, giới hạn thời gian/dịch vụ, cô lập DMZ, Block/Reject, Firewall Log và State Table.
- Thu thập bằng chứng, hoàn thiện bảng kết quả và giải thích giới hạn của mô hình.

**Giới hạn cần hiểu:** Captive Portal xác thực username AD, nhưng firewall rule chuẩn trong bài vẫn khớp theo IP/port/interface/schedule. Alias CLIENT_A và CLIENT_B đại diện cho IP máy khách, không phải nhóm quyền AD động. Các site mô phỏng bằng IP trong DMZ không tương đương Web Category Filtering thực tế.

## 2. Môi trường và khác biệt so với đề

Đề mô tả VirtualBox; môi trường đang thực hành dùng VMware Workstation. Cần xác nhận giảng viên chấp nhận cách triển khai tương đương và ghi rõ khác biệt trong báo cáo.

| Thành phần | Theo đề | Cấu hình thực hành |
|---|---|---|
| WAN | NAT Network 192.168.250.0/24 | VMnet8 NAT 192.168.190.0/24 |
| MGMT | Host-only 192.168.56.0/24 | VMnet4 Host-only 192.168.56.0/24, DHCP tắt |
| LAN | Internal Network lab6-lan | LAN Segment lab6-lan |
| DMZ | Internal Network lab6-dmz | LAN Segment lab6-dmz |

VMnet3 có subnet 192.168.250.0/24 nhưng được ghi nhận là Host-only, nên không dùng làm WAN NAT. Không thay đổi mạng Lab5 để phục vụ Lab6.

pfSense-LAB6 đã được reset cấu hình trước khi triển khai. DC01-LAB6 là Windows Server riêng; không dùng DC Lab5 thuộc domain vietnam.local làm DC của domain mới.

## 3. Mô hình mạng

```text
Internet
   |
VMnet8 NAT — 192.168.190.0/24
   |
pfSense-LAB6 WAN (em0, DHCP)
   +-- MGMT (em1): 192.168.56.254/24 — VMnet4
   |      +-- Windows thật: 192.168.56.1/24
   |
   +-- LAN (em2): 10.6.0.1/24 — lab6-lan
   |      +-- DC01:     10.6.0.10/24
   |      +-- Client-A: 10.6.0.21/24
   |      +-- Client-B: 10.6.0.22/24
   |
   +-- DMZ (em3): 172.16.6.1/24 — lab6-dmz
          +-- DMZ-WEB: 172.16.6.10/24
                 IP phụ: 172.16.6.20, .30, .40
```

| Thiết bị/interface | Địa chỉ | Gateway | DNS / vai trò |
|---|---|---|---|
| pfSense WAN | DHCP; đã ghi nhận 192.168.190.132/24 | DHCP từ VMware | Có thể thay đổi sau khi cấp lại lease |
| pfSense MGMT | 192.168.56.254/24 | Không đặt upstream gateway | WebGUI quản trị |
| Windows thật VMnet4 | 192.168.56.1/24 | Để trống | Không đặt DNS trên card MGMT |
| pfSense LAN | 10.6.0.1/24 | Không đặt upstream gateway | Gateway LAN-DOMAIN |
| DC01 | 10.6.0.10/24 | 10.6.0.1 | DNS NIC: 10.6.0.10; domain mục tiêu lab6.test |
| Client-A | 10.6.0.21/24 | 10.6.0.1 | DNS 10.6.0.10; đăng nhập thử sv_a |
| Client-B | 10.6.0.22/24 | 10.6.0.1 | DNS 10.6.0.10; đăng nhập thử sv_b |
| pfSense DMZ | 172.16.6.1/24 | Không đặt upstream gateway | Gateway DMZ |
| DMZ-WEB | 172.16.6.10/24 và IP phụ .20/.30/.40 | 172.16.6.1 | Ubuntu Server + Nginx; DNS upstream phù hợp |

Tất cả các mạng nội bộ Lab6 dùng mask **255.255.255.0 (/24)**. DHCP Server trên MGMT/LAN/DMZ tắt; các máy dùng IP tĩnh.

### Mapping card pfSense đã ghi nhận

| Card VMware | Kết nối | Interface pfSense | MAC |
|---|---|---|---|
| Adapter 1 | VMnet8 | WAN / em0 | 00:0c:29:ca:5c:16 |
| Adapter 2 | VMnet4 | MGMT / em1, internal ID opt1 | 00:0c:29:ca:5c:20 |
| Adapter 3 | lab6-lan | LAN / em2 | 00:0c:29:ca:5c:2a |
| Adapter 4 | lab6-dmz | DMZ / em3, internal ID opt2 | 00:0c:29:ca:5c:34 |

Nếu tạo lại/clone VM, đối chiếu MAC trong VMware thay vì mặc định tên em0–em3 luôn giữ nguyên.

## 4. Phân chia thực hành thành ba phần

### Phần 1 — Dựng môi trường riêng (mục A–B)

1. Tạo các mạng WAN, MGMT, LAN và DMZ.
2. Cài/reset pfSense-LAB6, gán bốn interface và đặt IP.
3. Cho phép Windows thật truy cập HTTPS qua MGMT; hoàn tất wizard và đổi mật khẩu admin.
4. Cài Windows Server mới, đặt hostname DC01 và IP 10.6.0.10/24.
5. Cài AD DS + DNS, tạo forest lab6.test và DNS Forwarder.
6. Tạo OU LAB6-Users, user sv_a/sv_b và service account svc_pfsense_ldap.
7. Chuẩn bị Client-A/Client-B và Ubuntu DMZ-WEB với bốn IP website.

**Điều kiện hoàn thành:** quản trị pfSense qua MGMT được; AD/DNS hoạt động; các máy đúng IP/route; bốn địa chỉ website DMZ trả lời HTTP trong phép thử nền tảng có rule cho phép.

### Phần 2 — AD/LDAP, Captive Portal, Alias và Schedule (mục C–F)

1. Thêm LDAP Authentication Server LAB6-AD.
2. Kiểm tra cả sv_a và sv_b tại Diagnostics → Authentication.
3. Tạo Captive Portal Zone LAB6_USERS trên LAN, dùng backend LAB6-AD.
4. Đăng nhập từ hai client và chụp session Captive Portal.
5. Tạo Alias và Schedule theo bảng bên dưới.

**Điều kiện hoàn thành:** hai tài khoản xác thực thành công; có ảnh login/session; Alias và Schedule được kiểm tra trước khi áp chính sách.

### Phần 3 — Chính sách, kiểm thử và báo cáo (mục G–K)

1. Thực hiện bảy kịch bản theo đề.
2. Ghép ruleset tổng hợp đúng thứ tự.
3. Kiểm thử trong/ngoài schedule, ghi kết quả thật và ảnh bằng chứng.
4. Phân tích Firewall Log, State Table và trả lời 12 câu hỏi.
5. Xuất báo cáo Word và cấu hình pfSense cuối cùng.

## 5. Các cấu hình nền tảng

### Quản trị pfSense

- WebGUI từ Windows thật: **https://192.168.56.254**.
- Rule trên MGMT: Pass IPv4 TCP, source 192.168.56.1 → destination 192.168.56.254, port 443.
- OPT1 đã đổi Description thành MGMT; OPT2 thành DMZ. Đổi Description của rule không thay tên interface.
- Hostname/domain trên Dashboard đã ghi nhận: pfSense-Lab6.lab6.test. Đặt tên domain trên pfSense không tự tạo Active Directory.
- Timezone hướng dẫn: Asia/Ho_Chi_Minh hoặc Asia/Bangkok (UTC+7); kiểm tra giờ thật trước khi thử Schedule.

Lệnh đã dùng tại Shell pfSense để mở quyền quản trị:

```sh
easyrule pass opt1 tcp 192.168.56.1 192.168.56.254 443
```

Kiểm tra từ PowerShell trên Windows thật khi cần:

```powershell
Test-NetConnection 192.168.56.254 -Port 443
```

### DC01 và Active Directory

Thực hiện trong PowerShell Administrator **trên máy ảo DC01**:

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
```

Sau khi cài role, trong Server Manager chọn **Promote this server to a domain controller → Add a new forest → lab6.test**. Giữ DNS Server và Global Catalog bật, đặt mật khẩu DSRM riêng và hoàn tất kiểm tra prerequisites. Sau reboot, đăng nhập LAB6\Administrator.

Kiểm tra:

```powershell
Get-ADDomain | Format-List DNSRoot,NetBIOSName,PDCEmulator
Get-Service NTDS,DNS
Resolve-DnsName lab6.test -Server 10.6.0.10
```

DNS NIC của DC chỉ trỏ về 10.6.0.10; cấu hình DNS công cộng trong **DNS Manager → Server Properties → Forwarders**, không đặt làm Alternate DNS trên NIC của DC. DC cần rule cho phép truy vấn upstream qua pfSense.

Trong Active Directory Users and Computers:

- Tạo OU **LAB6-Users** tại gốc domain.
- Tạo **sv_a** và **sv_b** trong OU đó; tài khoản phải được bật và có mật khẩu sử dụng được khi test LDAP.
- Tạo **svc_pfsense_ldap** trong container **Users** để khớp Bind DN bên dưới. Dùng mật khẩu riêng và không thêm vào Domain Admins.
- Không ghi mật khẩu admin, DSRM hoặc LDAP bind vào README/ảnh chụp.

### DMZ-WEB

Dùng Ubuntu Server riêng hoặc bản sao sạch phù hợp, nối lab6-dmz. Không đổi trực tiếp máy huy2222 đang phục vụ Lab5 nếu cần giữ nguyên bài cũ.

Cài Nginx khi máy đã có kết nối Internet phù hợp:

```bash
sudo apt update
sudo apt install -y nginx
ip -br addr
ip route
systemctl status nginx --no-pager
```

Cấu hình bền vững IP chính .10 và các IP phụ .20/.30/.40 bằng Netplan theo đúng tên card thực tế. Lệnh `ip addr add` trong đề chỉ thêm địa chỉ tạm thời, không tự giữ sau reboot. Kiểm tra HTTP trên cả bốn địa chỉ trước khi dùng chúng để đánh giá rule.

## 6. LDAP và Captive Portal

System → User Manager → Authentication Servers → Add:

| Trường | Giá trị |
|---|---|
| Descriptive name | LAB6-AD |
| Type | LDAP |
| Hostname/IP | 10.6.0.10 |
| Port / Transport | 389 / TCP Standard, chỉ trong lab cô lập |
| Protocol version | 3 |
| Search scope | Entire Subtree |
| Base DN | DC=lab6,DC=test |
| Authentication Containers | OU=LAB6-Users,DC=lab6,DC=test |
| Bind DN | CN=svc_pfsense_ldap,CN=Users,DC=lab6,DC=test |
| Initial Template | Microsoft AD |
| User Naming Attribute | samAccountName |
| Group Naming Attribute | cn |
| Group Member Attribute | memberOf |

Chỉ tiếp tục sau khi Diagnostics → Authentication xác thực được cả hai user. Khi triển khai ngoài lab, dùng LDAP được bảo vệ bằng TLS với CA/hostname đúng.

Captive Portal: **Services → Captive Portal → Add Zone → LAB6_USERS**, Enable trên **LAN**, chọn authentication backend **LAB6-AD**. Từ mỗi client mở `http://172.16.6.10`, đăng nhập bằng tài khoản tương ứng và kiểm tra session. Cần rule LAN cho phép luồng thử; đăng nhập portal thành công không thay thế firewall rule. Quản trị tiếp tục qua MGMT.

## 7. Alias và Schedule

| Alias | Loại | Giá trị |
|---|---|---|
| CLIENT_A | Host | 10.6.0.21 |
| CLIENT_B | Host | 10.6.0.22 |
| POLICY_CLIENTS | Hosts | 10.6.0.21, 10.6.0.22 |
| GENERAL_SITE | Host | 172.16.6.10 |
| JOB_SITE | Host | 172.16.6.20 |
| SOCIAL_SITE | Host | 172.16.6.30 |
| RESTRICTED_SITE | Host | 172.16.6.40 |
| WEB_PORTS | Ports | 80, 443 |

Tạo tại Firewall → Aliases; đối chiếu các alias địa chỉ tại Diagnostics → Tables. Dùng IP cố định để kết quả tái lập được, không dùng FQDN/CDN làm mô phỏng nhóm nội dung.

| Schedule | Thời gian |
|---|---|
| LUNCH_WINDOW | Thứ Hai–thứ Sáu, 11:30–13:00 |
| WORK_HOURS | Thứ Hai–thứ Sáu, 08:00–17:00 |
| TEST_NOW | Khoảng 10–15 phút chứa thời điểm kiểm thử |

Schedule không active làm rule đó không được xét; action không tự đổi từ Pass thành Block. Cần rule Block phía dưới. Sau thử nghiệm, khôi phục lịch chuẩn và xóa/disable rule thử TEST_NOW.

## 8. Bảy kịch bản và kết quả mong đợi

Trước mỗi tình huống: kiểm tra rule rộng/default allow có che khuất chính sách không, Apply Changes, tạo kết nối mới; Reset States khi cần. Giữ quyền quản trị MGMT. Các kết quả sau là tiêu chí kiểm thử, chưa phải kết quả đo.

| Kịch bản | Cấu hình chính | Bằng chứng cần có |
|---|---|---|
| 1. Chặn JOB_SITE cho A | LAN: Block CLIENT_A → JOB_SITE TCP/80 trước rule Pass phù hợp | A mở .10 được, .20 bị chặn; log đúng source/destination |
| 2. Chặn Restricted | Block POLICY_CLIENTS → RESTRICTED_SITE TCP/80; Pass tới GENERAL_SITE | Cả A/B bị chặn .40 nhưng mở .10 được |
| 3. Social theo giờ | Pass CLIENT_B → SOCIAL_SITE TCP/80 + LUNCH_WINDOW; Block cùng luồng phía dưới | Trong lịch mở .30 được, ngoài lịch bị chặn; dùng TEST_NOW để thử nhanh |
| 4. Web giờ hành chính | Pass POLICY_CLIENTS → Any TCP/WEB_PORTS + WORK_HOURS; Block POLICY_CLIENTS → Any phía dưới | Trong giờ HTTP/HTTPS được, ping/dịch vụ khác bị chặn; ngoài giờ Web bị chặn trong phép thử riêng |
| 5. Cô lập DMZ | DMZ: Block DMZ net → LAN net; LAN: Pass client → GENERAL_SITE theo chính sách | Client khởi tạo Web tới DMZ được; kết nối mới từ DMZ tới DC bị chặn |
| 6. Block và Reject | Thử cùng source/destination TCP lần lượt với Block rồi Reject | Ghi thời gian và lỗi thực tế; Reject thường báo lỗi nhanh hơn |
| 7. Log và State | Bật log, tạo một phép thử mới, lọc theo IP client | Đối chiếu interface/action/IP/port; chụp state và thử lại sau xóa state |

Với kịch bản 5, nên xác minh dịch vụ đích có phản hồi trước khi thêm Block để loại trừ lỗi host firewall. Không dùng traffic trả lời cho kết nối do LAN khởi tạo để kết luận DMZ được phép khởi tạo vào LAN; pfSense là stateful.

Client dùng DNS DC cùng subnet, nên lưu lượng client → DC không qua pfSense. DC vẫn phải được phép truy vấn DNS upstream. Khi dùng HTTPS làm minh chứng, cần máy đích thật sự cung cấp HTTPS; cài Nginx mặc định chỉ chứng minh HTTP.

## 9. Ruleset LAN tổng hợp

| Ưu tiên | Action | Source | Destination | Protocol/port | Schedule |
|---|---|---|---|---|---|
| 1 | Block + Log | CLIENT_A | JOB_SITE | TCP/80 | Always |
| 2 | Block + Log | POLICY_CLIENTS | RESTRICTED_SITE | TCP/80 | Always |
| 3 | Pass | CLIENT_B | SOCIAL_SITE | TCP/80 | LUNCH_WINDOW |
| 4 | Block | CLIENT_B | SOCIAL_SITE | TCP/80 | Always |
| 5 | Pass | POLICY_CLIENTS | GENERAL_SITE | TCP/80 | Always |
| 6 | Pass | POLICY_CLIENTS | Any | TCP/WEB_PORTS | WORK_HOURS |
| 7 | Block + Log | POLICY_CLIENTS | Any | Any | Always |

Rule cho các dịch vụ nền tảng của DC phải được xử lý riêng, không mở rộng quyền cho POLICY_CLIENTS ngoài thiết kế. GENERAL_SITE được phép ngoài giờ trong ruleset tổng hợp do rule 5; cần phân biệt với phép thử kịch bản 4 độc lập.

## 10. Tiến độ đã xác nhận

| Hạng mục | Trạng thái theo bằng chứng đã gửi |
|---|---|
| VMnet4 MGMT | Windows thật có 192.168.56.1/24 |
| pfSense-LAB6 | Đã reset, gán 4 card và đặt đúng IP LAN/MGMT/DMZ |
| WebGUI MGMT | Đã đăng nhập qua 192.168.56.254; rule TCP/443 đúng |
| Tên interface | Menu đã hiển thị WAN, LAN, MGMT, DMZ |
| DC01 Windows Server | Đã cài, hostname DC01, IP 10.6.0.10/24, DNS NIC 10.6.0.10 |
| Gateway DC01 | Đã sửa active/persistent default route về 10.6.0.1 |
| Kết nối DC01 | Ping 10.6.0.1 và 8.8.8.8 đều nhận 4/4 gói |
| AD DS/DNS và forest lab6.test | Đã hướng dẫn; chưa nhận kết quả xác nhận hoàn thành |
| OU, user và LDAP bind account | Chưa có bằng chứng hoàn thành |
| Client-A/B và DMZ-WEB Lab6 | Chưa có bằng chứng hoàn thành |
| LDAP, Portal, Alias, Schedule, 7 kịch bản | Chưa có kết quả xác nhận |

**Bước tiếp theo:** hoàn tất promote DC01 và gửi kết quả `Get-ADDomain`, `Get-Service NTDS,DNS`, `Resolve-DnsName lab6.test -Server 10.6.0.10`; sau đó cấu hình Forwarders và tạo tài khoản.

## 11. Xử lý sự cố

| Hiện tượng | Thứ tự kiểm tra |
|---|---|
| Không mở WebGUI | pfSense bật → VMnet4/IP host → IP MGMT → rule TCP/443 → Test-NetConnection |
| DC ping gateway được nhưng không ra ngoài | Default route → rule LAN → NAT → WAN; ping cùng subnet không chứng minh gateway đúng |
| LDAP thất bại | Route/port 389 → Base DN/OU → Bind DN/password → trạng thái user → log |
| Portal không hiện | Interface/zone → HTTP test → session cũ → DNS/IPv6 |
| Rule không chặn | Đúng interface → thứ tự → alias/IP/port → state cũ → log |
| Schedule sai | Giờ/timezone → Active/Inactive → rule Block phía dưới |
| DMZ site không mở | Nginx → IP phụ → gateway → rule → host firewall |

## 12. Yêu cầu nộp bài

- [ ] Báo cáo cá nhân: **[Mã lớp]-Lab6_MSSV-TenSV.docx**.
- [ ] Ảnh bốn adapter pfSense và topology; nêu rõ việc dùng VMware và WAN khác subnet mẫu.
- [ ] Ảnh cấu hình IP/interface, DC và các máy thực hành.
- [ ] LDAP test thành công cho cả sv_a và sv_b.
- [ ] Captive Portal login và session.
- [ ] Alias và Schedule.
- [ ] Mỗi kịch bản ít nhất một ảnh rule và một ảnh kiểm thử.
- [ ] Firewall Log và State Table.
- [ ] Bảng kết quả thực tế đầy đủ, gồm phép thử trong/ngoài schedule.
- [ ] Trả lời đủ 12 câu hỏi phân tích của đề.
- [ ] Xuất **Diagnostics → Backup & Restore → Download configuration**, nộp kèm **config.xml** cuối cùng.

Config.xml có thể chứa thông tin cấu hình và bí mật xác thực; chỉ chuyển qua kênh nộp bài được chỉ định, không đưa lên kho mã công khai. Ảnh minh chứng phải lấy từ máy thực hành, không hiển thị mật khẩu.
