# Use an official lightweight web server image
FROM nginx:alpine

# Copy the HTML file and image into the Nginx HTML directory
COPY login.html /usr/share/nginx/html/index.html
COPY nrl-logo.png /usr/share/nginx/html/

# Expose the default Nginx port
EXPOSE 80
