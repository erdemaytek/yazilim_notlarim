Bu yazımda Zimbra Mail Sunucuya Comodo/Sectigo SSL Certificate Sertifika nasıl yüklenir basit bir şekilde aktarmaya çalışacağım.

# Aşama 1: İhtiyacımız Olan Sertifika Bilgileri

- AddTrustExternalCARoot.crt

- COMODORSAAddTrustCA.crt

- COMODORSADomainValidationSecureServerCA.crt

- DomainAdı.crt (Domain sertifikamız)

- .key uzantılı Privite key dosyası




# Aşama 2: .pfx Uzantısını .key Uzantılı Dosyaya çevirmek (Varsa sizde bu adımı geçebilirsiniz)


OpenSSL indirilelim: http://www.slproweb.com/products/Win32OpenSSL.html


Kurulum bittikten sonra kurulum dizinindeki bin klasörüne cmd kullanarak girelim Örnek`C:\OpenSSL-Win32\bin`. Aşağıdaki adımları sırası ile uygulayalım.

```shell
Set OPENSSL_CONF=c:\openssl-win32\bin\openssl.cfg 
openssl pkcs12 -in filename.pfx -nocerts -out key.pem
openssl rsa -in key.pem -out myserver.key
```

Bu işlemin sonucunda artık elimizde .key uzantılı privite keyimiz var.

# Aşama 3: Sertifikaların birleştirilmesi

```shell
cat AddTrustExternalCARoot.crt COMODORSAAddTrustCA.crt COMODORSADomainValidationSecureServerCA.crt > /tmp/commercial_ca.crt
```




sertifikalarımızı cat komutunu kullanarak `commercial_ca.crt `isminde `/tmp `dizinimize oluşturuyoruz.

```shell
cp doman_sertifika.crt /tmp/commercial.crt
```


ikinci olarak ise domain sertifikasını da `tmp `dizinine `commercial.crt` isminde kopyalıyoruz.

```shell
cp keydosyası.key /opt/zimbra/ssl/zimbra/commercial/commercial.key
```

`/opt/zimbra/ssl/zimbra/commercial/` yoluna privite key dosyamızı `commercial.key` olarak isimlendirip kopyalıyoruz

```shell
/opt/zimbra/bin/zmcertmgr verifycrt comm /opt/zimbra/ssl/zimbra/commercial/commercial.key /tmp/commercial.crt /tmp/commercial_ca.crt 
** Verifying /tmp/commercial.crt against /opt/zimbra/ssl/zimbra/commercial/commercial.key
Certificate (/tmp/commercial.crt) and private key (/opt/zimbra/ssl/zimbra/commercial/commercial.key) match.
Valid Certificate: /tmp/commercial.crt: OK

```

Yukarıdaki komut sayesinde keyimiz ile sertifikalarımızı kontrol ediyoruz. Eğer sorun yoksa kuruluma başlayabiliriz. Bu aşamayı zimbra `ZCS 8.7` ve üzeri ise zimbra kullanıcısı ile yapmanız gerekiyor.

# Aşama 4: Sertifikanın Aktif Edilmesi

```shell
/opt/zimbra/bin/zmcertmgr deploycrt comm /tmp/commercial.crt /tmp/commercial_ca.crt 
** Verifying /tmp/commercial.crt against /opt/zimbra/ssl/zimbra/commercial/commercial.key
Certificate (/tmp/commercial.crt) and private key (/opt/zimbra/ssl/zimbra/commercial/commercial.key) match.
Valid Certificate: /tmp/commercial.crt: OK
** Copying /tmp/commercial.crt to /opt/zimbra/ssl/zimbra/commercial/commercial.crt
** Appending ca chain /tmp/commercial_ca.crt to /opt/zimbra/ssl/zimbra/commercial/commercial.crt
** Importing certificate /opt/zimbra/ssl/zimbra/commercial/commercial_ca.crt to CACERTS as zcs-user-commercial_ca...done.
** NOTE: mailboxd must be restarted in order to use the imported certificate.
** Saving server config key zimbraSSLCertificate...done.
** Saving server config key zimbraSSLPrivateKey...done.
** Installing mta certificate and key...done.
** Installing slapd certificate and key...done.
** Installing proxy certificate and key...done.
** Creating pkcs12 file /opt/zimbra/ssl/zimbra/jetty.pkcs12...done.
** Creating keystore file /opt/zimbra/mailboxd/etc/keystore...done.
** Installing CA to /opt/zimbra/conf/ca...done.

```



`/opt/zimbra/bin/zmcertmgr deploycrt comm /tmp/commercial.crt /tmp/commercial_ca.crt` komutunu kullanarak sertifikamızı aktif ediyoruz. İşlem bittikten sonra `zmcontrol `servisini yeniden başlatarak kurulumu bitiriyoruz.


```shell
zmcontrol restart
```


