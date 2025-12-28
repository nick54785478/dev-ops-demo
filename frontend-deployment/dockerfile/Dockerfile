FROM nginx:stable-alpine
WORKDIR /usr/share/nginx/html
COPY dist/rbac-frontend/browser /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf
COPY nginx.gzip.conf /etc/nginx/conf.d/gzip.conf
COPY nginx.site.conf /etc/nginx/conf.d/default.conf
EXPOSE 80/tcp
EXPOSE 443/tcp
CMD ["/bin/sh", "-c", "exec nginx -g 'daemon off;'"]