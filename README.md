# nexus-nginx-ssl

## 1. Create SSL certificate

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

## 2. Install Nexus Repo

1. Create dir for nexus
    mkdir -p /some/dir/nexus3/data
   chown -R 200 /some/dir/nexus3/data
   
2.  Go to the “nexus3” directory and Create a file named “docker-compose.yml” with the following content:
  nexus:
    image: sonatype/nexus3:latest
    ports:
        - "8081:8081"
        - "8123:8123"
    volumes:
        - ./data:/nexus-data
3. Launch Nexus
   docker-compose up -d
   docker-compose logs
   
4. Create newtwork
   docker network create nexusnet
   
5. Connect network running container
   
6. Open the URL [Docker host IP address]:8081 in a web browser   


*Note : https://www.ivankrizsan.se/2016/06/09/create-a-private-docker-registry/

## 3. Install Nginx proxy
A. Create Nginx Image:
  
1. Create folder nginx at level of nexus
   mkdir nginx
   
2. Create DOckerfile with below context:
   	FROM nginx
	COPY ./nginx.conf /etc/nginx/nginx.conf
	
3. Create Nginx.conf file at same level of dockerfile

4. run below command to create nginx image with all required details
   docker build -t nginx-img .

   
B. Run nginx image 
1. create folder and All the commands will then need to be run from this directory
   mkdir docker_ssl_proxy
   
2.  Copy certificates to this directory
 
3. execute below command to run nginx
   docker run --name nginx -d -v `pwd`:/etc/nginx/conf.d -p 443:443 --network nexusnet nginx-img
   
Note: https://blog.sonatype.com/setting-up-a-docker-private-registry-with-authentication-using-nexus-and-nginx   

## 4. Docker client setup
### REDHAT7.x or CENTOS7.x
A. Configure certificate on machine

1.  Add entry to /etc/hosts file 
    52.170.206.176  repo.imp.com nexus.imp.com

2. Use Oracle java `keytool` to retrieve and print the Nexus server certificate for the Nexus instance running at ${NEXUS_DOMAIN}:${SSL_PORT} :
   keytool -printcert -sslserver repo.imp.com -rfc
	
3. Copy certificate recived from above step to below localtion
   /etc/pki/ca-trust/source/anchors/repo.imp.com.crt
   
4. Fires command 
   sudo update-ca-trust enable; sudo update-ca-trust extract	
   

B. Configure certificate on docker client	

1. Create folder with repo name in /etc/docker/certs.d dir
    sudo mkdir /etc/docker/certs.d/repo.imp.com
	
2. Copy certificate to this new folder with name ca.cert
   sudo cp /etc/pki/ca-trust/source/anchors/repo.imp.com.crt repo.imp.com/ca.crt
 
3.  restart docker service 
   sudo systemctl restart docker

4.  Login to docker registry , this login should be successful
   docker login repo.imp.com 

5. try to pull or push images  


### UBUNTU/DEBIAN:

A. Configure certificate on machine

1.  Add entry to /etc/hosts file 
    52.170.206.176  repo.imp.com nexus.imp.com

2. Use Oracle java `keytool` to retrieve and print the Nexus server certificate for the Nexus instance running at ${NEXUS_DOMAIN}:${SSL_PORT} :
   keytool -printcert -sslserver repo.imp.com -rfc
	
3. Copy certificate recived from above step to below localtion
   /usr/local/share/ca-certificates/repo.imp.com.crt
   
4. Fires command 
   sudo update-ca-certificates	
   
5. verify correctnes using below command
    curl https://repo.imp.com

B. Configure certificate on docker client	

1. Create folder with repo name in /etc/docker/certs.d dir
    sudo mkdir repo.imp.com
	
2. Copy certificate to this new folder with name ca.cert
   sudo cp /usr/local/share/ca-certificates/repo.imp.com.crt repo.imp.com/ca.crt
 
3.  restart docker service 
   sudo service docker restart

4.  Login to docker registry , this login should be successful
   docker login repo.imp.com 

5. try to pull or push images   

Note: https://docs.docker.com/registry/insecure/#docker-still-complains-about-the-certificate-when-using-authentication
      https://support.sonatype.com/hc/en-us/articles/217542177-Using-Self-Signed-Certificates-with-Nexus-Repository-Manager-and-Docker-Daemon step 3
