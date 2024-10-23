# JMX
## JVM options without tls:
```bash
-Dcom.sun.management.jmxremote=true
-Dcom.sun.management.jmxremote.local.only=false
-Dcom.sun.management.jmxremote.ssl=false
-Dcom.sun.management.jmxremote.authenticate=false
-Dcom.sun.management.jmxremote.port=12345
-Dcom.sun.management.jmxremote.rmi.port=12345
-Djava.rmi.server.hostname=10.1.2.3
```
## Опции JVM для двунаправленного TLS с логином и паролем:
```bash
-Dcom.sun.management.jmxremote.access.file=pathto/jmxremote.access
-Dcom.sun.management.jmxremote.authenticate=true #password and login authentication
-Dcom.sun.management.jmxremote.local.only=false 
-Dcom.sun.management.jmxremote.password.file=pathto/jmxremote.password 
-Dcom.sun.management.jmxremote.port=44444
-Dcom.sun.management.jmxremote.ssl=true
-Dcom.sun.management.jmxremate.ssl.need.client.auth=true #2way TLS
-Djava.rmi.server.hostname=localhost
-Djavax.netssl.keyStore=pathto/keystore.jks
-Djavax.net.ssl.keyStorePassword=password
-Djavax.net.ssl.trustStore=pathto/ks.jks
-Djavax.net.ssl.trustStorePassword=password
-Dcom.sun.management.jmxremote=true
```
Sample of access file content:
jmxUsr readwrite
Sample of  password file content:
jmxUsr <password>
In order for it to work correctly, you need to run it under a separate user in a container and give file reading rights only to this
user.