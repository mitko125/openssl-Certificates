# Използване на `openssl` за създаване на самоподписани SSL сертификати

Връзката в интернет между клиентите (уеб браузъри) и интернет страниците е шифрована. Всеки сайт `https сървър` притежава одобрен от сертифициращ орган сертификат. За домашни или вътрешнофирмени връзки може да ползва нешифрована `http` или самоподписан сертификат, който се изготвя с [openSSL](https://www.openssl.org/) инструмент. Всички команди към `openssl` са през `CMD`. Самоподписания сертификат трябва да се регистрира при клиента, за де не протестира уеб браузъра (остават само червени съобщения че сървъра е ненадежден). Разглеждат се два случая на издаване на сертификати:

- няколко клиента към няколко сървъра [многосвързаност](#няколко-сървърни-сертификата-с-един-общ-потребителски-клиентски-root-ca);
- един клиент с един сървър [единична връзка](#единичен-сървърен-сертификат)

## Няколко сървърни сертификата с един общ потребителски (клиентски) root CA

Издава се само един SSL сертификат `rootCA`, който се `регистрира` във всички компютри `ползващи` вътрешни `https сървъри`. Всички компютри или устройства с `https сървъри`имат SSL сертификати подписанни от `rootCA`. По този начин се избягва `регистрирането` на всички `https сървъри` във всички, които ги `ползват`.

### Създайте `rootCA.key` (еднократно)

### 1. Създаване на основен ключ  

**Внимание:** файла `rootCA.key` е ключът, използван за подписване на заявките за сертификати, всеки, който го притежава, може да подписва сертификати от ваше име. Затова го пазете на сигурно място!  

```text
openssl genrsa -des3 -out rootCA.key 4096
```  

Ако искате незащитен с парола ключ, просто премахнете `-des3` опцията.  

### 2. Създайте и подпишете самостоятелно основния сертификат (за 1024 дни)

```text
openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 1024 -out rootCA.crt
```  

Тук използвахме нашия основен ключ, за да създадем основния сертификат (файла [rootCA.crt](#1-създаване-на-основен-ключ)), който **трябва да бъде разпространен във всички компютри**, които трябва да ни се доверят.  
Трябва да попълним малко данни (може и игнор...):

```text
Country Name (2 letter code) [AU]:BG
State or Province Name (full name) [Some-State]:Sliven
Locality Name (eg, city) []:Sliven
Organization Name (eg, company) [Internet Widgits Pty Ltd]:«ELL-DANEV & BOZHILOV» LTD
Organizational Unit Name (eg, section) []:ELL for test
Common Name (e.g. server FQDN or YOUR name) []:
Email Address []:ell@ell-bg.com
```

Може да проверим:  

```text
openssl x509 -in rootCA.crt -noout -text
```  

### 3. Разпросраняване `регистриране` във всички компютри `ползващи` вътрешни `https сървъри`

Във Windows двукратно щракване върху файла `rootCA.crt`, `инсталиеане..`, избиране на хранилище надежни главни сертификати и ....

### Създайте краен сертификат (извършва се за всеки `https сървър`)

### 1. Създайте ключа на сертификата

```text
openssl genrsa -out prvtkey.pem 4096
```

Някои избират име за ключа, като на сървъра например `mydomain.com.key`.Аз понеже си играя с [ESP-IDF](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/get-started/index.html) за удобство ползвам тяхното име `prvtkey.pem`. Този ключ се съхранява в сървъра и не се разпостранява.

### 2. Създаване на подпис за сертификата (csr)

#### Метод A (интерактивен)

```text
openssl req -new -key prvtkey.pem -out mydomain.com.csr
```

Всички данни се попълват от командния ред. Подписа `mydomain.com.csr` е нужен за издаването на сертификата и не се рапространява.

#### Метод B (конфигурационен файл)

- създаване на файла `mydomain.com.conf` (ползвам конфигурацията от тестова платка ESP32)

```conf
[req]
distinguished_name = req_distinguished_name
req_extensions = req_ext
prompt = no
[req_distinguished_name]
C = BG
ST = Sliven
L = Sliven
O = "ELL-DANEV & BOZHILOV" LTD
OU = ELL for test
CN = test_esp32_cert.local

[req_ext]
keyUsage = keyEncipherment, dataEncipherment, digitalSignature
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = test_esp32_cert
DNS.2 = test1.test_esp32_cert.local
DNS.3 = '*.test1.test_esp32_cert'
IP.1 = 192.168.1.105
IP.2 = 192.168.1.4
```

Важни за нас са `CN = ..`, `DNS... = ..` и `IP... = ..`. Сетрификата е за тези сайтове.

- създаване на подписа

```text
openssl req -new -sha256 -out mydomain.com.csr -key prvtkey.pem -config mydomain.com.conf
```

- можем да го проверим

```text
openssl req -in mydomain.com.csr -noout -text
```

### 3. Генерирайте сертификата на сървъра (за 500 дни)

```text
openssl x509 -req -in mydomain.com.csr -CA rootCA.crt -CAkey rootCA.key -CAcreateserial -out servercert.pem -days 500 -sha256
```

- можем да го проверим

```text
openssl x509 -in servercert.pem -text -noout
```

Готови сме. Трябва да вградим в сървъра двата файла [servercert.pem](#3-генерирайте-сертификата-на-сървъра-за-500-дни) и [prvtkey.pem](#1-създайте-ключа-на-сертификата).

## Единичен сървърен сертификат

В примерите на [ESP-IDF](https://github.com/espressif/esp-idf/tree/master/examples/protocols/https_server), правят сертификат с една команда:

```text
openssl req -newkey rsa:2048 -nodes -keyout prvtkey.pem -x509 -days 3650 -out servercert.pem -subj "/CN=test_esp32_cert.local"
```

Имаме само `CN = ..`, няма `DNS... = ..` и `IP... = ..`. Но и така работи с трите адреса <https://192.168.1.105/>, <https://test_esp32_cert/> и <https://test_esp32_cert.local/>.  
Ако ни трябват, ще се направи комбинация с [горния метод](#2-създаване-на-подпис-за-сертификата-csr) и опцията `-nodes`.

## ЗАКЛЮЧЕНИЕ

Самоподписаните сертификати стават за вътрешни тестове. Май са вършели работа преди 2018г. Но сега трябва да се търся други пътища като `Reverse PROXY`, `Let's Encrypt`.
