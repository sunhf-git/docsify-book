# 一键证书安装脚本 <!-- {docsify-ignore-all} -->
运行环境 centos 7+
```shell script
storepass=123456
keypass=123456
s27_dname="C=CN,ST=GD,L=SZ,O=vihoo,OU=NIFI,CN=s27.nifiserver.com"
s29_dname="C=CN,ST=GD,L=SZ,O=vihoo,OU=NIFI,CN=s29.nifiserver.com"
echo "generate s27 keystore"
keytool -genkeypair -alias s27key -keypass $keypass -storepass $storepass \
    -dname $s27_dname \
    -keyalg RSA -keysize 2048 -validity 3650 -keystore s27.keystore
echo "generate s29 keystore"
keytool -genkeypair -alias s29key -keypass $keypass -storepass $storepass \
    -dname $s29_dname \
    -keyalg RSA -keysize 2048 -validity 3650 -keystore s29.keystore
echo "export s27 certificate"
keytool -exportcert -keystore s27.keystore -file s27.cer -alias s27key -storepass $storepass
echo "export s29 certificate"
keytool -exportcert -keystore s29.keystore -file s29.cer -alias s29key -storepass $storepass
echo "add s27 cert to s29 trust keystore"
keytool -importcert -keystore s29_trust.keystore -file s27.cer -alias s29_trust_s27 \
    -storepass $storepass -noprompt
echo "add s29 cert to s27 trust keystore"
keytool -importcert -keystore s27_trust.keystore -file s29.cer -alias s27_trust_s29 \
    -storepass $storepass -noprompt

echo "jks convert to p12"
keytool -importkeystore -srckeystore s27.keystore  -destkeystore s27.p12 \
-srcstoretype jks -deststoretype pkcs12 -srcalias s27key -destalias s27 \
-deststorepass $storepass -srcstorepass $storepass

keytool -importkeystore -srckeystore s29.keystore  -destkeystore s29.p12 \
-srcstoretype jks -deststoretype pkcs12 -srcalias s29key -destalias s29 \
-deststorepass $storepass -srcstorepass $storepass

echo "jks convert to pem and key"
openssl pkcs12 -in s27.p12 -clcerts -nokeys -password pass:$storepass -out s27.crt
openssl pkcs12 -in s27.p12  -nocerts -password pass:$storepass -passout pass:$storepass -out s27.key
openssl pkcs12 -in s29.p12 -clcerts -nokeys -password pass:$storepass -out s29.crt
openssl pkcs12 -in s29.p12  -nocerts -password pass:$storepass -passout pass:$storepass -out s29.key

```