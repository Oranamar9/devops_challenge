# Official nginx image, small Alpine version
FROM nginx:1.30-alpine

# Replace the default config with ours
COPY nginx.conf /etc/nginx/nginx.conf

# Give the nginx user ownership of the folder nginx writes temp files to
RUN chown -R nginx:nginx /var/cache/nginx

# Run as the "nginx" user instead of root
USER nginx

EXPOSE 80

# Start nginx in the foreground, with its PID file in /tmp (a non-root user can write there)
CMD ["nginx", "-g", "pid /tmp/nginx.pid; daemon off;"]