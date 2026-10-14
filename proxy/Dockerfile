# Proxy image inherited from the previous team.

FROM ubuntu:24.04


ARG DEBIAN_FRONTEND=noninteractive
RUN apt-get update && \
    apt-get install -y nginx curl htop vim git

COPY nginx.conf /etc/nginx/nginx.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]


### Bonus Question 1.

# FROM nginx:alpine

# COPY nginx.conf /etc/nginx/nginx.conf

# EXPOSE 80

# CMD ["nginx", "-g", "daemon off;"]


### Bonus Question 2.

# FROM nginx:alpine

# RUN touch /var/run/nginx.pid && \
#     chown -R nginx:nginx /var/run/nginx.pid /var/cache/nginx /etc/nginx

# COPY nginx.conf /etc/nginx/nginx.conf

# USER nginx

# EXPOSE 8080

# CMD ["nginx", "-g", "daemon off;"]