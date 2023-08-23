## Install 
### Rspamd
```
sudo apt install software-properties-common lsb-release
sudo apt install lsb-release wget
wget -O- https://rspamd.com/apt-stable/gpg.key | sudo apt-key add -
echo "deb http://rspamd.com/apt-stable/ $(lsb_release -cs) main" | sudo tee -a /etc/apt/sources.list.d/rspamd.list
sudo apt update
sudo apt install rspamd
```
## Config
### Rspamd
- /etc/rspamd/local.d/worker-normal.inc
```
vim /etc/rspamd/local.d/worker-normal.inc
```
 
```
bind_socket = "127.0.0.1:11333";
```

- /etc/rspamd/local.d/worker-proxy.inc
```
vim /etc/rspamd/local.d/worker-proxy.inc
```
 
```
bind_socket = "127.0.0.1:11332";
milter = yes;
timeout = 120s;
upstream "local" {
  default = yes;
  self_scan = yes;
}
```
- /etc/rspamd/local.d/worker-controller.inc
```
vim /etc/rspamd/local.d/worker-controller.inc
```
  
```
password = "$2$khz7u8nxgggsfay3qta7ousbnmi1skew$zdat4nsm7nd3ctmiigx9kjyo837hcjodn1bob5jaxt7xpkieoctb";
```
- /etc/rspamd/local.d/classifier-bayes.conf
```
vim /etc/rspamd/local.d/classifier-bayes.conf
```
 
```
servers = "127.0.0.1";
backend = "redis";
```
- /etc/rspamd/local.d/milter_headers.conf
```
vim /etc/rspamd/local.d/milter_headers.conf
```
 
```
use = ["x-spamd-bar", "x-spam-level", "authentication-results"];
```
- Restart
``` sudo systemctl restart rspamd ```
### Postfix 

```
sudo postconf -e "milter_protocol = 6"
sudo postconf -e "milter_mail_macros = i {mail_addr} {client_addr} {client_name} {auth_authen}"
sudo postconf -e "milter_default_action = accept"
sudo postconf -e "smtpd_milters = inet:127.0.0.1:11332"
sudo postconf -e "non_smtpd_milters = inet:127.0.0.1:11332"
```
- Restart
```sudo systemctl restart postfix```


## Tích hợp Clamav vs RSpamd
- /etc/clamav/clamd.conf
```
vim /etc/clamav/clamd.conf
```

```
TCPSocket 3310
TCPAddr 127.0.0.1
# TemporaryDirectory is not set to its default /tmp here to make overriding
# the default with environment variables TMPDIR/TMP/TEMP possible
User clamav
ScanMail true
ScanArchive true
ArchiveBlockEncrypted false
MaxDirectoryRecursion 15
FollowDirectorySymlinks false
FollowFileSymlinks false
ReadTimeout 180
MaxThreads 12
MaxConnectionQueueLength 15
LogSyslog true
LogRotate true
LogFacility LOG_LOCAL6
LogClean false
LogVerbose false
PreludeEnable no
PreludeAnalyzerName ClamAV
DatabaseDirectory /var/lib/clamav
OfficialDatabaseOnly false
SelfCheck 3600
Foreground false
Debug false
ScanPE true
MaxEmbeddedPE 10M
ScanOLE2 true
ScanPDF true
ScanHTML true
MaxHTMLNormalize 10M
MaxHTMLNoTags 2M
MaxScriptNormalize 5M
MaxZipTypeRcg 1M
ScanSWF true
ExitOnOOM false
LeaveTemporaryFiles false
AlgorithmicDetection true
ScanELF true
IdleTimeout 60
CrossFilesystems true
PhishingSignatures true
PhishingScanURLs true
PhishingAlwaysBlockSSLMismatch false
PhishingAlwaysBlockCloak false
PartitionIntersection false
DetectPUA false
ScanPartialMessages false
HeuristicScanPrecedence false
StructuredDataDetection false
CommandReadTimeout 60
SendBufTimeout 200
MaxQueue 100
ExtendedDetectionInfo true
OLE2BlockMacros false
AllowAllMatchScan true
ForceToDisk false
DisableCertCheck false
DisableCache false
MaxScanTime 120000
MaxScanSize 100M
MaxFileSize 25M
MaxRecursion 16
MaxFiles 10000
MaxPartitions 50
MaxIconsPE 100
PCREMatchLimit 10000
PCRERecMatchLimit 5000
PCREMaxFileSize 25M
ScanXMLDOCS true
ScanHWP3 true
MaxRecHWP3 16
StreamMaxLength 50M
LogFile /var/log/clamav/clamav.log
LogTime true
LogFileUnlock false
LogFileMaxSize 0
Bytecode true
BytecodeSecurity TrustSigned
BytecodeTimeout 60000
OnAccessMaxFileSize 5M
```
- /etc/rspamd/local.d/antivirus.conf
```
vim /etc/rspamd/local.d/antivirus.conf
```

```
# local.d/antivirus.conf
enabled = true

# multiple scanners could be checked, for each we create a configuration block with an arbitrary name
clamav {
  # If set force this action if any virus is found (default unset: no action is forced)
  action = "reject";
  message = '${SCANNER}: virus found: "${VIRUS}"';
  # Scan mime_parts seperately - otherwise the complete mail will be transfered to AV Scanner
  #attachments_only = true; # Before 1.8.1
  #scan_mime_parts = true; # After 1.8.1
  # Scanning Text is suitable for some av scanner databases (e.g. Sanesecurity)
  #scan_text_mime = false; # 1.8.1 +
  #scan_image_mime = false; # 1.8.1 +
  # If `max_size` is set, messages > n bytes in size are not scanned
  #max_size = 20000000;
  # symbol to add (add it to metric if you want non-zero weight)
  symbol = "CLAM_VIRUS";
  # type of scanner: "clamav", "fprot", "sophos" or "savapi"
  type = "clamav";
  # If set true, log message is emitted for clean messages
  #log_clean = false;
  # Prefix used for caching in Redis: scanner-specific defaults are used. If Redis is enabled and
  # multiple scanners of the same type are present, it is important to set prefix to something unique.
  #prefix = "rs_cl_";
  # For "savapi" you must also specify the following variable
  #product_id = 12345;
  # servers to query (if port is unspecified, scanner-specific default is used)
  # can be specified multiple times to pool servers
  # can be set to a path to a unix socket
  servers = "127.0.0.1:3310";
  timeout = 30;
  # if `patterns` is specified virus name will be matched against provided regexes and the related
  # symbol will be yielded if a match is found. If no match is found, default symbol is yielded.
  patterns {
    # symbol_name = "pattern";
    JUST_EICAR = '^Eicar-Test-Signature$';
  }
  # In version 1.7.0+ patterns could be extended
  #patterns = {SANE_MAL = 'Sanesecurity\.Malware\.*', CLAM_UNOFFICIAL = 'UNOFFICIAL$'};
  # `whitelist` points to a map of signature names. Hits on these signatures are ignored.
  whitelist = "/etc/rspamd/antivirus.wl";
}
```

## Web check log RSpamd
http://hostname/rspamd

pass : P4ssvv0rD
## Remove Amavis ( Rspamd and Clamav )
- Stop Amavis
```
 systemctl stop amavis.service
 ```
 
 - /etc/postfix/main.cf
 ```
 vim /etc/postfix/main.cf
 ```
 Comment 
 ```
content_filter = scan:[127.0.0.1]:10024
content_filter = smtp-amavis:[127.0.0.1]:10024
 ```
 Add 
 ```
milter_protocol = 6
milter_mail_macros = i {mail_addr} {client_addr} {client_name} {auth_authen}
milter_default_action = accept
smtpd_milters = inet:127.0.0.1:11332
non_smtpd_milters = inet:127.0.0.1:11332
 ```
 - /etc/postfix/master.cf
 ```
 vim /etc/postfix/master.cf
 ```
 Find and Comment 
 ```
 smtp-amavis   unix   -   -   n   -   2   smtp
    -o syslog_name=postfix/amavis
    -o smtp_data_done_timeout=1200
    -o smtp_send_xforward_command=yes
    -o disable_dns_lookups=yes
    -o max_use=20
    -o smtp_tls_security_level=none
```
- Restart 
```
systemctl restart rspamd
systemctl restart postfix
```
