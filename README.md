# nexus-nginx-ssl

## Create SSL certificate

1. Generate public private key pair using keytool:

keytool -genkeypair -keystore keystore.jks -storepass changeit -alias imp.com \
 -keyalg RSA -keysize 2048 -validity 5000 -keypass changeit \
 -dname 'CN=*.imp.com, OU=Sonatype, O=Sonatype, L=Unspecified, ST=Unspecified, C=US' \
 -ext 'SAN=DNS:nexus.imp.com,DNS:clm.imp.com,DNS:repo.imp.com,DNS:www.imp.com'
 
 
2. Generate PEM encoded public certificate file using keytool:

keytool -exportcert -keystore keystore.jks -alias imp.com -rfc > imp.cert
pwd: changeit

3. Convert our Java specific keystore binary".jks" file to a widely compatible PKCS12 keystore ".p12" file:

keytool -importkeystore -srckeystore keystore.jks -destkeystore imp.p12 -deststoretype PKCS12

4. (Optional) Extract pem (certificate) from ".p12" keystore file ( this is same as step 2, but openssl spits out more verbose contents ):

openssl pkcs12 -nokeys -in imp.p12 -out imp.pem

5. Extract unencrypted private key file from ".p12" keystore file:

openssl pkcs12 -nocerts -nodes -in imp.p12 -out imp.key

Note: https://support.sonatype.com/hc/en-us/articles/213465768?_ga=2.157000955.1301493906.1563870801-525921679.1563783091
