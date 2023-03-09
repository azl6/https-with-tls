# Questions

1. How widely is CloudFlare used?
2. Does the usage of SSL certificates imply on the usage of some sort of reverse-proxy?
3. Snake-oil certificates (self-signed) are not for public access, since the browser won't trust it. However, it means that we MUST use a reverse-proxy with it, right?
4. How do I configure NGINX to point to a server instead of pure HTML pages?
5. Does Lets Encrypt work similarly as free signed certificates, where we provide a CSR and get a signed certificate?
6. How does certificates work when our reverse-proxy isn't NGINX, like an AWS LB? Do we store the certificates in either IAM certificate store or ACM and point to these certificates?

# Symmetric and asymmetric algorithms

AES: Simetric, like AES256 used for SSE-S3. **Same key to encrypt and decrypt**.

RSA: Asymmetric, like SSH keys generated by `ssg-keygen -t rsa`. **Different keys to encrypt and decrypt**.

# Encryption algorithms

![image](https://user-images.githubusercontent.com/80921933/221655495-b6c42783-cc87-4d36-b963-f4abee26c20f.png)

# Hashing algorithms

![image](https://user-images.githubusercontent.com/80921933/221645440-40d1c495-c327-4a01-9cff-3541512cce80.png)

# Using OpenSSL

The following command generates a RSA key

```bash
openssl genrsa
```

To encrypt it, we can use the flag `-aes256`

```bash
openssl genrsa -aes256
```

To save the private-key to a file, we use the `-out` flag

```bash
openssl genrsa -aes256 -out myoutfile.txt
```

When we generate a RSA key with openssl, the public-key comes "mixed" with it. To extract the public-key, we use the following command:

```bash
openssl rsa -in <RSAOUTFILEFROMLASTCOMMAND> -outform PEM -pubout -out <PUBLICKEYNAME>
```

We can also generate different sizes of keys, like

```bash
openssl genrsa 4096
```

# How does the browser trust certain certificates

The OS comes "packed" with the root certificates installed. 

That means, if the certificate provided by the website matches the OS-downloaded root-certificate, a HTTPS connection will be successfully established

This is possible because intermediate-certificates are signed by the root-certificates, and the intermediate-certificates sign the end-user certificate

![Screenshot from 2023-02-27 17-06-08](https://user-images.githubusercontent.com/80921933/221672411-8c601bea-5835-42f3-ad64-564edfc6e039.png)

![sgds](https://user-images.githubusercontent.com/80921933/221672955-fd9f4d73-134b-4aa4-97f1-1b58272ea315.png)

# Certificates trust chain 

![image](https://user-images.githubusercontent.com/80921933/221681490-9b1f5dd0-be16-420b-893f-97b5ebd7ec05.png)

# Configuring NGINX on Ubuntu

Let's follow this tutorial: https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-20-04

Explanation of some aspects of the configuration:

This configuration consists in creating the `/var/www/your_domain/html/index.html` structure, changing `your_domain` for our created domain, aka alexthedeveloper.com.br. The `index.html` file at the end of the address contains the content that we want to serve.

After that, we create the `your_domain` file (aka alexthedeveloper.com.br) in the following folder: `/etc/nginx/sites-available/your_domain`. This is where we will describe our routing strategies for Nginx to follow:

```bash
server {
        listen 80; ########## Listening port for IPV4
        listen [::]:80; ##### Listening port for IPV6

        root /var/www/alexthedeveloper.com.br/html; ### Place the client will be redirected to when entering the address in the server_name field
        index index.html index.htm index.nginx-debian.html;

        server_name alexthedeveloper.com.br www.alexthedeveloper.com.br; ### Adresses that will redirect to the root file

        location / {
                try_files $uri $uri/ =404;
        }
}
```

After that, we just normally follow the tutorial.

# Installing certificates issued by CloudFlare on NGINX 

If we're using CloudFlare, we can go to the **SSL/TLS > Origin Server > Create Certificate** option

![image](https://user-images.githubusercontent.com/80921933/223791933-17389882-8052-40d2-ae69-311f90754d33.png)

As the options, we can leave:

- RSA (2048)
- The default domains (*.domain.com, domain.com)
- 15 years 

As a result, the screen will show the generated **Origin Certificate** and **Private-Key**

![image](https://user-images.githubusercontent.com/80921933/223792847-6972c94d-5dd0-45bb-9661-ab6ebdaca8f3.png)

The end of the screen has a tutorial to install the certificates on different reverse-proxies

![image](https://user-images.githubusercontent.com/80921933/223793462-c82cbd95-0f44-4a05-b006-34a80a9dac8b.png)

After clicking on it, we can choose the `NGINX` option

![image](https://user-images.githubusercontent.com/80921933/223793635-12990242-274c-4f00-a7e5-9546d315826d.png)

Since we already have the private-key and certificate generated by CloudFlare, we can jump to step **II.4 - Edit the Nginx virtual hosts file**

**REFERENCE 1** In a nutshell, this step simply tells us to create the `/etc/ssl/your_domain_name.pem` and `/etc/ssl/your_domain_name.key` files, and add the following lines to our `/etc/nginx/sites-available/your_domain` file:

```
server {
        # ...
        listen   443;

        ssl    on;
        ssl_certificate    /etc/ssl/your_domain_name.pem; (or bundle.crt)
        ssl_certificate_key    /etc/ssl/your_domain_name.key;
        # ...

}
```

# Generating a self-signed certificate and using it on NGINX

**PRE-REQUISITES:** For this to work, we need NGINX running on the server and DNS records pointing to it.

Let's generate the **key** and **certificate** (in the /etc/ssl/ folder, since it's a good practice) with the following command:

Source for command below: https://stackoverflow.com/questions/10175812/how-to-generate-a-self-signed-ssl-certificate-using-openssl

Consider using the `-nodes` flag to skip the password. If we don't use this flag, we will be prompted to provide a password, and will need to configure NGINX to use a password file in the future (also explained in this tutorial).

(Change \<MYDOMAIN> for your domain, like **alexthedeveloper.com.br**. I'm still not sure if this is a required step. I think not.)

```bash
openssl req -x509 -newkey rsa:4096 -keyout <MYDOMAIN>.pem -out <MYDOMAIN>.pem -sha256 -days 365
```

We will be prompted to insert the data of the certificate. The most relevant part is the "Common name" prompt, where we need to insert our domain-name:

![image](https://user-images.githubusercontent.com/80921933/223854078-df30eae5-ca8d-4623-8693-738ee8a73a40.png)

After that, we simply need to point our `/etc/nginx/sites-available/\<your_domain>` server block to our newly-created certificates, with similar steps to **REFERENCE 1** in the above step.

After configuring the block, we need to restart nginx (`sudo service nginx stop` and `sudo service nginx start`). If we run into problems, check the following:

1. If the error says something about "more arguments being passed", make sure thatthe added lines end with `;`
2. If the error says the certificate couldn't be loaded, that's because we provided a passphrase when creating the certificate, and the NGINX configuration file needs to know that passphrase. We can choose simply to not encrypt the private-key and ignore this error message, passing the `-nodes` flag to the openssl command. However, if you still want to fix that, we must update the `/etc/nginx/nginx.conf` OR the `/etc/nginx/sites-available/\<your_domain>` (recommended) file with the following tutorial: https://stackoverflow.com/questions/33084347/pass-cert-password-to-nginx-with-https-site-during-restart. Important to notice that the passphrase file must have the chosen password inside for the command to work.

After restarting NGINX and accessing our DNS, we'll see the following page:

![image](https://user-images.githubusercontent.com/80921933/223860846-e9b21b6f-001b-4c86-a47f-b7e97a832fec.png)

This happens because our certificate is self-signed, and therefore, it's not trusted by the browser.

To get a certificate that is trusted by the browsers, we need to generate a CSR (Certificate Sign Request) and get a certificate from a free SSL certificate issuer (there are many in the internet, Bogdan used the following on class 102: ssl.com)

# Generating a CSR to get a free signed SSL certificate

Use the command below:

```bash
openssl req -new -newkey rsa:4096 -nodes -keyout <KEYNAME>.key -out <CSRNAME>.csr
```

This will generate a Certificate Sign Request (CSR) that can be used to retrieve a signed SSL certificate from a free SSL certificate issuer.

This certificate is mostly used to be directly faced by customers, since self-signed certificates aren't made to be customer-facing since the browser won't trust them.

For a free SSL certificate issuer website, I recommend **ssl.com**, or to follow Bogdan's class 102 or following a 3rd party tutorial.

# Very important problem after getting a signed certificate

We can use the following website to check our certificate: https://www.sslshopper.com/ssl-checker.html#hostname=

If, at the end of the check, we see the following error:

![image](https://user-images.githubusercontent.com/80921933/223880930-d738dc38-46bb-400c-bbeb-34e9ac2ac5f6.png)

It means that we only provided the main certificate.

To fix that, we must get the **Intermediate Certificate** (usually comes together with the email when we require it)

![image](https://user-images.githubusercontent.com/80921933/223881268-3a7f7121-1c99-45e5-a226-bc7aa3c75992.png)

Then, we convert it to the **.txt** format (to make it "readable") and concatenate it at the `.pem` certificate file, like that:

![Screenshot from 2023-03-08 21-08-47](https://user-images.githubusercontent.com/80921933/223881596-4b395c4d-2262-440b-9e77-14a3025ff47d.png)

# Configuring redirects from HTTP to HTTPS in NGINX

We can add the following line to our `server` block in the `sudo nano /etc/nginx/sites-available/\<your_domain>` file

```
server {
        # ...
        return 301 https://example.com$request_uri;
}
```

The `request_uri` means that any path passed will also be redirected to HTTPS, like: http://example.com/aboutme -> https://example.com/aboutme 

# Generating a certificate with Certbot and Lets Encrypt

Follow this video's tutorial: https://www.youtube.com/watch?v=5wzs-pcDQ3k
