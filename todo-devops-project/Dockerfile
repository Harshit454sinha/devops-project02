# =============================================================
# Tasklane Todo App - Dockerfile
# Serves a static HTML/CSS/Bootstrap site with Nginx
# =============================================================

FROM nginx:1.27-alpine

# Remove the default Nginx welcome page
RUN rm -rf /usr/share/nginx/html/*

# Copy static site files into Nginx's web root
COPY index.html /usr/share/nginx/html/index.html
COPY style.css  /usr/share/nginx/html/style.css

# Nginx listens on port 80 by default
EXPOSE 80

# Base image already runs "nginx -g 'daemon off;'" as its default CMD,
# so the container starts Nginx automatically.
