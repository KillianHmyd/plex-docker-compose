# Docker-Compose for Plex Server

This `docker-compose.yml` allows to quickly deploy an exposed Plex with tools to automatically downloads movies and tv shows via Torrent and store them on an encrypted Google Drive.

## Requirements

You need to install (`Docker`)[https://docs.docker.com/get-docker/] and (`Docker-Compose`)[https://docs.docker.com/compose/install/]

You also will need to expose your ports `80` and `443` to be able to communicate with your Plex from the outside 👍.

You will also need to (install rclone)[https://rclone.org/install/] (This is only needed to configure rclone, you will be able to delete it after).

## Setup

### Traefik Proxy
Firstly you'll need to configure (`Traefik`)[https://docs.traefik.io/] to expose your services.
In the (`.env` file)[.env] setup the `DOMAIN` vairable with the domain under which the services will be exposed.

Change the `TRAEFIK_HTPASSWD` (by default set to root:root). To generate a new one you can use `htpasswd` :
```sh
$> htpasswd -n -b MY_USERNAME MY_PASSWORD
```

To let the TLC communication properly work you'll need to add your vertificate and key in [certs folder](configs/traefik/certs) under the name `domain.cert` and `domain.key`. `Traefik` also allows you to setup an ACME challenge with open-encrypt, you can do so by following [this tutorial](https://docs.traefik.io/https/acme/).

Now you can run `docker-compose up -d traefik` and you'll be able to see the GUI under `https://traefik.<YOUR_DOMAIN>` 👏

### RClone

RClone will allow you to mount your Google Drive on your server.

> Note that this step is completely facultative if you prefer to store your files on your disk. In that case you can just create a folder `/media` where you clone this repository (You'll also need to remove rclone from the docker-compose).

Firstly you'll need to generate a Google Drive API Client ID and Client Secret. To do so you can follow this [tutorial from Cloudbox](https://github.com/Cloudbox/Cloudbox/wiki/Google-Drive-API-Client-ID-and-Client-Secret). Note the keys, you won't be able to recover them.

Now run at the root of the repository the following command : 
```sh
$> rclone --config ./configs/rclone-encrypted/rclone.conf config
```

You will be prompt the following : 
```
Current remotes:

Name                 Type
====                 ====
gcache               cache
gcrypt               crypt
gdrive               drive

e) Edit existing remote
n) New remote
d) Delete remote
r) Rename remote
c) Copy remote
s) Set configuration password
q) Quit config
e/n/d/r/c/s/q> 
```

Select `e`, rclone should now ask you wich remote to edit, select `gdrive` (3).
Then select `y` to edit the `client_id` and enter the client_id that you stored earlier.
Do the same for the secret.
On the `Use auto config?` enter `n` and follow the instruction. ⚠️ If your browser display you a page with `This app isn't verified`, select `advanced` and then `Go to rclone (unsafe)`.
Copy the code and paste it in your terminal. 
Finish by entering `y` then `q`

Finally you'll need to update the [`rclone.conf`](./configs/rclone-encrypted/rclone.conf) file to enter your Plex Username and Password (line `plex_username` and `plex_password`).

You can also edit the `remote` lines to mep your mount volume to another Google Drive folder than `media` : 
⚠️Note that if you already have files in this folder, they won't be available since rclone will only process encrypted data !⚠️

Start rclone by running `docker-compose up -d rclone`, you can now see a folder `media` containing your drive data ! 🎉

### Transmission 

Transmission is a Torrent client, it will be used to automatically download your media.

You just need to setup your credentials, in [configs/transmission/settings.json](./configs/transmission/settings.json) file search for `rpc-username` and `rpc-password` and set your password/username you want to access the transmission GUI.

Start Transmission by running `docker-compose up -d rclone`, you can now access your transmission GUI under `https://transmission.<YOUR_DOMAIN>` 🙌

### Radarr, Syncarr & Radarr4K

Radarr allows you to automatically download movies. Radarr4K is just another instance of Radarr allowing you to downloads 4K movies, syncarr is here to sync both Radarr.

Using two Radarr is useful to avoid clients transcoding some 4K, which is a pretty heavy operation.
If you don't want to download 4K or don't mind transcoding 4K you can simply delete `radarr4K` and `syncar` from the `docker-compose.yml`.

Start by running the two Radarr : `docker-compose up -d radarr radarr4K`
Your Radarr are now up on `https://radarr.<YOUR_DOMAIN>` and `https://radarr4K.<YOUR_DOMAIN>`.
**You must set some credentials before !**. Go on `Settings` => `General` and select `Forms (Login Page)` on `Authentication` input. Setup your credentials and then click on save.

Setup Transmission connection by going on the `Download Client`, click on `+` and select `Transmission`.
As `Host` set `transmission` for the Username and Password field, enter the credentials you've setup on transmission. Do it for both Radarr.

In `Profile` tab create your favorite profiles (probably `4K` for Radarr4K and `1080p` for Radarr), note the setted name.  

Now return in `General` tab and copy the `API Key` of both Radarr. 

Edit the (`.env`)[./.env] file by setting `RADARR_FHD_KEY`, `RADARR_4K_KEY`, `RADARR_FHD_PROFILE` and `RADARR_4K_PROFILE`.

Finish by running `docker-compose up -d syncarr`, your Radarr are all up 🎊

### Sonarr

Sonarr allows you to automatically download your tv shows. 

Nothing much to do here, start sonarr by running `docker-compose up -d sonarr`. Now on `https://sonarr.<YOUR_DOMAIN>` you will be able to setup your password and profiles.

You'll also need to setup the Transmission connection the same way than on Radarr 👍

### Plex

The Rockstar of this docker-compose, Plex will allows you to stream all your media !

First you will need to find a Plex Token, you can follow [this tutorial](https://support.plex.tv/articles/204059436-finding-an-authentication-token-x-plex-token/). Copy your Claim and paste it in the [.env](./.env) file at the `PLEX_CLAIM` key.

Start your plex server with `docker-compose up -d plex` and you can now browse it on `https://plex.<YOUR_DOMAIN>` !
When adding a Library, always start from the `/media` path. It is where is mapped your Google Drive ! 💃

### Tautilli

Tautilli allows you to monitor your Plex.

Just start Tautilli by running `docker-compose up -d tautilli`. It is now up on `https://tautulli.<YOUR_DOMAIN>` !
You might need to configure the Plex connection by going on `Settings` > `PLEX MEDIA SERVER`. 😄

### Monitoring stack

All this services allows you to monitor your Docker Services and your host.

Start them all by running `docker-compose up -d grafana` and you can now monitor your services on `https://grafana.<YOUR_DOMAIN>` 🤓
