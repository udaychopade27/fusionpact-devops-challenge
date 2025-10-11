FROM nginx:alpine

WORKDIR /app

# Copy website files
COPY . /var/www/html

# Remove default config and use yours
RUN rm -rf /etc/nginx/conf.d/default.conf
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Permissions (optional but good practice)
RUN chmod -R 755 /var/www/html
RUN chown -R nginx:nginx /var/www/html  

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
