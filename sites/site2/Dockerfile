FROM nginx:alpine

# Copier les fichiers du site
COPY . /usr/share/nginx/html/

# Exposer le port 80
EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]