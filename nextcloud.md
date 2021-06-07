# Nextcloud in Docker with WSL 2 and letsencrypt

Run Nextcloud in Docker on Windows using the WSL 2 framework and letsencrypt for HTTPS security. 

# Prerequisites

* WSL 2 setup on your Windows 10 system. 
https://docs.microsoft.com/en-us/windows/wsl/install-win10

  It should look like this: 
  ```
  PS C:\Users\alex> wsl -l -v
    NAME                   STATE           VERS
    * Ubuntu-20.04           Running         2
  ```

* A DNS address to use with the letsencrypt container. A free duckdns.org address is great. 
* Docker Desktop for Windows installed. 
* Local storage location for all the Nextcloud and related data. Think, where you want all the photos, etc from Nextcloud to be stored. 

# Setup

1. Create a folder in your desired storage location, named something like "nextcloud"
2. Create an file called `docker-compose.yml` in that directory. 
3. Edit the file with your favorite editor with the following contents: 
	```
	version: '3'  

    services:
        proxy:
            image: jwilder/nginx-proxy:alpine
            labels:
	            - "com.github.jrcs.letsencrypt_nginx_proxy_companion.nginx_proxy=true"
            container_name: nextcloud-proxy
            networks:
	            - nextcloud_network
            ports:
	            - 80:80
	            - 443:443
            volumes:
	            - ./proxy/conf.d:/etc/nginx/conf.d:rw
	            - ./proxy/vhost.d:/etc/nginx/vhost.d:rw
	            - ./proxy/html:/usr/share/nginx/html:rw
	            - ./proxy/certs:/etc/nginx/certs:ro
	            - /etc/localtime:/etc/localtime:ro
	            - /var/run/docker.sock:/tmp/docker.sock:ro
            restart: unless-stopped
        letsencrypt:
            image: jrcs/letsencrypt-nginx-proxy-companion
            container_name: nextcloud-letsencrypt
            depends_on:
	            - proxy
            networks:
	            - nextcloud_network
            volumes:
	            - ./proxy/certs:/etc/nginx/certs:rw
	            - ./proxy/vhost.d:/etc/nginx/vhost.d:rw
	            - ./proxy/html:/usr/share/nginx/html:rw
	            - /etc/localtime:/etc/localtime:ro
	            - /var/run/docker.sock:/var/run/docker.sock:ro
            restart: unless-stopped
         db:
            image: mariadb
            container_name: nextcloud-mariadb
            networks:
	            - nextcloud_network
            volumes:
	            - db:/var/lib/mysql
	            - /etc/localtime:/etc/localtime:ro
            environment:
	            - MYSQL_ROOT_PASSWORD=FIX_WITH_PASSWORD
	            - MYSQL_PASSWORD=FIX_WITH_PASSWORD
	            - MYSQL_DATABASE=nextcloud
	            - MYSQL_USER=nextcloud
            restart: unless-stopped     
        app:
            image: nextcloud:latest
            container_name: nextcloud-app
            networks:
	            - nextcloud_network
            depends_on:
	            - letsencrypt
	            - proxy
	            - db
            volumes:
	            - nextcloud:/var/www/html
	            - ./app/config:/var/www/html/config
	            - ./app/custom_apps:/var/www/html/custom_apps
	            - ./app/data:/var/www/html/data
	            - ./app/themes:/var/www/html/themes
	            - /etc/localtime:/etc/localtime:ro
            environment:
	            - VIRTUAL_HOST=FIX_WITH_YOUR_DNS_ADDRESS
	            - LETSENCRYPT_HOST=FIX_WITH_YOUR_DNS_ADDRESS
	            - LETSENCRYPT_EMAIL=FIX_WITH_YOUR_EMAIL
            restart: unless-stopped

    volumes:
	    nextcloud:
	    db:

    networks:
	    nextcloud_network:
	```
	A breakdown of this file: 
		* Update the fields for passwords and DNS address and email with your desired information. 
		* Used to define all the Docker elements that will be used.
		* Creates shared docker volumes and docker network for all the containers. 
		* Starts an nginx-proxy container to handle all HTPP traffic using the letsencrypt certificate. 
		* Starts a letsencrypt container to create certificates for use with proxy container
		* Starts a database container to use with Nextcloud. Stores database in a docker volume (`db`).
		* Starts a Nextcloud container using all the above. 
		* All paths that begin with `./` are relative paths to the location of the `docker-compose.yml` file. So the Nextlcoud app, config, data, etc will be stored locally.  Paths that begin with `nextcloud` or `db` are stored in a docker volume. 

# Starting the containers

1. Open powershell
2. Run `docker-compose -f /path/to/docker-compose.yml up`
3. This will start all the containers. Wait about 3 min for everything to settle. The first time it starts up takes the longest. 
4. Open a browser to https://your-dns.duckdns.org
5. Go through first time setup for nextcloud:
		1. Create an admin user and password
		2. Select `mysql` as the database type
			1. Enter the username and password for the database user (not root user) defined in the `docker-compose.yml`
			2. Enter `db` as the database host (the last field in the prompt). 
6. Hit continue to complete setup. 

Don't go further! 

# Modify config
There's some config to change before you can start really using Next cloud. 

1. Stop all the containers by hitting `ctrl+c` in the powershell window that started the `docker-compose` command. 
2. Go to the directory containing the `docker-compose.yml` file. You'll see a bunch of directories were created by the containers starting up. 
3. Edit the `./app/config/config.php` file. 
	1. Under the line that starts with `datadirectory`add the line `'check_data_directory_permissions' => false,`
	2. This is because Windows permissions are not compatible with Nextcloud when using Docker and WSL 2. See the comment in the example config about this setting: https://github.com/nextcloud/server/blob/master/config/config.sample.php
4. Create a file named `uploadsize.conf` in the the `./proxy/conf.d` directory with the following contents:
    ```
    client_max_body_size 10G;
    proxy_request_buffering off;
    ```
    This allows `nginx` to accept file uploads from Nextcloud clients up to 10G (you can adjust this up or down if you'd like). The default for `nginx` is 1MB, so too small for most uses. 

# Using Nextcloud
Run `docker-compose -f /path/to/docker-compose.yml up`

Nextcloud should now startup, an you can create normal users and being using as desired. 
Data uploaded by users will be saved in the `./app/data` directory. You should backup the entire directory container the `docker-compose.yml` file to ensure all config and user data is saved. 

The Docker shared volumes for `db` and `nextcloud` do contain some important data, too. Those are stored within the WSL 2 environment. 

```
PS C:\Users\alex> docker volume list
DRIVER    VOLUME NAME
local     nextcloud_db
local     nextcloud_nextcloud
```
These are accessible from the Windows host at this path: 
```
\\wsl$\docker-desktop-data\version-pack-data\community\docker\volumes
```
This path maps to a virtual disk stored at: 
```
C:\Users\username\AppData\Local\Docker\wsl\data\ext4.vhdx
```
So, if you're not already backing up your Windows boot drive and user folder, you should!

# VirtualBox Note
Docker Desktop does require Hyper-V to run with WSL2. It is possible to get this working with Virtual Box. 
https://www.how2shout.com/how-to/use-virtualbox-and-hyper-v-together-on-windows-10.html
