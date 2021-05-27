# greenlight-keycloak
Deploying greenlight and keycloak over docker

## Prerequisites
 - Install docker and docker-compose
 - Make sure you have your own DNS and a public domain name
 
The DNS should look something like this:

```
<JOHN>.blindside-dev.com.        A       <YOUR_IP_ADDRESS>.            60
bbb.<JOHN>.blindside-dev.com.    A       <YOUR_BBB_IP_ADDRESS>.        60
gl.<JOHN>.blindside-dev.com.     CNAME   <JOHN>.blindside-dev.com.     60
*.gl.<JOHN>.blindside-dev.com.   CNAME   <JOHN>.blindside-dev.com.     60
kc.<JOHN>.blindside-dev.com.     CNAME   <JOHN>.blindside-dev.com.     60
```

## Installation

Clone this repository

```
git clone git@github.com:billzzhang/greenlight-keycloak.git
cd greenlight-keycloak
```

Copy the dotenv at the root of the project:
```
cp dotenv .env
```

Edit the .env file located in the root of the project
```
vi .env
```

Copy the dotenv in the greenlight folder:
```
cp greenlight/dotenv greenlight/.env
mkdir greenlight/app
```

Ex: Configuration .env
```
## Set the hostname using your own domain (Required)
NGINX_HOSTNAME=bill.blindside-dev.com

## Use only with postgres instance outside the one pre-packaged with docker-compose (Optional)
## DATABASE_URL=postgres://myuser:mypass@localhost

## Use with a repo other than the default "bigbluebutton" (Optional)
## DOCKER_REPO=my-repo
## Use with a tag other than the default "latest" (Optional)
## DOCKER_TAG=my-tag

## Site Template to be used by the nginx as a proxy (Optional)
SITES_TEMPLATE=docker
```

Create your own SSL Letsencrypt certificates. As you are normally going to have this deployment running on your own computer (or in a private VM), you need to generate the SSL certificates with certbot by adding the challenge to your DNS.

Install letsencrypt in your own computer
```
sudo apt-get update
sudo apt-get -y install letsencrypt
```

Make yourself root
```
sudo -i
```

Start creating the certificates
```
certbot certonly --manual -d gl.<JOHN>.blindside-dev.com -d *.gl.<JOHN>.blindside-dev.com --agree-tos --no-bootstrap --manual-public-ip-logging-ok --preferred-challenges=dns --email hostmaster@blindside-dev.com --server https://acme-v02.api.letsencrypt.org/directory
```

You will see something like this

```
-server https://acme-v02.api.letsencrypt.org/directory
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator manual, Installer None
Obtaining a new certificate
Performing the following challenges:
dns-01 challenge for gl.<JOHN>.blindside-dev.com
dns-01 challenge for gl.<JOHN>.blindside-dev.com

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please deploy a DNS TXT record under the name
_acme-challenge.gl.<JOHN>.blindside-dev.com with the following value:

2dxWYkcETHnimmQmCL0MCbhneRNxMEMo9yjk6P_17kE

Before continuing, verify the record is deployed.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Press Enter to Continue
```

Create a TXT record in your DNS for _acme-challenge.gl.<JOHN>.blindside-dev.com with the challenge string as its value 2dxWYkcETHnimmQmCL0MCbhneRNxMEMo9yjk6P_17kE and add another challenge string under the same record

As this certificate requires the root and a wildcard, it will require two challenges on the same request.

Repeat the Letsencrypt procedure for 
```
-d kc.<JOHN>.blindside-dev.com
```

Copy the certificates to your greenlight-keycloak directory. Although /etc/letsencrypt/live/ holds the latest certificate, they are only symbolic links. The real files must be copied and renamed
First create an empty directory:
```
mkdir -p proxy/letsencrypt/live
```
```
cp -R /etc/letsencrypt/archive/gl.<JOHN>.blindside-dev.com <YOUR ROOT>/greenlight-keycloak/proxy/letsencrypt/live
cp -R /etc/letsencrypt/archive/kc.<JOHN>.blindside-dev.com <YOUR ROOT>/greenlight-keycloak/proxy/letsencrypt/live
```
```
mv <YOUR ROOT>/greenlight-keycloak/proxy/letsencrypt/live/gl.<JOHN>.blindside-dev.com/cert1.pem <YOUR ROOT>/greenlight-keycloak/proxy/letsencrypt/live/gl.<JOHN>.blindside-dev.com/cert.pem
mv <YOUR ROOT>/greenlight-keycloak/proxy/letsencrypt/live/gl.<JOHN>.blindside-dev.com/chain1.pem <YOUR ROOT>/greenlight-keycloak/proxy/letsencrypt/live/gl.<JOHN>.blindside-dev.com/chain.pem
mv <YOUR ROOT>/greenlight-keycloak/proxy/letsencrypt/live/gl.<JOHN>.blindside-dev.com/fullchain1.pem <YOUR ROOT>/greenlight-keycloak/proxy/letsencrypt/live/gl.<JOHN>.blindside-dev.com/fullchain.pem
mv <YOUR ROOT>/greenlight-keycloak/proxy/letsencrypt/live/gl.<JOHN>.blindside-dev.com/privkey1.pem <YOUR ROOT>/greenlight-keycloak/proxy/letsencrypt/live/gl.<JOHN>.blindside-dev.com/cert.pem
```
And do the same with `kc`

And finally, start your environment with docker-compose

```
cd <YOUR ROOT>/greenlight-multitenant
docker-compose up
```

If everything goes well, you will see all the containers starting and at the end you will have access to greenlight through (or something similar):
```
https://gl.<JOHN>.blindside-dev.com/
```
and you will be able to access keycloak through (something similar):
By default, the admin console can be accessed with username: `admin` password: `Password!`
```
https://kc.<JOHN>.blindside-dev.com/
```

## Setup
### Using keycloak with greenlight using OIDC

Log into Keycloak 

(Optional) Create a realm for greenlight by hovering "master" dropdown and clicking new realm:
![image](https://user-images.githubusercontent.com/21375588/119866764-29a1e580-beeb-11eb-8a18-39370c39a5fb.png)

In realm settings > Login, enable user registration and remember to click save.\
In clients, click create and give an appropriate client id (Ex: `greenlight`)\
Set the access type to confidential, and fill out redirect url and web origins.
Ex configuration:
![image](https://user-images.githubusercontent.com/21375588/119872156-2873b700-bef1-11eb-91f5-d1bc291a0466.png)
Copy `Client ID` in the admin console on the same page and paste it into `greenlight/.env` under `OPENID_CONNECT_CLIENT_ID`\
Under the same page/client, click on the credentials tab and copy and paste it into `greenlight/.env` under `OPENID_CONNECT_CLIENT_SECRET`
 
Go back to the Realm settings on the sidebar and click on the link `OpenID Endpoint Configuration` where you will find the issuer.\
Copy the `issuer` and paste it into `greenlight/.env` under `OPENID_CONNECT_ISSUER`\
In addition, fill out the `OAUTH2_REDIRECT` with the appropriate URL.
Ex sample:
```
OPENID_CONNECT_CLIENT_ID=greenlight
OPENID_CONNECT_CLIENT_SECRET=efee718a-9aec-4e08-a1b0-99a96ef70936
OPENID_CONNECT_ISSUER=https://kc.test.bill.blindside-dev.com/auth/realms/greenlight
OPENID_CONNECT_HD=
OPENID_CONNECT_UID_FIELD=

# OAUTH2_REDIRECT allows you to specify the redirect_url passed to oauth on sign in.
# It is useful for cases when Greenlight is deployed behind a Network Load Balancer or proxy
OAUTH2_REDIRECT=https://gl.test.bill.blindside-dev.com/
```
 
Finally, restart the deployment and signing up using OIDC should be possible
 
