FROM httpd:2.4-alpine

# Valeurs par defaut, surchargeables au lancement (-e ou docker-compose)
ENV BACKEND_HOST=backend \
    BACKEND_PORT=8080

COPY index.html /usr/local/apache2/htdocs/

# On garde le httpd.conf par defaut de l'image et on y inclut uniquement notre configuration proxy
COPY proxy.conf /usr/local/apache2/conf/extra/proxy.conf
RUN echo "Include conf/extra/proxy.conf" >> /usr/local/apache2/conf/httpd.conf
