# synapse-element-postgres
Docker-Compose for Matrix-Synapse, Element-Web with postgres Database with Caddy Reverse Proxy

# Server Setup

  Update
    Ubuntu/Debian:  
    ```sudo apt update && sudo apt upgrade```  
    CentOS/Fedora:  
    ```sudo dnf upgrade```  

  Install automatic updates
    Ubuntu/Debian:  
    ```sudo apt install unattended-upgrades```   
    CentOS/Fedora:  
    ```sudo dnf install -y dnf-automatic```  

  Change SSH Port:  
  ```sudo nano /etc/ssh/sshd_config```  

Remove the # infront of Port 22 and then change it (30000-50000 is ideal).
This is security though obsucurity which is not ideal but port 22 just gets abused by bots.

Also Disable root login by changing  
   ```#permitrootlogin true```  
    ```permitrootlogin false```  
    
Restart SSH:  
   ```sudo systemctl restart sshd```  

Install fail2ban
    Ubuntu/Debian:  
    ```sudo apt install fail2ban```  
    CentOS/Fedora:  
    ```sudo dnf install fail2ban```
    
Install UFW Firewall

  Install
    Ubuntu/Debian:  
    ```sudo apt install ufw```  
    CentOS/Fedora:  
    ```sudo dnf install ufw```  

Replace SSH-PORT to your SSH port:  
```sudo ufw allow <SSH-PORT>/tcp```  

Allow HTTP/s traffic:  
  
  ```sudo ufw allow 80/tcp```  
  ```sudo ufw allow 443/tcp```    

    Enable Firewall: sudo ufw enable

Setup a sudo user  
    ```adduser <USERNAME>```  
    Add user to sudoers  
    ```sudo adduser <USERNAME> sudo```  
    Login as the new user  
    ```su - <USERNAME>```  

Install Docker  

Ubuntu/Debian  
```curl -fsSL https://get.docker.com -o get-docker.sh ```  
```sh get-docker.sh```  

Install Matrix and Element

   Create docker network, this is so Matrix and Element can be on their own isolated network:
   ```sudo docker network create --driver=bridge --subnet=10.10.10.0/24 --gateway=10.10.10.1 matrix_net```  

   Create Matrix directory:  
   ```sudo mkdir matrix```  
Use the following template:  
   ```sudo nano docker-compose.yaml```  
```
version: '2.3'
services:
  postgres:
    image: postgres:14
    restart: unless-stopped
    networks:
      default:
        ipv4_address: 10.10.10.2
    volumes:
     - ./postgresdata:/var/lib/postgresql/data

    # These will be used in homeserver.yaml later on
    environment:
     - POSTGRES_DB=synapse
     - POSTGRES_USER=synapse
     - POSTGRES_PASSWORD=STRONGPASSWORD
     
  element:
    image: vectorim/element-web:latest
    restart: unless-stopped
    volumes:
      - ./element-config.json:/app/config.json
    networks:
      default:
        ipv4_address: 10.10.10.3
        
  synapse:
    image: matrixdotorg/synapse:latest
    restart: unless-stopped
    networks:
      default:
        ipv4_address: 10.10.10.4
    volumes:
     - ./synapse:/data

networks:
  default:
      name: matrix_net
```  

Create Element Config  
```sudo nano element-config.json```  

Copy and paste example contents into your file.  

```
{
    "default_server_name": "matrix.org",
    "brand": "Element",
    "integrations_ui_url": "https://scalar.vector.im/",
    "integrations_rest_url": "https://scalar.vector.im/api",
    "integrations_widgets_urls": [
        "https://scalar.vector.im/_matrix/integrations/v1",
        "https://scalar.vector.im/api",
        "https://scalar-staging.vector.im/_matrix/integrations/v1",
        "https://scalar-staging.vector.im/api",
        "https://scalar-staging.riot.im/scalar/api"
    ],
    "hosting_signup_link": "https://element.io/matrix-services?utm_source=element-web&utm_medium=web",
    "bug_report_endpoint_url": "https://element.io/bugreports/submit",
    "uisi_autorageshake_app": "element-auto-uisi",
    "showLabsSettings": true,
    "piwik": {
        "url": "https://piwik.riot.im/",
        "siteId": 1,
        "policyUrl": "https://element.io/cookie-policy"
    },
    "roomDirectory": {
        "servers": [
            "matrix.org",
            "gitter.im",
            "libera.chat"
        ]
    },
    "enable_presence_by_hs_url": {
        "https://matrix.org": false,
        "https://matrix-client.matrix.org": false
    },
    "terms_and_conditions_links": [
        {
            "url": "https://element.io/privacy",
            "text": "Privacy Policy"
        },
        {
            "url": "https://element.io/cookie-policy",
            "text": "Cookie Policy"
        }
    ],
    "hostSignup": {
      "brand": "Element Home",
      "cookiePolicyUrl": "https://element.io/cookie-policy",
      "domains": [
          "matrix.org"
      ],
      "privacyPolicyUrl": "https://element.io/privacy",
      "termsOfServiceUrl": "https://element.io/terms-of-service",
      "url": "https://ems.element.io/element-home/in-app-loader"
    },
    "sentry": {
        "dsn": "https://029a0eb289f942508ae0fb17935bd8c5@sentry.matrix.org/6",
        "environment": "develop"
    },
    "posthog": {
        "projectApiKey": "phc_Jzsm6DTm6V2705zeU5dcNvQDlonOR68XvX2sh1sEOHO",
        "apiHost": "https://posthog.hss.element.io"
    },
    "features": {},
    "map_style_url": "https://api.maptiler.com/maps/streets/style.json?key=fU3vlMsMn4Jb6dnEIFsx"
}

```  

Remove "default_server_name": "matrix.org" from element-config.json as this is deprecated  

Add our custom homeserver to the top of element-config.json:  

  ```
  "default_server_config": {
      "m.homeserver": {
          "base_url": "https://matrix.example.com",
          "server_name": "matrix.example.com"
      },
      "m.identity_server": {
          "base_url": "https://vector.im"
      }
  },
```  

Generate Synapse Config:  
```
sudo docker run -it --rm \
    -v "$HOME/matrix/synapse:/data" \
    -e SYNAPSE_SERVER_NAME=matrix.example.com \
    -e SYNAPSE_REPORT_STATS=yes \
    matrixdotorg/synapse:latest generate
```  
    
Comment out sqlite database (as we have setup postgres to replace this) in synapse/homeserver.yaml:  
```
#database:
#  name: sqlite3
#  args:
#    database: /data/homeserver.db
```  
Add the Postgres config to synapse/homeserver.yaml  

```database:
  name: psycopg2
  args:
    user: synapse
    password: STRONGPASSWORD
    database: synapse
    host: postgres
    cp_min: 5
    cp_max: 10
```
Deploy: sudo docker-compose up -d  

Create New Users  
Access docker shell:  
```sudo docker exec -it matrix_synapse_1 bash```  
```register_new_matrix_user -c /data/homeserver.yaml http://localhost:8008```

Follow the on screen prompts

Enter ``exit`` to leave the container's shell

To allow anyone to register an account set 'enable_registration' to true in the homeserver.yaml. 
This is NOT recomended.


# Install Reverse Proxy (Caddy)

Caddy will be used for the reverse proxy. This will handle incomming HTTPS connections and forward them to the correct docker containers. It a simple setup process and Caddy will automatically fetch and renew Let's Encrypt certificates for us!

remove current Caddyfile  
```sudo nano rm /etc/caddy/Caddyfile```

create a new Caddyfile and copy the following code.  
```sudo nano /etc/caddy/Caddyfile```


```matrix.example.com {
  reverse_proxy /_matrix/* 10.10.10.4:8008
  reverse_proxy /_synapse/client/* 10.10.10.4:8008
  
  header {
    X-Content-Type-Options nosniff
    Referrer-Policy  strict-origin-when-cross-origin
    Strict-Transport-Security "max-age=63072000; includeSubDomains;"
    Permissions-Policy "accelerometer=(), camera=(), geolocation=(), gyroscope=(), magnetometer=(), microphone=(), payment=(), usb=(), interest-cohort=()"
    X-Frame-Options SAMEORIGIN
    X-XSS-Protection 1
    X-Robots-Tag none
    -server
  }
}

element.example.com {
  encode zstd gzip
  reverse_proxy 10.10.10.3:80

  header {
    X-Content-Type-Options nosniff
    Referrer-Policy  strict-origin-when-cross-origin
    Strict-Transport-Security "max-age=63072000; includeSubDomains;"
    Permissions-Policy "accelerometer=(), camera=(), geolocation=(), gyroscope=(), magnetometer=(), microphone=(), payment=(), usb=(), interest-cohort=()"
    X-Frame-Options SAMEORIGIN
    X-XSS-Protection 1
    X-Robots-Tag none
    -server
  }
}
``` 

Enable the config:  
    ```caddy reload```

Login

Head to your element domain and login!

Don't forget to update every now and then

Pull the new docker images and then restart the containers:  
```sudo docker-compose pull && sudo docker-compose up -d```
