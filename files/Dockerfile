FROM alpine:latest
MAINTAINER R. Soto "https://numerica.cl"

# install packages
RUN apk add --no-cache bash openssh

# define main process
COPY entrypoint.sh /
RUN chmod +x /entrypoint.sh

# delete default empty password to unlock root
RUN passwd -d root

# setup root ssh access
COPY ssh/sysadmin.pub /root/.ssh/authorized_keys

# ensure root login is allowed
COPY ssh/sshd_config /etc/ssh/sshd_config

# declare ssh port
EXPOSE 22

ENTRYPOINT ["/entrypoint.sh"]
