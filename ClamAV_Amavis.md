## Install
### ClamAV
```
sudo apt -y install clamav
sed -i -e "s/^NotifyClamd/#NotifyClamd/g" /etc/clamav/freshclam.conf
systemctl stop clamav-freshclam
freshclam
systemctl start clamav-freshclam
```
### Redis
```
sudo -i 
sudo apt install redis-server
sudo apt update
sudo apt install unbound
sudo echo "nameserver 127.0.0.1" >> /etc/resolvconf/resolv.conf.d/head
sudo resolvconf -u
```
## Amavis and ClamAV
### Install
```
apt -y install clamav-daemon amavisd-new
```
### Config
```
vim /etc/amavis/conf.d/15-content_filter_mode
```
```
# uncomment to enable virus scanning
@bypass_virus_checks_maps = (
   \%bypass_virus_checks, \@bypass_virus_checks_acl, \$bypass_virus_checks_re);

```
```
echo 'dinhha.online' > /etc/mailname
```
- Config with postfix
```
vim /etc/postfix/main.cf
```
```
# add to the end
content_filter=smtp-amavis:[127.0.0.1]:10024
```
```
vim /etc/postfix/master.cf
```
```
scan unix  -       -       n       -       16       smtp
    -o smtp_send_xforward_command=yes
    -o disable_mime_output_conversion=yes
    -o smtp_generic_maps=
    -o smtp_tls_security_level=none
    -o smtpd_tls_security_level=none
    -o smtpd_authorized_xforward_hosts=127.0.0.0/8
    -o receive_override_options=no_unknown_recipient_checks,no_header_body_checks

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
