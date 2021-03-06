FROM debian:stable-slim as builder

#Set mosquitto and plugin versions.
#Change them for your needs.
ENV MOSQUITTO_VERSION=1.6.7
ENV PLUGIN_COMMIT_ID=bbb9ee9e802f59dc99dd16876c487a1bed900db1
ENV GO_VERSION=1.12.12

WORKDIR /app

#Get mosquitto build dependencies.
RUN apt-get update && apt-get install -y libwebsockets8 libwebsockets-dev libc-ares2 libc-ares-dev openssl uuid uuid-dev wget build-essential git
RUN mkdir -p mosquitto/auth mosquitto/conf.d

RUN wget http://mosquitto.org/files/source/mosquitto-${MOSQUITTO_VERSION}.tar.gz
RUN tar xzvf mosquitto-${MOSQUITTO_VERSION}.tar.gz && rm mosquitto-${MOSQUITTO_VERSION}.tar.gz 

#Build mosquitto.
RUN cd mosquitto-${MOSQUITTO_VERSION} && make WITH_WEBSOCKETS=yes && make install && cd ..

#Get Go.
RUN wget https://dl.google.com/go/go${GO_VERSION}.linux-amd64.tar.gz && tar -C /usr/local -xzf go${GO_VERSION}.linux-amd64.tar.gz
RUN export PATH=$PATH:/usr/local/go/bin && go version && rm go${GO_VERSION}.linux-amd64.tar.gz

#Get the plugin.
RUN git clone https://github.com/iegomez/mosquitto-go-auth.git && \
    cd mosquitto-go-auth && \
    git checkout ${PLUGIN_COMMIT_ID} && \
    mv * ..

#Build the plugin.
RUN export PATH=$PATH:/usr/local/go/bin && export CGO_CFLAGS="-I/usr/local/include -fPIC" && export CGO_LDFLAGS="-shared" &&  make

#Start from a new image.
FROM debian:stable-slim

#Get mosquitto dependencies.
RUN apt-get update && apt-get install -y libwebsockets8 libc-ares2 openssl uuid curl

#Setup mosquitto env.
RUN mkdir -p /var/lib/mosquitto /var/log/mosquitto 
RUN groupadd mosquitto \
    && useradd -s /sbin/nologin mosquitto -g mosquitto -d /var/lib/mosquitto \
    && chown -R mosquitto:mosquitto /var/log/mosquitto/ \
    && chown -R mosquitto:mosquitto /var/lib/mosquitto/

#Copy confs, plugin so and mosquitto binary.
#COPY --from=builder /app/mosquitto/ /mosquitto/
COPY --from=builder /app/go-auth.so /usr/local/lib/auth-plug.so
COPY --from=builder /usr/local/sbin/mosquitto /usr/local/sbin/mosquitto

VOLUME ["/var/lib/mosquitto"]

EXPOSE 1883 8883 9883

ADD mosquitto.conf /etc/mosquitto/mosquitto.conf
ADD run.sh /run.sh
RUN chmod +x /run.sh

ENTRYPOINT ["/run.sh"]
CMD ["mosquitto"]