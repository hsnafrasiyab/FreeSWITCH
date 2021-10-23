FROM debian:buster

RUN apt-get update && apt-get install -yq gnupg2 wget lsb-release 
RUN apt-get install -y init && apt-get clean all
RUN wget -O - https://files.freeswitch.org/repo/deb/debian-release/fsstretch-archive-keyring.asc | apt-key add - 

RUN    echo "deb http://files.freeswitch.org/repo/deb/debian-release/ `lsb_release -sc` main" > /etc/apt/sources.list.d/freeswitch.list 
RUN    echo "deb-src http://files.freeswitch.org/repo/deb/debian-release/ `lsb_release -sc` main" >> /etc/apt/sources.list.d/freeswitch.list 

RUN apt-get update && apt-get install -y vim  
RUN  apt-get build-dep freeswitch -y

RUN apt-get install -y git

WORKDIR  /usr/src/

RUN cd /usr/src 
RUN git clone https://github.com/signalwire/freeswitch.git -bv1.10 freeswitch 
RUN cd freeswitch 
WORKDIR  /usr/src/freeswitch/

RUN git config pull.rebase true

RUN ./bootstrap.sh -j

COPY modules.conf /usr/src/freeswitch/ 

COPY switch_rtp.c /usr/src/freeswitch/src/

RUN ./configure 
RUN  make 
RUN   make install

RUN /usr/sbin/groupadd freeswitch 
RUN /usr/sbin/adduser --quiet --system --home /usr/local/freeswitch --gecos "FreeSWITCH open source softswitch" --ingroup freeswitch freeswitch --disabled-password


RUN chown -R freeswitch:freeswitch /usr/local/freeswitch/ 
RUN chmod -R ug=rwX,o= /usr/local/freeswitch/ 
RUN chmod -R u=rwx,g=rx /usr/local/freeswitch/bin/* 
RUN ln -s /usr/local/freeswitch/bin/freeswitch /usr/bin/

RUN ln -s /usr/local/freeswitch/bin/fs_cli /usr/bin 

VOLUME ["/usr/local/freeswitch/"]

EXPOSE 8021/tcp
EXPOSE 5060/tcp 5060/udp 5080/tcp 5080/udp
EXPOSE 5061/tcp 5061/udp 5081/tcp 5081/udp
EXPOSE 7443/tcp
EXPOSE 5070/udp 5070/tcp
EXPOSE 64535-65535/udp
EXPOSE 16384-32768/udp

CMD ["freeswitch"]
