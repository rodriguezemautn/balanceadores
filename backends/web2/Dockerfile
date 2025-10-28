FROM nginx:alpine

# Copia el archivo index.html
COPY index.html /usr/share/nginx/html/index.html

# El puerto 80 ya está expuesto por la imagen base
EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]