# EJBCA-docker-compose-NGINX-TLS

[LAB DURATION = 1h]

Steps to deploy EJBCA first without proxy, then deploy the proxy and EJBCA behind it in production like and on premise.

When i started my journey to deploy EJBCA CE in a production like environement, I wanted to use proxy directly to use it as fronted for EJBCA.
What I did not understand was that we need first to start EJBCA container once and generate the Management CA and the SuperAdmin.p12. And only after that we can setup nginx as frontend. 

With the SuperAdmin.p12, you can create the ROOT CA and SUB CA and then sign a Web TLS Certificate and and the .pem cert and .pem key to NGINX.

# **TL;DR**

1. [Prerequisites](https://github.com/s0p4L1N/ejbca-docker-nginx/blob/main/README.md#action-to-do-before-running-anything)
2. [Run EJBCA without Nginx](https://github.com/s0p4L1N/ejbca-docker-nginx/blob/main/README.md#initial-configuration---ejbca-without-proxy)
3. [Creation of ROOT, SUB CA and TLS Profile and Certs](https://github.com/s0p4L1N/ejbca-docker-nginx/blob/main/README.md#creation-of-root-sub-ca-and-tls-profile-and-certs)
4. [Run EJBCA with NGINX](https://github.com/s0p4L1N/ejbca-docker-nginx/blob/main/README.md#add-nginx-ejbca-conf)
5. Done




## Action to do before running anything

Prerequites for RHEL 9/Rocky Linux 9/Alma Linux 9

- Create user for docker

a. Create user and change is home folder (it can be any folder)
```
adduser -m -d /pki pki
chown -R pki:pki /pki
passwd pki
echo 'pki    ALL=(ALL)       ALL' | sudo EDITOR='tee -a' visudo
su - pki
sudo usermod -aG docker pki
```

b. Download project

```
git clone https://github.com/s0p4L1n3/EJBCA-docker-compose-NGINX-TLS.git
```

d. Create tree folder project and go inside
```
mkdir -p ejbca-standalone/{data,nginx/{certs,conf}}
cd ejbca-standalone
```

Tree view of docker project:

```
├── data
├── docker-compose.yml
├── nginx
│──────── ├── certs
│──────── └── conf
```

- **data** will be the volume for the database, even if the container is deleted, the data will remain
- **docker-compose.yml** is the main configuration file for this project
- **nginx** is the folder that will contains on part 2 nginx configuration for EJBCA and the certificates
  

## Initial configuration - EJBCA without proxy
To start with EJBCA and docker compose, I've followed the official documentation: https://doc.primekey.com/ejbca/tutorials-and-guides/tutorial-start-out-with-ejbca-docker-container
**I did some tweaks compare to the tutorial, such as additionnal environment key, and TLS_SETUP_ENABLED with value true instead of simple**

Download the docker-compose file [EJBCA Docker Compose Standalone:](https://github.com/s0p4L1n3/EJBCA-docker-compose-NGINX-TLS/blob/main/ejbca-standalone/docker-compose.yml)

- Run the docker compose command
```
docker compose up -d
```

## Second step -  Enrolment
- Check the logs after it is started

```
docker compose logs -f
```

At the end of the deployment, you will see some indications on how to get the SuperAdmin.p12 certificate.

<details>

<summary> Logs showing username and password for SuperAdmin.p12 </summary>

```
Health check now reports application status at /ejbca/publicweb/healthcheck/ejbcahealth
ejbca            *********************************************************************************************
ejbca            *                                                                                           *
ejbca            * A fresh installation was detected and a ManagementCA was created for your initial         *
ejbca            * administration of the system.                                                             *
ejbca            *                                                                                           *
ejbca            * Initial SuperAdmin client certificate enrollment URL (adapt port to your mapping):        *
ejbca            *                                                                                           *
ejbca            *   URL:      https://ejbca-node1:443/ejbca/ra/enrollwithusername.xhtml?username=superadmin *
ejbca            *   Password: Mft/2RdSOdf8nmaQAp9dRJBr                                                      *
ejbca            *                                                                                           *
ejbca            * Once the P12 is downloaded, use "Mft/2RdSOdf8nmaQAp9dRJBr" to import it.                  *
ejbca            *                                                                                           *
ejbca            *********************************************************************************************

```

</details>


#### STEP 1

On your host container, check the ip address with `ip address` and on you computer with the web browser access to the URL: https://your_guest_hosting_docker_ip/ejbca/ra/enrollwithusername.xhtml?username=superadmin


Enter the username and password showed in the previous logs, in my example user: superadmin and password: Mft/2RdSOdf8nmaQAp9dRJBr

<img width="844" alt="image" src="https://github.com/s0p4L1n3/EJBCA-docker-compose-NGINX-TLS/assets/126569468/3396a1af-380f-49a7-b621-012f539268d5">

<img width="952" alt="image" src="https://github.com/s0p4L1n3/EJBCA-docker-compose-NGINX-TLS/assets/126569468/707a77f4-0607-4d56-90cb-9bbba3024d32">

#### STEP 2

- Choose the key Specification according to the ManagementCA wich is create with RSA 4096.

Download the PKCS Certificate.

#### STEP 3

Import the certificate with the password showed in the previous logs.

<img width="501" alt="image" src="https://github.com/s0p4L1n3/EJBCA-docker-compose-NGINX-TLS/assets/126569468/000fe93d-80f5-40c8-b175-ee4b71a063b0">


#### STEP 4
And now you access again the PKI UI, a pop up displays, accept the risk

For Admin task: https://your_ip/ejbca/adminweb/
For RA Task: https://your_ip/ejbca/ra/

<details>
<summary> Click to view image of enrolment </summary>
  
  ![image](https://github.com/s0p4L1N/ejbca-docker-nginx/assets/92848369/6a943093-7655-4fe8-858e-8740889769e2)
  
</details>

#### STEP 5
And you have now access to the adminweb thanks to the SuperAdmin certificate.
<details>
<summary> Click to view image of enrolment </summary>
  
  ![image](https://github.com/s0p4L1N/ejbca-docker-nginx/assets/92848369/b2e67666-530e-4d10-a992-556c7f0125e9)

</details>

See next step of PKI Creation.

# Creation of ROOT CA, SUB CA and TLS Profile Cert

Before continuing with NGINX as frontend for EJBCA, you will need to create your PKI Hierarchy.

Download the templates I created and [Import them](https://github.com/s0p4L1n3/EJBCA-docker-compose-NGINX-TLS/tree/main/EJBCA-Profiles-Exported)

- CA Function
<img width="1088" alt="image" src="https://github.com/s0p4L1n3/EJBCA-docker-compose-NGINX-TLS/assets/126569468/077bae1a-9b71-4772-9ba2-1ed5d4415d4e">

- RA Function
<img width="713" alt="image" src="https://github.com/s0p4L1n3/EJBCA-docker-compose-NGINX-TLS/assets/126569468/fdfdbc5e-1d36-41b4-81ca-f7f20eaa665e">

With RA Cert profile, you get this message:

`Non existent CAs (with id 1277826676) removed from End Entity Profile TLS Server Profile.`

As you do not have the CAs created yet, the imported profile for server certificate does not know any CA.
You will need to edit it and change managementCA to the SUB CA you will create (see below for example)

<img width="373" alt="image" src="https://github.com/s0p4L1n3/EJBCA-docker-compose-NGINX-TLS/assets/126569468/2c4eac85-4733-4593-bbc6-1df4c40e2ff1">

Follow then [this guide](https://github.com/s0p4L1n3/EJBCA-docker-compose-NGINX-TLS/blob/main/tutorial-create-a-pki-hierarchy-in-ejbca.md) to create the ROOT CA and SUB CA based on the imported certificate profiles.

Create also the certificate on [Steps 3:](https://github.com/s0p4L1n3/EJBCA-docker-compose-NGINX-TLS/blob/main/tutorial-issue-tls-server-certificates-with-ejbca.md) 

# NGINX as Frontend

Now, you will deploy NGINX to use it as fronted for EJBCA.

1. Split the **PEM file** in two seperate file, one containing the **private key**, the other one containing the **certificates**.

- Recognize the difference
  - The private Key start by -----BEGIN PRIVATE KEY----- and end with -----END PRIVATE KEY-----
  - The certificates start by -----BEGIN CERTIFICATE----- and end with -----END CERTIFICATE-----

- The certificate file should look like below:
<details>
<summary> management_cert.pem </summary>

``` 
-----BEGIN CERTIFICATE-----
MIICijCCAjGgAwIBAgIUM3QOuvRix1FSWj2nLF6hHmSHURYwCgYIKoZIzj0EAwQw
NTEMMAoGA1UEBhMDVVNBMQwwCgYDVQQKDANVRk8xFzAVBgNVBAMMDkNvbXBhbnkg
U1VCIENBMB4XDTIzMDYyMjA5MTM1M1oXDTI0MDYyMTA5MTM1MlowMzEMMAoGA1UE
BhMDVVNBMQwwCgYDVQQKDANVRk8xFTATBgNVBAMMDE1hbmFnZW1lbnRDQTBZMBMG
ByqGSM49AgEGCCqGSM49AwEHA0IABIVV76Z0itpxB3ZLXYtT0Uz5XEXttpJTJbr/
53dLP2or9ehthtv002Pi9WbqQCbuuBy3w1mkQacUe06L0aQUTBGjggEfMIIBGzAf
BgNVHSMEGDAWgBQ4rMurXaeeeedegNVHQ8BAf8EBAMCBaAwCgYIKoZI
zj0EAwQDRwAwRAIgeKUK+Qxz9d7CIH2zDK8s9eLoRGk3LXKxy2+zgkXueAICIAxC
gqD8gu9cBRE5tSjcYCK9Zn6we56iMVtjDKst1Y5y
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
MIICcjCCAhegAwIBAgIUM9x9Da0t7LyjyUzQ26gfpS3Im1kwCgYIKoZIzj0EAwQw
NjEMMAoGA1UEBhMDVVNBMQwwCgYDVQQKDANVRk8xGDAWBgNVBAMMD0NvbXBhbnkg
Uk9PVCBDQTAeFw0yMzA2MjExNDU5MDNaFw0zODA2MTcxNDU5MDJaMDUxDDAKBgNV
BAYTA1VTQTEMMAoGA1UECgwDVUZPMRcwFQYDVQQDDA5Db21wYW55IFNVQiBDQTBZ
MBMGByqGSM49AgEGCCqGSM49AwEHA0IABA1tyGpFMf6fNOXc3U87YIJtI3tL9ZA1
F5IABq7dRv4a8l58ZQwQLD9hD3AwvI+rBt7p8HGpGpH1RRVkMM3rI6ajggECMIH/
MBIGA1UdEwEB/wQIMAYBAf8CAQAwHdedededeBzAChiNodHRwOi8vcGtpLmlz
cy5sYW4vY2VydHMvUk9PVENBLmNydDAjBggrBgEFBQcwAYYXaHR0cDovL3BraS5p
c3MubGFuL29jc3AwMwYDVR0fBCwwKjAooCagJIYiaHR0cDovL3BraS5pc3MubGFu
L2NybHMvUk9PVENBLmNybDAdBgNVHQ4EFgQUOKzLq12rcbr9EGq5AZ3knDovCDAw
DgYDVR0PAQH/BAQDAgGGMAoGCCqGSM49BAMEA0kAMEYCIQCNfsG1naKRNAF3GFnr
5rlj8QN2yM+up91P5RQHJI4W2wIhAKk8r+Fj2zXytjaSGp48S0vlm8qlaR2aaJLd
Awg8VWHu
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
MIIB0zCCAXmgAwIBAgIUYBdw1t4l6/o7Q/a46ktDeYkYc+AwCgYIKoZIzj0EAwQw
NjEMMAoGA1UEBhMDVVNBMQwwCgYDVQQKDANVRk8xGDAWBgNVBAMMD0NvbXBhbnkg
Uk9PVCBDQTAgFw0yMzA2MjExNDUxNDBaGA8yMDUzMDYxMzE0NTEzOVowNjEMMAoG
A1UEBhMDVVNBMQwwCgYDVQQKDANVRdeedeedeP4vZhTG0SswcMLILySu68vSHvpW
aM2Ss6zPcW45pCehMrEh3o9khZm5dcywr6NwJXCcUw8mx4p+GGl2cucKUNKjYzBh
MA8GA1UdEwEB/wQFMAMBAf8wHwYDVR0jBBgwFoAUQfL+EFkiBb53zzGlPwjWT6QW
+rMwHQYDVR0OBBYEFEHy/hBZIgW+d88xpT8I1k+kFvqzMA4GA1UdDwEB/wQEAwIB
hjAKBggqhkjOPQQDBANIADBFAiBwS0KvQ90cYqAhbgWQxkLhvD+Wa1r4kDCy6v+J
gTZukQIhANzUUTrUobkbrm2WtQl7ckrTF+p5QqhwDa1wfClyRDrU
-----END CERTIFICATE-----

```
  
</details>

- The private key file should look like below:
<details>

<summary> management_pvkey.pem </summary>

```
-----BEGIN PRIVATE KEY-----
MIGTAgEAMBMGByqGeefefeSM49AwEHBHkwdwIBAQQgG0zrNE8M2tLk0fVJ
bqdeZILP58OWi6reT4ePhpN9Ri9+igCgYIKoZgrgehRANCAASFVe+mdIracQd2
S12LU9FM+VxF7baSUyW6/+d3dvdSz9qK/XobYbb9NNj4vdcVm6kAm7rgct8NZpEGnFHtO
i9GkFEwR
-----END PRIVATE KEY-----
```
  
</details>

- You will also need to paste the Management CA (internal CA) for Certificate Authentication, so NGINX will forward the certificate to the internal Apache server of EJBCA.

![Download_Management_CA](https://github.com/s0p4L1N/ejbca-docker-nginx/assets/92848369/277c19ba-9c5a-45e5-82b7-d97704ec291c)

Now copy paste these three files into you nginx/certs folder on your host where the docker projet is. (`/opt/docker/ejbca/nginx/certs`)

- CA_Management.pem --> the internal 'ManagementCA' for Client Certificate Authentication verification
- management_cert.pem --> the TLS Server Certificate for the PKI itself (Nginx frontend)
- management_pvkey.pem --> the TLS Server private key

## Add NGINX ejbca conf

1. In `/opt/docker/ejbca/nginx/conf` paste downloaded file ejbca.conf [file and copy paste the below configuration

[Configuration file for EJBCA NGINX](https://github.com/s0p4L1n3/EJBCA-docker-compose-NGINX-TLS/blob/main/ejbca-nginx-TLS/ejbca.conf)


- And the full docker compose file projet

[Download the docker-compose for EJBCA with NGINX](https://github.com/s0p4L1n3/EJBCA-docker-compose-NGINX-TLS/blob/main/ejbca-nginx-TLS/docker-compose.yml)

What changes between old docker-compose and the new one for NGINX

- For the ejbca-node1 service:
  - addedd PROXY_HTTP_BIND=0.0.0.0 to the environment block to inform ejbca that there is a proxy as frontend
  - removed the ports block as it should not open the port to the outside world
 
- Added NGINX service code block
- added nginx conf to proxypass to the container
- Added a network for NGINX

  Now, without removing the containers, just execute `docker compose up -d` it will create the nginx container and recreate ejbca container according to the new configuration.

```
[+] Running 8/8
 ✔ nginx 7 layers [⣿⣿⣿⣿⣿⣿⣿]      0B/0B      Pulled                                                                                                                                                      14.1s
   ✔ 5b5fe70539cd Pull complete                                                                                                                                                                          8.8s
   ✔ 441a1b465367 Pull complete                                                                                                                                                                         11.5s
   ✔ 3b9543f2b500 Pull complete                                                                                                                                                                         11.5s
   ✔ ca89ed5461a9 Pull complete                                                                                                                                                                         11.5s
   ✔ b0e1283145af Pull complete                                                                                                                                                                         11.6s
   ✔ 4b98867cde79 Pull complete                                                                                                                                                                         11.6s
   ✔ 4a85ce26214d Pull complete                                                                                                                                                                         11.6s
[+] Building 0.0s (0/0)
[+] Running 4/4
 ✔ Network ejbca_proxy-bridge  Created                                                                                                                                                                   0.5s
 ✔ Container ejbca-database    Running                                                                                                                                                                   0.0s
 ✔ Container ejbca             Started                                                                                                                                                                  18.1s
 ✔ Container ejbca-proxy       Started 
 ```



## Issues during my testings:

- [EJBCA CE docker compose behind proxy, ManagementCA not created and no adminweb access](https://github.com/Keyfactor/ejbca-ce/discussions/302)
- [SuperAdmin can not match rules anymore](https://github.com/Keyfactor/ejbca-ce/discussions/305)




