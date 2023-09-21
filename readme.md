## First Run

From the nginx_https_docker folder on your server, run the command

```bash
docker compose up --build nginx
```

this will start our nginx server. Leave it running and you will also be able to see the logs here. Probably try accessing your server using the IP address or using http://yourdomain. Make sure you use http when using both. This should show you the nginx home page or a 404 not found page.

```notes
If at this point you get an error like
ERROR: for nginx Cannot create container for service nginx: Conflict. The container name “/nginx-service” is already in use by container “31bd6a36f4ecc7a43f1ea991c7686ae60a4ddb13c749b9c5b848cbacb69e89d5”. You have to remove (or rename) that container to be able to reuse that name.
ERROR: Encountered errors while bringing up the project.

then run this command to remove old container:
docker stop nginx-service && docker container rm nginx-service
```

Now in another terminal window go to the folder nginx_https_docker and run the letsencrypt container with the below command

```bash
docker compose -f docker-compose-le.yaml up --build
```

if things work well you will see something like this

```bash
Successfully received certificate.
letsencrypt_1  | Certificate is saved at: /etc/letsencrypt/live/yourdomain.com/fullchain.pem
letsencrypt_1  | Key is saved at:         /etc/letsencrypt/live/yourdomain.com/privkey.pem
letsencrypt_1  | This certificate expires on 2021-10-20.
letsencrypt_1  | These files will be updated when the certificate renews.
letsencrypt_1  |
letsencrypt_1  | NEXT STEPS:
letsencrypt_1  | - The certificate will need to be renewed before it expires. Certbot can automatically renew the certificate in the background, but you may need to take steps to enable that functionality. See https://certbot.org/renewal-setup for instructions.
letsencrypt_1  |
letsencrypt_1  | - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
letsencrypt_1  | If you like Certbot, please consider supporting our work by:
letsencrypt_1  |  * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
letsencrypt_1  |  * Donating to EFF:                    https://eff.org/donate-le
letsencrypt_1  | - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

Generated certificates will be available under `/etc/letsencrypt/<your domain name>` directory on your machine.

The above container stops itself after installing the certificates. For the other container running in the first terminal, stop it by pressing Ctrl+C or CMD+C.

## Enable Lets Encrypt

Open the config/nginx.conf again and delete everything. Paste below contents and remember again to remember to replacetest.leangaurav.com with your domain name at all the four places.

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name your-domain.com;
    location / {
        return 301 https://$host$request_uri;
    }
    location ~ /.well-known/acme-challenge {
        allow all;
        root /tmp/acme_challenge;
    }
}
server {
    listen 443 ssl;
    listen [::]:443 ssl http2;
    server_name your-domain.com;
    ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;
}
```

Now run the nginxcontainer again but with the -d flag this time.

```bash
docker compose up --build -d nginx
```

Now navigate to your domain and you should find an nginx 404 page served over https.

## Reverse Proxy (Optional)

All these things can be customized according to your needs. What I have provided is just an example.

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name your-domain.com;
    location / {
        return 301 https://$host$request_uri;
    }
    location ~ /.well-known/acme-challenge {
        allow all;
        root /tmp/acme_challenge;
    }
}
server {
    listen 443 ssl;
    listen [::]:443 ssl http2;
    server_name your-domain.com;
    ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;


        location / {
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header Host $host;
                proxy_set_header X-NginX-Proxy true;
                proxy_pass http://your-ip;
                proxy_redirect http://your-ip https://$server_name;
        }
}
```
