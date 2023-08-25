## Chuyển qua lại Rspamd và Amavis
### Chuyển từ dùng Rspamd sang Amavis
- Dừng rspamd Service
```
systemctl stop rspamd 
```

- Sửa file cấu hình `main.cf` của Postfix
    + Mở file
    ```
    vim /etc/postfix/main.cf
    ```
    + Comment những dòng sau
    ```
    
    milter_protocol = 6
    milter_mail_macros = i {mail_addr} {client_addr} {client_name} {auth_authen}
    milter_default_action = accept
    smtpd_milters = inet:127.0.0.1:11332
    non_smtpd_milters = inet:127.0.0.1:11332

    ```
    + Thêm dòng dưới đây vào cuối cùng của file.
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
- Khởi động lại các service để áp dụng cấu hình 
```
systemctl restart postfix
systemctl restart amavis
```

### Chuyển từ dùng Amavis sang Rspamd 
- Dừng Amavis Service
```
systemctl stop amavis 
```

- Chỉnh sửa file cấu hình `main.cf` của Postfix
    + Mở file
    ```
    vim /etc/postfix/main.cf
    ```
    + Comment dòng sau
    ```
    content_filter=smtp-amavis:[127.0.0.1]:10024
    ```
    + Thêm những dòng sau vào cuối file.
    ```
    milter_protocol = 6
    milter_mail_macros = i {mail_addr} {client_addr} {client_name} {auth_authen}
    milter_default_action = accept
    smtpd_milters = inet:127.0.0.1:11332
    non_smtpd_milters = inet:127.0.0.1:11332
    ```
-  Chỉnh sửa file cấu hình `master.cf` của Postfix
    + Mở file
    ```
    vim /etc/postfix/master.cf
    ```
    + Comment những dòng dưới đây

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
- Khởi động lại các service để áp dụng cấu hình 
```
systemctl restart postfix
systemctl restart rspamd
```
