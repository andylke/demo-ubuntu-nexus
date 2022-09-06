# Setup Nexus

## Install JDK-8 (Required by Nexus Repository OSS)

`sudo apt install openjdk-8-jdk-headless`

## Install Nexus Repository OSS

Create user `nexus` without password

`sudo adduser --no-create-home --disabled-login --disabled-password nexus`

```
Adding user `nexus' ...
Adding new group `nexus' (1001) ...
Adding new user `nexus' (1001) with group `nexus' ...
Not creating home directory `/home/nexus'.
Changing the user information for nexus
Enter the new value, or press ENTER for the default
        Full Name []:
        Room Number []:
        Work Phone []:
        Home Phone []:
        Other []:
Is the information correct? [Y/n] Y
```

Download and Extract:

```
cd /tmp
wget https://download.sonatype.com/nexus/3/latest-unix.tar.gz
tar -zxvf latest-unix.tar.gz
sudo mv $(ls | grep "nexus-3.*") /opt/nexus
sudo mv sonatype-work /opt/sonatype-work
```

```
sudo chown -R nexus:nexus /opt/nexus
sudo chown -R nexus:nexus /opt/sonatype-work
```

## Configure Nexus

To run nexus as service at boot time, open /opt/nexus/bin/nexus.rc file, uncomment it and add nexus user as shown below:
`sudo nano /opt/nexus/bin/nexus.rc`
`run_as_user="nexus"`

`sudo nano /opt/nexus/bin/nexus.vmoptions`

To run nexus as service using Systemd
`sudo nano /etc/systemd/system/nexus.service`
Paste the below lines into it.

```
[Unit]
Description=nexus service
After=network.target

[Service]
Type=forking
LimitNOFILE=65536
ExecStart=/opt/nexus/bin/nexus start
ExecStop=/opt/nexus/bin/nexus stop
User=nexus
Restart=on-abort

[Install]
WantedBy=multi-user.target
```

Set the path for a JDK-8 location
`sudo nano /opt/nexus/bin/nexus`

```
# Uncomment the following line to override the JVM search sequence
INSTALL4J_JAVA_HOME_OVERRIDE=/usr/lib/jvm/java-8-openjdk-amd64
```

`sudo systemctl daemon-reload`

To start nexus service using systemctl
`sudo systemctl start nexus`

To enable nexus service at system startup
`sudo systemctl enable nexus`

Check nexus service status
`sudo systemctl status nexus`

Access Nexus Repository
`http://192.168.1.30:8081'

`cat /opt/sonatype-work/nexus3/admin.password`

## Setup SSL for Nexus

Stop Nexus
`sudo systemctl stop nexus.service`

Convert SSL keys to PKCS12 format

```
openssl pkcs12 -export -out nexus.p12 \
-passout 'pass:admin@123' -inkey server.key \
-in server.crt -certfile ca.crt -name nexus.server.example.com
```

Convert PKCS12 to JKS format

```
keytool -importkeystore -srckeystore nexus.p12 \
-srcstorepass 'admin@123' -srcstoretype PKCS12 \
-srcalias nexus.server.example.com -deststoretype JKS \
-destkeystore nexus.jks -deststorepass 'admin@123' \
-destalias nexus.server.example.com
```

Move JKS to `/opt/nexus/etc/ssl`

```
sudo cp nexus.jks /opt/nexus/etc/ssl
sudo chown -R nexus: /opt/nexus/etc/ssl
sudo chmod 700 /opt/nexus/etc/ssl
sudo chmod 644 /opt/nexus/etc/ssl/nexus.jks
```

`sudo nano /opt/nexus/etc/jetty/jetty-https.xml`

```
  <New id="sslContextFactory" class="org.eclipse.jetty.util.ssl.SslContextFactory$Server">
    <Set name="KeyStorePath"><Property name="ssl.etc"/>/nexus.jks</Set>
    <Set name="KeyStorePassword">admin@123</Set>
    <Set name="KeyManagerPassword">admin@123</Set>
    <Set name="TrustStorePath"><Property name="ssl.etc"/>/nexus.jks</Set>
    <Set name="TrustStorePassword">admin@123</Set>
    <Set name="EndpointIdentificationAlgorithm"></Set>
    <Set name="NeedClientAuth"><Property name="jetty.ssl.needClientAuth" default="false"/></Set>
    <Set name="WantClientAuth"><Property name="jetty.ssl.wantClientAuth" default="false"/></Set>
    <Set name="IncludeProtocols">
      <Array type="java.lang.String">
        <Item>TLSv1.2</Item>
      </Array>
    </Set>
  </New>
```

`sudo nano /opt/sonatype-work/nexus3/etc/nexus.properties`

```
# Jetty section
application-port=8081
application-port-ssl=8481
```

Start Nexus
`sudo systemctl start nexus.service`
