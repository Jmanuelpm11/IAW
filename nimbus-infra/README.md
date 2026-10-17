Practica IAW


I. Diseño y Requisitos

Antes de tocar una terminal, documentad en el README estas decisiones — la observación directa empezará precisamente por aquí:

•​ Los dos dominios ficticios elegidos (uno por marca) y por qué los habéis nombrado así:

    En cuanto a los dominios , en de la marca docs.nimbus.local para la publica y priv.nimbus.local , para la privada. He cooonsidera llamar así a mis dominios ya que me ha parecido una buena manera de llamarlos, ademas no incumplo 

•​ El MPM de Apache elegido y la justificación técnica de esa elección (repasad UD2,bloque de MPM).

 Usamos el MPM Event porque gestiona las peticiones concurrentes de manera asíncrona mediante hilos, optimizando el rendimiento frente a modelos más antiguos como worker. 
 
 
 
 - Disposicion de las carpetas utilizadas
 
 mkdir -p nimbus-infra/apache/conf nimbus-infra/apache/htdocs/public-docs nimbus-infra/apache/htdocs/priv-docs nimbus-infra/nginx/conf nimbus-infra/nginx/certs

nimbus-infra/
├── docker-compose.yml
├── apache/
│   ├── Dockerfile
│   └── conf/
│       ├── 00-default.conf
│       ├── docs-public.conf
│       └── priv-secure.conf
└── nginx/
    ├── Dockerfile
    ├── conf/
    │   └── nimbus-proxy.conf
    └── certs/
        ├── nimbus.crt
        └── nimbus.key
        
        
Bloque II: Fase 1 - Backend Apache Multi-Marca en Docker (UD2)

    Lo primero que hacemos es estrablecer en apache su Dockerfile , para ello uso cd , para poder posicionmarme en la carpeta , luego hago un nano Dockerfile y meto adentro esta linea (FROM httpd:2.4) que es para que coja una imagen de apache.
    
    
    - 2 Configuración de los Virtual Hosts
    
    Una vez ya lo tenga lo que voy a hacer luego es crear el fichero de nuestra marca publica, para ello me muevo a apache/conf y hago un nano docs-public.conf, tambien creo el de la marca privada nano priv-secure.conf.
    
    Para la parte publica use esta configuracion: 
    
    <VirtualHost *:8080>
    ServerName docs.nimbus.local
    DocumentRoot /usr/local/apache2/htdocs/public-docs
    ErrorLog /usr/local/apache2/logs/public-error.log
    CustomLog /usr/local/apache2/logs/public-access.log combined
    </VirtualHost>
    
    Para la privada esta otra:
    
    <VirtualHost *:8080>
    ServerName priv.nimbus.local
    DocumentRoot /usr/local/apache2/htdocs/priv-docs
    ErrorLog /usr/local/apache2/logs/priv-error.log
    CustomLog /usr/local/apache2/logs/priv-access.log combined

    <Directory /usr/local/apache2/htdocs/priv-docs>
        Options -Indexes
        Require all granted
    </Directory>

    <Files "nimbus-secrets.json">
        Require all denied
    </Files>
    </VirtualHost>
    
    
    - 3. Creación de Contenidos Web (htdocs)
    
    - Cuando acabemos de crear nuestras configuraciones para las marcas, lo que deberemos de hacer es crear las carpetas para el contenido htdocs, que sera donde guardaremos el contenido de nuestras paginas. 
    
    Para ello debemos posicionarnos en la carpeta de apache, y usamos el siguiente comando: mkdir -p htdocs/public-docs htdocs/priv-docs , esto hará que creemos dentro de htdocs, nuestro public-docs y priv-docs.
    
    Para poder meter contenenido lo que hago es usar este comando para hacerlo mas rapido , pero tambien podrias usar una nano Index.html sobre la carpeta de publics.docs, el comando es este: echo "<h1>Portal Publico de Documentacion - NimbusDocs</h1>" > htdocs/public-docs/index.html. Tambien hago lo mismo para el privado, usando este comando: echo "<h1>Portal Privado y Seguro - NimbusDocs</h1>" > htdocs/priv-docs/index.html
echo '{"token_secreto": "nimbus_secret_key_98765"}' > htdocs/priv-docs/nimbus-secrets.json
    
    - 4. Configuración del Fichero Principal httpd.conf
    
    - Luego lo que haremos sera configurar el archico httpd.conf para que apache pueda leer nuestras marcas. Para ello lo que hago es usar el comando cd ~/Desktop/nimbus-infra/apache/conf , para situarme, ahora hago nano httpd.conf , para crear el fichero donde voy a configurar y metemos esto : 
    
        ServerRoot "/usr/local/apache2"
Listen 8080

LoadModule mpm_event_module modules/mod_mpm_event.so
LoadModule authn_core_module modules/mod_authn_core.so
LoadModule authz_core_module modules/mod_authz_core.so
LoadModule unixd_module modules/mod_unixd.so
LoadModule log_config_module modules/mod_log_config.so
LoadModule mime_module modules/mod_mime.so
LoadModule dir_module modules/mod_dir.so
LoadModule alias_module modules/mod_alias.so
LoadModule authz_host_module modules/mod_authz_host.so

User daemon
Group daemon

DocumentRoot "/usr/local/apache2/htdocs"
<Directory "/usr/local/apache2/htdocs">
    AllowOverride none
    Require all granted
</Directory>

ErrorLog /proc/self/fd/2
CustomLog /proc/self/fd/1 common

Include conf/docs-public.conf
Include conf/priv-secure.conf


    III. Fase 2: Configuración del Proxy Inverso y TLS en Nginx (Frontend)

    Luego de meter la configuracion , ahora apache sabrá perfectamente donde tiene que ir para poder cargar los archivos de configuración de nuestras dos marcas.
    
    Ahora lo que voy a hacer es crear los certificados de nginx para ello uso el comando : 
    
    cd ~/Desktop/nimbus-infra
mkdir -p nginx/conf nginx/certs
openssl ecparam -name secp384r1 -genkey -noout -out nginx/certs/nimbus.key
openssl req -new -x509 -sha256 -key nginx/certs/nimbus.key -days 365 -out nginx/certs/nimbus.crt -subj "/CN=docs.nimbus.local"


    2. Configuración del Proxy Inverso
    
 y también creamos el acrchivo de configuracion de nuestro nginx: nano nginx/conf/nimbus-proxy.conf y dentro ponemos esta configuracion: 
 
 
 
 
 limit_req_zone $binary_remote_addr zone=nimbus_zone:10m rate=6r/s;

server {
    listen 80;
    server_name docs.nimbus.local priv.nimbus.local;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name docs.nimbus.local priv.nimbus.local;

    ssl_certificate /etc/nginx/certs/nimbus.crt;
    ssl_certificate_key /etc/nginx/certs/nimbus.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;

    location / {
        limit_req zone=nimbus_zone burst=4 nodelay;
        proxy_pass http://apache-backend:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}



    IV. Fase 3: Despliegue y Orquestación
        
    Una vez allamos terminado , ahora lo que haremos sera crear nuestro docker compose , por lo que nos vamos a la raiz de nuestra carpeta y creamos un docker coompose con nano o usando visual studio y ponemos esto:
    
    
    version: '3.8'

services:
  apache-backend:
    build: ./apache
    container_name: nimbus-apache
    networks:
      - nimbus-net
    volumes:
      - ./apache/conf/docs-public.conf:/usr/local/apache2/conf/docs-public.conf
      - ./apache/conf/priv-secure.conf:/usr/local/apache2/conf/priv-secure.conf
      - ./apache/conf/httpd.conf:/usr/local/apache2/conf/httpd.conf
      - ./apache/htdocs:/usr/local/apache2/htdocs

  nginx-proxy:
    image: nginx:latest
    container_name: nimbus-nginx
    ports:
      - "80:80"
      - "443:443"
    networks:
      - nimbus-net
    volumes:
      - ./nginx/conf:/etc/nginx/conf.d
      - ./nginx/certs:/etc/nginx/certs
    depends_on:
      - apache-backend

networks:
  nimbus-net:
    driver: bridge
    
    2. Resolución Local y Arranque de Contenedores
    
    ahora lo que hago es añadir mis dominios a /etc/host , para que mi maquina resuelva mis marcas localmente hacia docker
    
    echo "127.0.0.1 docs.nimbus.local priv.nimbus.local" | sudo tee -a /etc/hosts
    
    
    
    V. Fase 4: Pruebas de Verificación
    
    Luego hago un docker compose up -d --build para arrancar al estructura y por ultimo compruebo que todo este bien , para ello uso estos comandos: 
    curl -k https://docs.nimbus.local/
    curl -k https://priv.nimbus.local/
    
Los cuales me dan lo esta dentro de mis archivos .html

wlw@Ubuntu:~/Desktop/nimbus-infra$ curl -k https://docs.nimbus.local/
<h1>Portal Publico de Documentacion - NimbusDocs</h1>
wlw@Ubuntu:~/Desktop/nimbus-infra$ curl -k https://priv.nimbus.local/
<h1>Portal Privado y Seguro - NimbusDocs</h1>


     Y ya tendríamos la practica finalizada y bien configurada.
     
     

    - Comprobaciones 
    
    ​

​
•​ Demostrad con una ráfaga de peticiones repetidas (un bucle for con curl, o ab desde
un contenedor cliente) que el límite calculado realmente bloquea, adjuntando la salida.
    
    
    wlw@Ubuntu:~/Desktop/IAW/nimbus-infra$ for i in {1..15}; do curl -k -s -o /dev/null -w "%{http_code}\n" https://docs.nimbus.local/; done
200
200
200
200
200
200
200
503
503
503
503
503
503
503
503



    •​ Cabeceras de seguridad mínimas: Strict-Transport-Security,
X-Content-Type-Options y una tercera que elijáis y justifiquéis vosotros (repasad UD1,tabla de cabeceras).

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "DENY" always;

    he elegido X-Frame-Options por proteccion ante Clickjacking










