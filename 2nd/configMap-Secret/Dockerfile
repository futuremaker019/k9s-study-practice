FROM --platform=linux/amd64 nginx:1.27.0

ENV TITLE NONE

CMD ["/bin/sh", "-c", "echo \"Hello, $TITLE\" > /usr/share/nginx/html/index.html && nginx -g 'daemon off;'"]