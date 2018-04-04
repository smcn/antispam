# Açık Kaynak Antispam Teknolojileri

#### FreeBSD üzerinde  MailScanner e-posta ağ geçidi sistemi yapılandırılarak virüslerin, istenmeyen postaların ve kötü amaçlı yazılımların engellenmesine çalışılacaktır.

- Postfix
- Postgrey
- MailScanner
- ClamAv
- SpamAssassin(Razor, Pyzor, DCC)
- MailWatch 

### Postfix

MailScanner, MTA (Mail Transfer Agent) olarak Postfix, Sendmail, Exim  kullanılabilir.

Postfix, Linux ve Unix işletim sistemlerinde mail alıp mail göndermesine imkan sağlayan açık kaynak e-posta yazılımıdır.

Yoğun e-posta trafiklerinde, Postfix tercih ediliyor.

```
pkg install -y postfix

sysrc postfix_enable="YES"
sysrc sendmail_enable="NONE"

```

```
nano /etc/periodic.conf

daily_clean_hoststat_enable="NO"
daily_status_mail_rejects_enable="NO"
daily_status_include_submit_mailq="NO"
daily_submit_queuerun="NO"
```

```
cd /etc/mail 
nano /etc/mail/aliases
root: sistem@omu.edu.tr
postalias /etc/mail/aliases
```

```
cd /usr/local/etc/postfix
echo "omu.edu.tr smtp:[193.140.28.32]" > transport
postmap transport
```

```
echo "/^Received:/ HOLD" > header_checks
```

```
nano /usr/local/etc/postfix/main.cf
myhostname = mailgw.omu.edu.tr
mydomain = localhost
inet_interfaces = all
mydestination = $myhostname, localhost.$mydomain, localhost
mynetworks_style = host
mynetworks = 127.0.0.0/8
relay_domains = omu.edu.tr 
relayhost = [193.140.28.32] 
alias_maps = hash:/etc/mail/aliases 
alias_database = hash:/etc/mail/aliases 
header_checks = regexp:$config_directory/header_checks
```

```
nano /usr/local/etc/postfix/master.cf
qmgr        fifo  n       -       n       300     1       qmgr
bounce    unix  -       -       n       -          0       discard
```

### Postgrey

Postgrey ile MTA seviyesinde greylist yapabiliriz. Bu bize spam eposta gönderenleri bir nebze olsun azaltacaktır. 

Mantığı ise şu şekilde; doğru ayarlanmış bir eposta sunucusu cevap olarak 4xx hatası aldığında bir süre sonra bize eposta göndermeyi tekrar dener.

Spam eposta gönderenlerin ise böyle bir lüksü olmadığı için vazgeçebilirler.

Postgrey, X adresinden Y adresine ilk defa eposta gönderiliyorsa Postfix’ in 4xx hatası vererek gönderimi ertelemesini sağlıyor. Sunucu tekrar gönderim yaptığındaysa kabul edip whitelist’ e ekliyor.

Gri listeleme, posta sunucusu düzeyinde önemli miktarda spam‘ i engellemek için kullanılan yeni bir yöntemdir. 

Sonuç olarak, uygulamalar oldukça hafiftir ve posta sunucunuzdaki ağ trafiğini ve işlemci yükünü bile azaltabilir.
 
Halihazırda kendi başına çok etkili olmasına rağmen, diğer istenmeyen posta önleme biçimleriyle birlikte kullanıldığında en iyi sonucu verecektir.

```
pkg install –y postgrey
```

```
nano /usr/local/etc/rc.d/postgrey
: ${postgrey_enable:=YES}
: ${postgrey_dbdir:=/var/db/postgrey}
##: ${postgrey_flags:=--inet=10023}
: ${postgrey_flags:=--unix=/var/run/postgrey/postgrey.sock}
```

```
nano /usr/local/etc/postfix/main.cf
smtpd_recipient_restrictions = permit_mynetworks,reject_unauth_destination,check_policy_service unix:/var/run/postgrey/postgrey.sock
```

```
sysrc postgrey_enable="YES"
/usr/local/etc/rc.d/postgrey start
postfix reload
```

### MailScanner

MailScanner, Linux tabanlı e-posta ağ geçitleri için oldukça saygın bir açık kaynaklı e-posta güvenlik sistemi tasarımıdır.
 
Dünyanın dört bir yanındaki 40.000'den fazla sitede kullanılıyor, üst düzey hükümet dairelerini, ticari şirketleri ve eğitim kurumlarını koruyor.
 
Bu teknoloji, virüs koruması ve spam filtreleme için birçok ISS sitesinde standart e-posta çözümü haline gelmiştir. 

MailScanner e-postaları virüsler, spam, phishing, kötü amaçlı yazılımlar ve güvenlik açıklarına karşı yapılan diğer saldırılara karşı tarar ve bir ağın güvenliğinde önemli bir rol oynar. 

MailScanner, farklı MTA (Postfix, Sendmail, Exim) ları, başta ClamAV olmak üzere çeşitli antivirüs yazılımlarını(Avast!, F-Prot, AVIRA vb.) destekler.
 
Spam algılama işlemi standart spam algılama motoru olan Spamassassin ile gerçekleştirilir.

```
pkg install -y MailScanner

sysrc spamd_enable="YES"
sysrc mailscanner_enable="YES"
sysrc clamav_clamd_enable="YES"
sysrc clamav_freshclam_enable="YES"
```

```
mkdir /var/spool/MailScanner
mkdir /var/spool/MailScanner/incoming
mkdir /var/spool/MailScanner/quarantine
mkdir /var/spool/MailScanner/spamassassin

chown -R postfix:mail /var/spool/MailScanner
chmod -R 775 /var/spool/MailScanner
```

MailScanner işleyişi gereği Postfix’ in kuyruk dosyalarını manipüle ediyor. 

Tüm gelen epostalar Postfix’ in hold” kuyruğunda bekler, 

MailScanner kuyruk dizinini ayarladığımız süreler içerisinde kontrol eder, 

bekleyen eposta varsa alır, 

ekleri ve epostayı birbirinden ayırıp antivirüs ve antispam testlerinden geçirip postfix’ in “incoming” kuyruğuna koyar.

```
nano /usr/local/etc/MailScanner/MailScanner.conf
%org-name% = omu
%web-site% = www.omu.edu.tr
Run As User = postfix
Run As Group = mail
Incoming Queue Dir = /var/spool/postfix/hold
Outgoing Queue Dir = /var/spool/postfix/incoming
MTA = postfix
Incoming Work User = clamav
Incoming Work Group = mail
Virus Scanners = clamd
Still Deliver Silent Viruses = yes
Quarantine Whole Message = yes
Quarantine Whole Messages As Queue Files = yes
SpamAssassin User State Dir = /var/spool/MailScanner/spamassassin
Always Include SpamAssassin Report = yes
Log Spam = yes
Spam List = spamhaus-ZEN
```

High SpamAssassin Score default 10 gelmekte, ideali 5

```
Required SpamAssassin Score = 4.75
High SpamAssassin Score = 5
```

MailScanner debug modda çalıştırılarak, yüklü olmayan perl modülleri yüklenmeli.

```
MailScanner –D --lint
```

```
nano /usr/local/sbin/ms-update-sa
grep -q '^loadplugin.*Rule2XSBody' /usr/local/etc/mail/spamassassin/*pre 2>/dev/null
```

```
ln -s /usr/local/bin/freshclam /usr/bin/freshclam
/usr/local/sbin/ms-update-vs
```

### ClamAV

Clam AntiVirüs Unix/Linux sistemleri için e-posta sunucuları üzerinde e-posta virüs taraması yapmak üzere geliştirilmiştir.

1 milyonun üzerinde virüs, solucan, trojan, MS Office MacOffice makro virüsleri tespit edebilir.

clamav ve clamd olarak virüs taraması yapılabilir.

ClamAV’ı daemon olarak çalıştırmanız performans açısından büyük yarar sağlayacaktır. 

ClamAV her başlattığınızda virüs imza veritabanı tekrar tekrar okuyacağına, daemon olarak kullandığınızda bir defa okunup her tarama için yeni bir process oluşturacaktır.

```
cat /usr/local/etc/MailScanner/MailScanner.conf|grep "Virus Scanners"
Virus Scanners = clamd
```

ClamAV’in clamd ve freshclam olmak üzere iki adet daemon’ ı bulunuyor; 
clamd dosya tarama işlerini yaparken, 
freshclam ise ClamAV’ı imza veritabanını günceller.
MailScanner antivirüs ayarları için default ayarlar yeterli olacaktır.

```
/usr/local/etc/rc.d/clamav-freshclam start
/usr/local/etc/rc.d/clamav-clamd start
```

Virüs imza veritabanını güncellemek için

```
# freshclam
```

### SpamAssassin

Apache SpamAssassin, sistem yöneticilerine e-postayı sınıflandırmak ve spam engellemek için bir filtre sağlayan açık kaynak antispam platformudur. 

Metin analizleri, Bayes filtreleme, DNS blok listeleri ve işbirlikçi filtreleme veritabanları dahil olmak üzere, e-posta başlıklarında ve gövde metinlerinde çok çeşitli gelişmiş sezgisel ve istatistiksel analiz testlerini entegre etmek için sağlam bir puanlama çerçevesi ve eklentileri kullanır.

SpamAssassin MailScanner tarafından çağırılarak Perl modülü olarak kullanılır.

SpamAssassin debug modda çalıştırılarak, yüklü olmayan perl modülleri kurulmalı.

```
spamassassin -D --lint
```

```
pkg install razor-agents p5-LWP-UserAgent-Determined p5-Net-Patricia p5-Geo-IP p5-IO-Socket-INET6

```

```
sa-update
```

```
mkdir /usr/local/etc/MailScanner/bayes
chown root:www /usr/local/etc/MailScanner/bayes
chmod g+rws /usr/local/etc/MailScanner/bayes
cp /var/spool/MailScanner/spamassassin/bayes_* /usr/local/etc/MailScanner/bayes
chown root:www /usr/local/etc/MailScanner/bayes/bayes_*
chmod g+rw /usr/local/etc/MailScanner/bayes/bayes_*
```

```
nano /usr/local/etc/MailScanner/spamassassin.conf
bayes_ignore_header omu-MailScanner
bayes_ignore_header omu-MailScanner-SpamCheck
bayes_ignore_header omu-MailScanner-SpamScore
bayes_ignore_header omu-MailScanner-Information
bayes_path /usr/local/etc/MailScanner/bayes/bayes
bayes_file_mode 0660
envelope_sender_header X-omu-MailScanner-From
```

```
pkg install -y py36-pyzor dcc-dccd
```

```
nano /usr/local/etc/MailScanner/spamassassin.conf
ifplugin Mail::SpamAssassin::Plugin::Pyzor
        pyzor_path /usr/local/bin/pyzor
endif
ifplugin Mail::SpamAssassin::Plugin::DCC
        dcc_path /usr/local/bin/dccproc
endif
ifplugin Mail::SpamAssassin::Plugin::Razor2
        razor_config  /usr/local/etc/mail/spamassassin/razor/razor-agent.conf
endif
```

```
nano /usr/local/etc/mail/spamassassin/v310.pre
loadplugin Mail::SpamAssassin::Plugin::DCC
loadplugin Mail::SpamAssassin::Plugin::Pyzor
loadplugin Mail::SpamAssassin::Plugin::Razor2
```

```
cd /usr/local/etc/mail/spamassassin/razor/
razor-client
razor-admin -home=/usr/local/etc/mail/spamassassin/razor -register
razor-admin -home=/usr/local/etc/mail/spamassassin/razor -create
razor-admin -home=/usr/local/etc/mail/spamassassin/razor -discover
```

Spam olan ve olmayan test dosyaları indirilip pyzor ve razor2 nin çalıştığı test edilir.

```
cd /usr/local/share/doc/spamassassin
wget http://svn.apache.org/repos/asf/spamassassin/trunk/sample-nonspam.txt
wget http://svn.apache.org/repos/asf/spamassassin/trunk/sample-spam.txt
```

```
spamassassin -t -D pyzor < /usr/local/share/doc/spamassassin/sample-spam.txt
spamassassin -t -D pyzor < /usr/local/share/doc/spamassassin/sample-nonspam.txt
```

```
spamassassin -t -D razor2 < /usr/local/share/doc/spamassassin/sample-spam.txt
spamassassin -t -D razor2 < /usr/local/share/doc/spamassassin/sample-nonspam.txt
```

### MailWatch
MailWatch, PHP ve MySQL ile yazılmış MailScanner‘ ın web GUI sidir ve GNU Public License'ın şartları altında ücretsiz olarak kullanılabilir. 

MailScanner için MailScanner'ın tüm mesaj verilerini (gövde metni hariç) bir MySQL veritabanına kaydetmesini sağlayan ve daha sonra MailWatch tarafından raporlama ve istatistik için sorgulanan bir CustomConfig modülü ile birlikte gelir.

```
nano /usr/local/etc/MailScanner/MailScanner.conf

Always Looked Up Last = &MailWatchLogging
Spam Actions = store-spam
Is Definitely Not Spam = &SQLWhitelist
Is Definitely Spam = &SQLBlacklist
```

```
cd /usr/local/www/
wget https://github.com/mailwatch/MailWatch/archive/v1.2.8.zip
unzip v1.2.8.zip
mv MailWatch-1.2.8 mailwatch
cd mailwatch/mailscanner
chown root:www temp
chmod g+rw temp
cp conf.php.example conf.php
```

```
nano conf.php

define('TIME_ZONE', 'Europe/Istanbul');
define('DB_PASS', ' <secret>');
define('MAILWATCH_HOME', '/usr/local/www/mailwatch/mailscanner');
define('MS_CONFIG_DIR', '/usr/local/etc/MailScanner/');
define('MS_SHARE_DIR', '/usr/local/share/MailScanner/perl/MailScanner/');
define('MS_LIB_DIR', '/usr/local/lib/MailScanner/wrapper/');
define('MS_EXECUTABLE_PATH', '/usr/local/sbin/MailScanner
define('IMAGES_DIR', '/images/'); 
define('SA_DIR', '/usr/local/bin/');
define('SA_RULES_DIR', '/usr/local/share/spamassassin/');
define('SA_PREFS', MS_CONFIG_DIR . 'spamassassin.conf'); 
define('TEMP_DIR', '/tmp/');
```

```
pkg install -y mariadb102-server php72-mysqli apache24 mod_php72 php72-extensions
```

```
cp /usr/local/etc/php.ini-production /usr/local/etc/php.ini
```

```
nano /usr/local/etc/php.ini
date.timezone = "Europe/Istanbul"
```

```
sysrc mysql_enable="YES"
sysrc apache24_enable="YES"
service mysql-server start
mysql_secure_installation
```

```
nano /usr/local/etc/apache24/Includes/php.conf
<FilesMatch "\.php$">
    SetHandler application/x-httpd-php
</FilesMatch>
<FilesMatch "\.phps$">
    SetHandler application/x-httpd-php-source
</FilesMatch>
```

```
nano /usr/local/etc/apache24/httpd.conf
DirectoryIndex index.php index.html
DocumentRoot "/usr/local/www/mailwatch/mailscanner/"
<Directory "/usr/local/www/mailwatch/mailscanner/">
```

```
service apache24 start
```

```
cd /usr/local/www/mailwatch/
mysql -uroot -p < create.sql

mysql
mysql> GRANT ALL ON mailscanner.* TO mailwatch@localhost IDENTIFIED BY '<secret>';
mysql> GRANT FILE ON *.* TO mailwatch@localhost IDENTIFIED BY '<secret>';
mysql> FLUSH PRIVILEGES;
mysql> quit
```

```
mysql mailscanner -u mailwatch -p
mysql> INSERT INTO users SET username = 'admin', password = MD5('<password>'), fullname = '<name>', type = 'A';
mysql> FLUSH PRIVILEGES;
mysql> quit
```

```
ln -s /usr/local/www/mailwatch/MailScanner_perl_scripts/MailWatch.pm /usr/local/share/local/MailScanner/perl/custom
ln -s /usr/local/www/mailwatch/MailScanner_perl_scripts/MailWatchConf.pm /usr/local/share/MailScanner/perl/custom
ln -s /usr/local/www/mailwatch/MailScanner_perl_scripts/SQLBlackWhiteList.pm /usr/local/share/MailScanner/perl/custom
ln -s /usr/local/www/mailwatch/MailScanner_perl_scripts/SQLSpamSettings.pm /usr/local/share/MailScanner/perl/custom
```

```
nano /usr/local/www/mailwatch/MailScanner_perl_scripts/MailWatchConf.pm
my ($db_pass) = '<secret>';
```

```
nano /usr/local/www/MailWatch-1.2/mailscanner/sf_version.php
////exec("which postconf", $postconf);
$postconf = array("/usr/local/sbin/postconf");
```

```
perl -MCPAN -e shell
install DBD::mysql
```

```
cpan Encoding::FixLatin
```

![MailWatch](https://github.com/smcn/antispam/blob/master/mw.jpg)


