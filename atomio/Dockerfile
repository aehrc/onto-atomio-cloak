FROM quay.io/aehrc/atomio:latest

COPY server.crt /etc/ssl/certs/ontocloak_demo.crt

RUN keytool -import -alias ontocloak_crt -file /etc/ssl/certs/ontocloak_demo.crt -noprompt -storepass changeit -keystore /usr/lib/jvm/java-17-openjdk/lib/security/truststore

CMD ["/bin/bash"]