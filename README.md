<h1>Project Name</h1>
Setup digital ocean droplet for next.js application



<h2>Project Description</h2>
THis project represent the setup that you need to perform once before you want to use next.js app on digital ocean droplet

<h2>Motivation</h2>
You have a next.js application and you want to host it on digital ocean droplet - VPS. You want to use https and a domain. What setup is required on the droplt to achive this ?

<h2>Installation</h2>
Perform the following in order for one per droplet setup

<h3>make pm2 global (one point of truth)</h3>

1. Install pm2 Globally as Root
Ensure Node.js and npm are installed:

```bash
node -v
npm -v
```

Install pm2 globally:

```bash
sudo npm install -g pm2
```

Verify installation:

```bash
pm2 -v
```

2. Ensure Non-Root User Can Access pm2
Log in as the Non-Root User:

If you're currently logged in as root, switch to the non-root user:

```bash
su - <non-root-username>
```

Add /usr/local/bin (or wherever global npm binaries are installed , verify with 'npm config get prefix') to the non-root user's PATH:

```bash
echo 'export PATH=/usr/local/bin:$PATH' >> ~/.bashrc
source ~/.bashrc # reload changes immediately
```

Confirm the non-root user can invoke pm2:

```bash
su - <non-root-username>
pm2 -v
```

3. Grant Write Access to PM2 Files
Change ownership of pm2 directories to the non-root user:

```bash
sudo chown -R <non-root-username>:<non-root-username> /home/<non-root-username>/.pm2
```

Verify that the .pm2 directory for the user exists:

```bash
ls -l /home/<non-root-username>/.pm2
```

4. Test PM2 Commands as Non-Root User

```bash
su - <non-root-username>
```

Start an application (e.g., app.js):

```bash
pm2 start app.js --name "my-app"
pm2 list
```
you canalso use shell command e.g. ls instead of app.js

5. Configure CI/CD Pipeline to Use Non-Root User

Update your CI/CD pipeline's ssh commands to switch to the non-root user. For example:

```bash
ssh non-root-user@VPS_IP "pm2 restart my-app"
```

6. Set Up Autostart for PM2 (Optional for Production), need to be invoked once
While logged in as the non-root user, configure autostart for the system:

```bash
pm2 startup
```

Follow the generated instructions, which may include running a sudo command to finalize the setup.


Once done you can check the status as follows
```bash
sudo systemctl status pm2-<non-root-username>
```


7. Save PM2 Processes (Optional)
Save the current list of processes so they can be restored after a reboot:

```bash
pm2 save
```

<h3>Configure Nginx</h3>
<ul>
  <li>
    <strong>Create the file in your GitHub repository:</strong> Store the <code>my-app.conf</code> file in your project's <code>config/nginx</code> directory.
    <pre>
      <code>
server {
    listen 80;

    location / {
        proxy_pass http://localhost:3000; # Adjust port if needed
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
      </code>
    </pre>
  </li>
  <li>
    <strong>Move and link the file via SSH:</strong> In your GitHub Actions workflow, after the code is checked out, use <code>ssh</code> to execute the following commands on your DigitalOcean droplet:
    <pre>
      <code>
sudo mv ./config/nginx/my-app.conf /etc/nginx/sites-available/my-app.conf
sudo ln -sf /etc/nginx/sites-available/my-app.conf /etc/nginx/sites-enabled/my-app.conf
      </code>
    </pre>
    <p>Note: <code>./config/nginx/my-app.conf</code> refers to the file in the <code>config/nginx</code> directory of your cloned repository.</p>
  </li>
  <li>
    <strong>Reload Nginx:</strong> Immediately after the move and link commands, use <code>ssh</code> to reload the Nginx configuration:
    <pre>
      <code>
sudo systemctl reload nginx
      </code>
    </pre>
  </li>
</ul>

using this file you can now access the next.js app via the browser using the droplet ip and port 3000

<h3>setup domain in digital ocean droplet</h3>
<ol>
<li>purchase a domain</li>
<li>add the domain to digital ocean .navigate to mangae->networking->domains 
Enter domain - posttoyoutube.xyx, choose project - post 2 youtube and click 'Add domain' 
<img src='./figs/add-domain-to-do-droplet.png'/>
</li>
<li>create droplet :
hostname : @
will direct to : add here your doplet
click on the button create record

as shown in the following image
<img src='./figs/create-record.png'/>

create also record for hostname www with the same 'will direct to'

the resulted created records appear in the follwoing image where the @ record appears in brown and www record appears in blue 
<img src='./figs/created-records.png'/>
</li>
</ol>

<h3>setup domain in namecheap</h3>
Here we will tell namecheap about digital ocean
from the dashboard choose the domain post2youtube.xyz and click Manage
scroll down and for Nameservers choose "custom DNS' and enter what was written in digitl ocean : ns1.digitalocean.com. ns2.digitalocean.com. ns3.digitalocean.com. as follows

<img  src='./figs/nameservers-on-namechaep.png'/>

This might take time to tkae effect

if you try to access it immidiately you might not be able to see the page

<img src='./figs/page-can-not-be-displayed.png'/>


but after few minutes you will getthe default nginx page but with the correct domain post2youtube.xyz 

<img src='./figs/with-domain-show-default-nginx-page.png'/>

you can can access the next.js app using the domain but still need the 3000 port 

<img src='./figs/use-domain-but-still-need-port-for-next.png'/>


<h3>now i want to access next.js app without port</h3>
add     server_name post2youtube.xyz www.post2youtube.xyz; under server {
    listen 80; in config/nginx/my-app.conf

    ```bash
  sudo nginx -t # test configuration
  sudo systemctl reload nginx # reload Nginx to apply changes:
    ```
---------->it is not working because the default nginx get in the way so i reomve the symbolic link 

```bash
sudo rm /etc/nginx/sites-enabled/default

```
but it still exist in /etc/nginx/sites-available/ (yet not active because link removed from sites-enabled)

after this

```bash
sudo nginx -t  # Test the configuration for syntax errors
sudo systemctl reload nginx
```

now access http://post2youtube.xyz will access the next.js app without need for the port but still the connection is not secured because http is used - not https

<h3>now i want to access next.js app without 'Not Secure' -> need https and certificate</h3>
    <strong>1. Install Certbot</strong>
    <ul>
        <li>Update the package list:
            <pre><code>sudo apt update</code></pre>
        </li>
        <li>Install Certbot and the Nginx plugin:
            <pre><code>sudo apt install certbot python3-certbot-nginx</code></pre>
        </li>
    </ul>

  <strong>2. Obtain an SSL Certificate</strong>
    <ul>
        <li>Run Certbot to get a certificate and configure Nginx:
            <pre><code>sudo certbot --nginx -d post2youtube.xyz -d www.post2youtube.xyz</code></pre>
        </li>
        <li>Certbot will automatically configure SSL and set up redirection.</li>
    </ul>

  <strong>3. Verify Certificate Renewal</strong>
    <ul>
        <li>Test automatic renewal:
            <pre><code>sudo certbot renew --dry-run</code></pre>
        </li>
    </ul>

  <strong>4. Review Nginx Configuration</strong>
    <p>The Nginx configuration file will look like this:</p>
    <pre><code>
server {
    listen 80;
    server_name post2youtube.xyz www.post2youtube.xyz;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name post2youtube.xyz www.post2youtube.xyz;

    ssl_certificate /etc/letsencrypt/live/post2youtube.xyz/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/post2youtube.xyz/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
    </code></pre>

  <strong>5. Restart Nginx</strong>
    <ul>
        <li>Test and reload Nginx:
            <pre><code>
sudo nginx -t
sudo systemctl reload nginx
            </code></pre>
        </li>
    </ul>

  <strong>6. Verify HTTPS</strong>
    <ul>
        <li>Visit: <a href="https://post2youtube.xyz" target="_blank">https://post2youtube.xyz</a></li>
        <li>Ensure your site is accessible over HTTPS.</li>
    </ul>





<h2>Usage</h2>
....



<h2>Technologies Used</h2>
<ul>
<li>digital ocean droplet</li>
<li>nginx</li>
<li>ubuntu</li>
<li>pm2</li>
<li>https</li>
<li>namecheap</li>
<li>next.js</li>
</ul>





<h2>Demo</h2>
....

<h2>Points of Interest</h2>
<ul>
    <li>altough the subject of this repo is 'Setup digital ocean droplet for next.js application' it can be used for deploying and node based web site</li>
   
</ul>



