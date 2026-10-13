Pregunta sobre el ls salida del comando openssl s_client

Certificate chain
 0 s:CN=tudominio1.local
   i:CN=tudominio1.local
   a:PKEY: EC, (secp384r1); sigalg: ecdsa-with-SHA256
   v:NotBefore: Oct 13 01:28:38 2026 GMT; NotAfter: Oct 13 01:28:38 2027 GMT
---
Server certificate
-----BEGIN CERTIFICATE-----
MIIByDCCAU6gAwIBAgIUILAnWyX6MfGT0meNxcSHGXF2zFowCgYIKoZIzj0EAwIw
GzEZMBcGA1UEAwwQdHVkb21pbmlvMS5sb2NhbDAeFw0yNjEwMTMwMTI4MzhaFw0y
NzEwMTMwMTI4MzhaMBsxGTAXBgNVBAMMEHR1ZG9taW5pbzEubG9jYWwwdjAQBgcq
hkjOPQIBBgUrgQQAIgNiAAQatz8kvAyS4AvmlirZSLcMZLanZhEeKFFz2corTUa3
85yVsHJwNxnS/rOnc/v6/w8h+KldiGv3b9xpX5GfwuMu3jYfHhTdl/U61hHSb1zY
3/oW8V1l171Lxc3N82UbHNSjUzBRMB0GA1UdDgQWBBRPeirrQ50/QEm/Q2TCsrux
demIFzAfBgNVHSMEGDAWgBRPeirrQ50/QEm/Q2TCsruxdemIFzAPBgNVHRMBAf8E
BTADAQH/MAoGCCqGSM49BAMCA2gAMGUCMQCPH3l4iyhBAmIgcQVgaXbqXb0Bp9GN
7OBYjAuYlLTq6h2JZecBUenyJDtNxYyXf+ECMGoczq0om5PdzMhNYrs/DPZJzoaY
Az1r/SkRfoEcj+NFoR1xxMpvf9a5Il8kwV5Hzg==
-----END CERTIFICATE-----
subject=CN=tudominio1.local
issuer=CN=tudominio1.local
---
Peer signing digest: SHA384
Peer signature type: ecdsa_secp384r1_sha384
Negotiated TLS1.3 group: X25519MLKEM768
---
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
Protocol: TLSv1.3
Server public key is 384 bit


2. Salida del bucle de pruebas del Rate Limiting

https://tudominio1.local; done
Peticion 1 -> HTTP Status: 301
Peticion 2 -> HTTP Status: 301
Peticion 3 -> HTTP Status: 301
Peticion 4 -> HTTP Status: 301
Peticion 5 -> HTTP Status: 301
Peticion 6 -> HTTP Status: 301
Peticion 7 -> HTTP Status: 301
Peticion 8 -> HTTP Status: 503
Peticion 9 -> HTTP Status: 503
Peticion 10 -> HTTP Status: 503
Peticion 11 -> HTTP Status: 503
Peticion 12 -> HTTP Status: 503
Peticion 13 -> HTTP Status: 503
Peticion 14 -> HTTP Status: 503
Peticion 15 -> HTTP Status: 503
Peticion 16 -> HTTP Status: 503
Peticion 17 -> HTTP Status: 503
Peticion 18 -> HTTP Status: 503
Peticion 19 -> HTTP Status: 503
Peticion 20 -> HTTP Status: 503

Esta es la prueba definitiva de que la arquitectura completa (Nginx + TLS + Proxy Inverso + Apache Multi-Marca) está perfectamente integrada y enlazada

curl -k -L https://tudominio1.local
curl -k -L https://tudominio2.local
<html>
<head><title>503 Service Temporarily Unavailable</title></head>
<body>
<center><h1>503 Service Temporarily Unavailable</h1></center>
<hr><center>nginx/1.28.3 (Ubuntu)</center>
</body>
</html>
<html>
<head><title>503 Service Temporarily Unavailable</title></head>
<body>
<center><h1>503 Service Temporarily Unavailable</h1></center>
<hr><center>nginx/1.28.3 (Ubuntu)</center>
</body>
</html>



| Síntoma observado | Hipótesis inicial | Comando de diagnóstico usado | Causa real y solución |
| :--- | :--- | :--- | :--- |
| **Apache:** Al intentar recargar el servicio con `systemctl reload apache2`, el sistema devuelve un error de fallo y los sitios dejan de responder. | Se ha corrompido el puerto de escucha o hay un conflicto de direcciones IP en el Virtual Host. | `sudo apache2ctl configtest` o `sudo journalctl -u apache2 -e` | **Causa real:** Error de sintaxis al introducir una directiva no reconocida (`DocumentRootXYZ`) en el fichero de configuración del Virtual Host[cite: 6].<br>**Solución:** Corregir el nombre de la directiva a `DocumentRoot` y recargar de nuevo[cite: 6]. |
| **Nginx:** Al ejecutar `sudo nginx -t` o intentar reiniciar el servicio, Nginx muestra un error crítico y no arranca[cite: 5]. | El puerto 443 está siendo ocupado por otra aplicación del sistema operativo. | `sudo nginx -t` o `sudo systemctl status nginx`[cite: 5, 6] | **Causa real:** La ruta del fichero del certificado SSL (`ssl_certificate`) en el bloque `server` apuntaba a un archivo inexistente por un error tipográfico[cite: 5].<br>**Solución:** Restaurar la ruta correcta del certificado en el archivo de configuración de Nginx[cite: 5]. |


FALLO APACHE:

Antes modificación para el fallo;

<VirtualHost 127.0.0.1:8080>
    ServerName tudominio1.local
    DocumentRoot /var/www/tu_marca_1/html
    ErrorLog ${APACHE_LOG_DIR}/error_marca1.log
    CustomLog ${APACHE_LOG_DIR}/access_marca1.lo>
</VirtualHost>


Despues de la modificación para el fallo 

<VirtualHost 127.0.0.1:8080>
    ServerName tudominio1.local
    DocumentRootXYZ /var/www/tu_marca_1/html
    ErrorLog ${APACHE_LOG_DIR}/error_marca1.log
    CustomLog ${APACHE_LOG_DIR}/access_marca1.lo>
</VirtualHost>

FALLO:

Job for apache2.service failed.
See "systemctl status apache2.service" and "journalctl -xeu apache2.service" for details.



FALLO NGINX:

Modificación nginx sudo nano /etc/nginx/sites-available/nimbus-proxy.conf:

# Definir la zona de límite de peticiones (fuera del bloque server)
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=3r/s;

# Servidor para redirigir todo el tráfico HTTP (puerto 80) a HTTPS (puerto 443)
server {
    listen 80;
    server_name tudominio1.local tudominio2.local;

    # Redirección permanente (código 301)
    return 301 https://$host$request_uri;
}

# Servidor principal con terminación TLS y proxy inverso hacia Apache
server {
    listen 443 ssl;
    server_name tudominio1.local tudominio2.local;

    # Certificado y clave privada ECDSA generados
    ssl_certificate /etc/ssl/certs/nginx-ecdsa.crt;
    ssl_certificate_key /etc/ssl/private/nginx-ecdsa.key;

    # Restringir estrictamente a los protocolos seguros TLS 1.2 y TLS 1.3
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # Cabeceras de seguridad perimetral requeridas
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options nosniff always;
    add_header X-Frame-Options SAMEORIGIN always;

    location / {
        # Aplicación del límite de peticiones (rate limiting con ráfaga permitida de 6)
        limit_req zone=api_limit burst=6 nodelay;

        # Reenvío de peticiones al backend Apache local preservando la cabecera Host


Despues:

# Definir la zona de límite de peticiones (fuera del bloque server)
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=3r/s;

# Servidor para redirigir todo el tráfico HTTP (puerto 80) a HTTPS (puerto 443)
server {
    listen 80;
    server_name tudominio1.local tudominio2.local;

    # Redirección permanente (código 301)
    return 301 https://$host$request_uri;
}

# Servidor principal con terminación TLS y proxy inverso hacia Apache
server {
    listen 443 ssl;
    server_name tudominio1.local tudominio2.local;

    # Certificado y clave privada ECDSA generados
    ssl_certificate /etc/ssl/certs/error.certificado.crt;
    ssl_certificate_key /etc/ssl/private/nginx-ecdsa.key;

    # Restringir estrictamente a los protocolos seguros TLS 1.2 y TLS 1.3
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # Cabeceras de seguridad perimetral requeridas
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options nosniff always;
    add_header X-Frame-Options SAMEORIGIN always;

    location / {
        # Aplicación del límite de peticiones (rate limiting con ráfaga permitida de 6)
        limit_req zone=api_limit burst=6 nodelay;

        # Reenvío de peticiones al backend Apache local preservando la cabecera Host


Fallo:

2026/10/13 03:53:30 [emerg] 71950#71950: cannot load certificate "/etc/ssl/certs/error.certificado.crt": BIO_new_file() failed (SSL: error:80000002:system library::No such file or directory:calling fopen(/etc/ssl/certs/error.certificado.crt, r) error:10000080:BIO routines::no such file)
nginx: configuration file /etc/nginx/nginx.conf test failed