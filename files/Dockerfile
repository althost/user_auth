FROM alpine:latest
MAINTAINER R. Soto "https://numerica.cl"
RUN apk add --no-cache bash openssh
COPY entrypoint.sh /
RUN chmod +x /entrypoint.sh
EXPOSE 22
ENTRYPOINT ["/entrypoint.sh"]
