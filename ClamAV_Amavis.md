## Update
```
sudo apt update
```
## ClamAV
- Bước 1: 
    + Cài đặt ClamAV từ Repository
    ```
    sudo apt -y install clamav clamav-daemon
    ```

- Bước 2 : Cập nhật thủ công các cơ sở dữ liệu mẫu Virus
    + Cấu hình freshclam.conf
    ```
    sed -i -e "s/^NotifyClamd/#NotifyClamd/g" /etc/clamav/freshclam.conf
    ```
    + Dừng ClamAV Service
    ```
    systemctl stop clamav-freshclam
    ```
    + Cập nhật thủ công ClamAV Signature Database
    ```
    freshclam
    ```
    + Khởi động ClamAV Service
    ```
    systemctl start clamav-freshclam
    ```
- Bước 3 : Tải thử mẫu virus để kiểm tra hoạt động của ClamAV
    + Đây là một mẫu thử chỉ chứa dấu hiệu (signature) phổ biến không tồn tại mã độc
    ```
    wget -O- http://www.eicar.org/download/eicar.com.txt | clamscan -
    ```
    + Kết quả trả về đã quét được virus 
    ![image](https://github.com/trangnth/BizflyCloudEmail/assets/119484840/fc6b07a5-a857-482e-abba-433b38aa92e0)

## Amavis and ClamAV
- Bước 1 : Cài đặt Amavis từ Repository
```
sudo apt -y install  amavisd-new
```
- Bước 2 : Cấu hình Amavis 
    +  Khởi tạo tính năng `virus-checking` trên Amavis
    ```
    vim /etc/amavis/conf.d/15-content_filter_mode
    ```
    Bỏ comment những dòng dưới để bật tính năng 
    ```
    @bypass_virus_checks_maps = (
       \%bypass_virus_checks, \@bypass_virus_checks_acl, \$bypass_virus_checks_re);
    ```

- Bước 3 : Tích hợp Amavis với Postfix
    + Chỉnh sửa file cấu hình `main.cf` của Postfix
    ```
    vim /etc/postfix/main.cf
    ```
    Thêm dòng dưới đây vào cuối cùng của file.
    Điều này yêu cầu Postfix bật tính năng lọc nội dung bằng cách gửi mọi thư email đến tới Amavis, nơi sẽ tiếp tục lắng nghe trên port 10024
    ```
    content_filter=smtp-amavis:[127.0.0.1]:10024
    ```
    +  Chỉnh sửa file cấu hình `master.cf` của Postfix
    ```
    vim /etc/postfix/master.cf
    ```
    Thêm dòng dưới đây vào cuối cùng của file.
    Điều này giúp Postfix và Amavis trao đổi mail qua lại với nhau.
    ```
    smtp-amavis   unix   -   -   n   -   2   smtp
        -o syslog_name=postfix/amavis
        -o smtp_data_done_timeout=1200
        -o smtp_send_xforward_command=yes
        -o disable_dns_lookups=yes
        -o max_use=20
        -o smtp_tls_security_level=none

    127.0.0.1:10025   inet   n    -     n     -     -    smtpd
        -o syslog_name=postfix/10025
        -o content_filter=
        -o mynetworks_style=host
        -o mynetworks=127.0.0.0/8
        -o local_recipient_maps=
        -o relay_recipient_maps=
        -o strict_rfc821_envelopes=yes
        -o smtp_tls_security_level=none
        -o smtpd_tls_security_level=none
        -o smtpd_restriction_classes=
        -o smtpd_delay_reject=no
        -o smtpd_client_restrictions=permit_mynetworks,reject
        -o smtpd_helo_restrictions=
        -o smtpd_sender_restrictions=
        -o smtpd_recipient_restrictions=permit_mynetworks,reject
        -o smtpd_end_of_data_restrictions=
        -o smtpd_error_sleep_time=0
        -o smtpd_soft_error_limit=1001
        -o smtpd_hard_error_limit=1000
        -o smtpd_client_connection_count_limit=0
        -o smtpd_client_connection_rate_limit=0
        -o receive_override_options=no_header_body_checks,no_unknown_recipient_checks,no_address_mappings
    ```
- Bước 4 : Khởi động lại các service để áp dụng cấu hình 
```
systemctl restart clamav-daemon
systemctl restart postfix
systemctl restart amavis
```
