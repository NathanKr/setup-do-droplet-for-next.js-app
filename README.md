<h1>Project Name</h1>
Next.js Droplet Setup with Nginx and HTTPS

<h2>Project Description</h2>
This step-by-step tutorial details the process of setting up a DigitalOcean droplet to host a Next.js application, with instructions for configuring Nginx and enabling HTTPS.


<h2>Motivation</h2>
You have a Next.js application and want to deploy it on a DigitalOcean droplet (VPS) with HTTPS and a custom domain. What steps are needed to configure the droplet for this? This tutorial provides the answers.


<h2>Prerequisites</h2>
<ul>
  <li>A DigitalOcean account and an active droplet.</li>
  <li>A registered domain name.</li>
  <li>Basic proficiency with the Linux command-line interface.</li>
  <li>A functional Next.js application.</li>
  <li>System user permissions:
    <ul>
      <li>A non-root user with sudo privileges (recommended for enhanced security).</li>
      <li>The root user (use with caution).</li>
    </ul>
  </li>
</ul>

<h2>Installation</h2>
Execute these steps in sequence to configure your DigitalOcean droplet for initial use.

<h3>Install Node.js and npm</h3>
<p>Node.js and npm are essential for running Next.js applications. Node.js is the runtime environment, and npm is the package manager.</p>
<ol>
  <li><strong>Update your package index:</strong>
    <p>This ensures you have the latest package information.</p>
    <pre><code>sudo apt update</code></pre>
  </li>
  <li><strong>Install Node.js and npm:</strong>
    <pre><code>sudo apt install nodejs npm</code></pre>
  </li>
  <li><strong>Verify the installation:</strong>
    <p>Confirm that Node.js and npm are installed correctly.</p>
    <ul>
      <li>Check the Node.js version:
        <pre><code>node -v</code></pre>
      </li>
      <li>Check the npm version:
        <pre><code>npm -v</code></pre>
      </li>
    </ul>
  </li>
</ol>


<h3>Make pm2 global (one point of truth)</h3>

<ol>
<li><strong>Install pm2 Globally as Root</strong>

Install pm2 globally:

```bash
sudo npm install -g pm2
```

Verify installation:

```bash
pm2 -v
```
</li>
<li><strong>Ensure Non-Root User Can Access pm2</strong>
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
</li>
<li><strong>Grant Write Access to PM2 Files</strong>
Change ownership of pm2 directories to the non-root user:

```bash
sudo chown -R <non-root-username>:<non-root-username> /home/<non-root-username>/.pm2
```

Verify that the .pm2 directory for the user exists:

```bash
ls -l /home/<non-root-username>/.pm2
```
</li>
<li><strong>Test PM2 Commands as Non-Root User</strong>

```bash
su - <non-root-username>
```

Start an application (e.g., app.js):

```bash
pm2 start app.js --name "my-app"
pm2 list
```
You can also use shell command e.g. ls instead of app.js
</li>
<li><strong>Configure CI/CD Pipeline to Use Non-Root User</strong>

Update your CI/CD pipeline's ssh commands to switch to the non-root user. For example:

```bash
ssh non-root-user@VPS_IP "pm2 restart my-app"
```
</li>
<li><strong>Set Up Autostart for PM2 (Optional for Production)</strong> 
need to be invoked once
While logged in as the non-root user, configure autostart for the system:

```bash
pm2 startup
```

Follow the generated instructions, which may include running a sudo command to finalize the setup.


Once done you can check the status as follows
```bash
sudo systemctl status pm2-<non-root-username>
```
</li>

<li><strong>Save PM2 Processes (Optional)</strong>
Save the current list of processes so they can be restored after a reboot:

```bash
pm2 save
```
</li>
</ol>


<h3>Install Nginx</h3>
<ol>
  <li><code>sudo apt update</code> - Update your package index</li>
  <li><code>sudo apt install nginx</code> - Install Nginx</li>
  <li><code>sudo ufw allow 'Nginx Full'</code> - Open both ports 80 (HTTP) and 443 (HTTPS)</li>
  <li><code>sudo systemctl start nginx</code> - Start Nginx</li>
  <li><code>sudo systemctl enable nginx</code> - Enable Nginx to start on boot</li>
  <li><code>sudo systemctl status nginx</code> - Verify Installation</li>
</ol>

The configuration for this default behavior is usually stored in the /etc/nginx/sites-available/default file, and it’s symlinked to /etc/nginx/sites-enabled/default. This file appears in this repo : ./config/nginx/default

<h3>Configure Nginx</h3>
<ol>
  <li>
    <strong>Create the file in your GitHub repository:</strong> Store the <code>my-app.conf</code> file in your project's <code>config/nginx</code> directory. This configuration file sets up a reverse proxy for your Next.js app

```nginx
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
```
  </li>
  <li>
    <strong>Move and link the file</strong> In your GitHub Actions workflow, after the code is checked out,  execute the following commands on your DigitalOcean droplet:
    <pre>
      <code>
sudo mv ./config/nginx/my-app.conf /etc/nginx/sites-available/my-app.conf
sudo ln -sf /etc/nginx/sites-available/my-app.conf /etc/nginx/sites-enabled/my-app.conf
      </code>
    </pre>
    <p>Note: <code>./config/nginx/my-app.conf</code> refers to the file in the <code>config/nginx</code> directory of your cloned repository.</p>
  </li>
  <li>
    <strong>Reload Nginx:</strong> Immediately after the move and link commands, reload the Nginx configuration:
    <pre>
      <code>
sudo systemctl reload nginx
      </code>
    </pre>
  </li>
</ol>

Using this file you can now access the next.js app via the browser using the droplet ip and port 3000

<h3>Configure Domain on DigitalOcean</h3>

<p>To access your Next.js application using a domain name instead of the droplet's IP address and port, you need to configure your domain's DNS settings. This involves adding the domain to your DigitalOcean account and creating DNS records that point to your droplet.</p>

<ol>
  <li><strong>Purchase a domain:</strong> Obtain a domain name from a registrar such as Namecheap.</li>
  <li><strong>Add the domain to DigitalOcean:</strong>
    <ul>
      <li>Navigate to "Networking" -> "Domains" in your DigitalOcean control panel.</li>
      <li>Enter your domain (e.g., posttoyoutube.xyz), select your project, and click "Add domain."
      <img src="./figs/add-domain-to-do-droplet.png" alt="Add domain to DigitalOcean"></li>
    </ul>
  </li>
  <li><strong>Create DNS records:</strong>
    <ul>
      <li>Create an "A" record with:
        <ul>
          <li><strong>Hostname:</strong> @</li>
          <li><strong>Value:</strong> Your droplet's IP address</li>
        </ul>
        Click "Create Record."
      <img src="./figs/create-record.png" alt="Create DNS record in DigitalOcean"></li>
      <li>Create another "A" record with:
        <ul>
          <li><strong>Hostname:</strong> www</li>
          <li><strong>Value:</strong> Your droplet's IP address</li>
        </ul>
      </li>
      <li>The resulting records will appear in your DigitalOcean DNS settings. The "@" record is shown in brown, and the "www" record in blue
      <img src="./figs/created-records.png" alt="Created DNS records in DigitalOcean"></li>
    </ul>
  </li>
</ol>

<h3>Configure Domain Nameservers on Namecheap</h3>

<p>To point your domain to your DigitalOcean droplet, you need to update your domain's nameservers at your domain registrar (Namecheap, in this example). This tells Namecheap to use DigitalOcean's DNS servers for your domain.</p>

<ol>
  <li><strong>Access Domain Management:</strong>
    <ul>
      <li>Log in to your Namecheap account.</li>
      <li>From the dashboard, select your domain (e.g., post2youtube.xyz) and click "Manage."</li>
    </ul>
  </li>
  <li><strong>Update Nameservers:</strong>
    <ul>
      <li>Scroll down to the "NAMESERVERS" section.</li>
      <li>Select "Custom DNS."</li>
      <li>Enter the DigitalOcean nameservers provided in the previous step:
        <ul>
          <li>ns1.digitalocean.com</li>
          <li>ns2.digitalocean.com</li>
          <li>ns3.digitalocean.com</li>
        </ul>
      <img src="./figs/nameservers-on-namechaep.png" alt="Namecheap nameserver configuration"></li>
    </ul>
  </li>
  <li><strong>Propagation Time:</strong>
    <p>DNS changes can take some time to propagate across the internet. It might take a few minutes or up to 48 hours for the changes to take full effect.</p>
  </li>
  <li><strong>Initial Access:</strong>
    <ul>
      <li>If you try to access your domain immediately after changing nameservers, you might see a "page cannot be displayed" error.<img src="./figs/page-can-not-be-displayed.png" alt="Page cannot be displayed error"></li>
    </ul>
  </li>
  <li><strong>Successful Propagation:</strong>
    <ul>
      <li>After the DNS changes propagate, you should see the default Nginx welcome page when you visit your domain (e.g., post2youtube.xyz).<img src="./figs/with-domain-show-default-nginx-page.png" alt="Nginx default page with domain"></li>
    </ul>
  </li>
  <li><strong>Accessing Next.js Application:</strong>
    <ul>
      <li>At this point, you can access your Next.js application using your domain, but you will still need to specify the port (e.g., post2youtube.xyz:3000).<img src="./figs/use-domain-but-still-need-port-for-next.png" alt="Next.js app access with port"></li>
    </ul>
  </li>
</ol>

<h3>Access Next.js App Without Port</h3>

<p>To access your Next.js application without specifying the port (e.g., post2youtube.xyz instead of post2youtube.xyz:3000), you need to update the Nginx configuration. This involves modifying the <code>server_name</code> directive in your <code>my-app.conf</code> file and removing the default Nginx configuration.</p>

<ol>
  <li><strong>Update Nginx Configuration:</strong>
    <ul>
      <li>Open your <code>my-app.conf</code> file and update the <code>server_name</code> directive:

```nginx
      server {
          listen 80;
          server_name post2youtube.xyz www.post2youtube.xyz;
          # ... other configurations ...
      }
```
  </li>

  </ul>
  </li>
  <li><strong>Test Nginx Configuration:</strong>
    <ul>
      <li>Test the Nginx configuration for syntax errors:</li>

  ```bash
  sudo nginx -t
  ```

  </ul>
  </li>
  <li><strong>Remove Default Nginx Configuration:</strong>
    <ul>
      <li>The default Nginx configuration file (<code>/etc/nginx/sites-enabled/default</code>) may interfere with your custom configuration. Remove the symbolic link:

  ```bash
  sudo rm /etc/nginx/sites-enabled/default
  ```
  </li>

  <li>Note that the file still exists in <code>/etc/nginx/sites-available/</code>, but it is no longer active.</li>
    </ul>
  </li>
    <li><strong>Reload Nginx:</strong>
    <ul>
      <li>Reload Nginx to apply the changes:

  ```bash
  sudo systemctl reload nginx
  ```
  </li>
    </ul>
  </li>
  <li><strong>Verify Access:</strong>
    <ul>
      <li>You should now be able to access your Next.js application using your domain name without the port (e.g., http://post2youtube.xyz).</li>
      <li>However, the connection is still using HTTP, which is not secure.</li>
    </ul>
  </li>
</ol>


<h3>Use https and certificate</h3>
<p>Now i want to access next.js app without 'Not Secure'</p>
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

```nginx
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
```

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


<h2>Technologies Used</h2>
<ul>
  <li>DigitalOcean Droplet</li>
  <li>Nginx (1.26.0)</li>
  <li>Ubuntu (24.10)</li>
  <li>Node.js (v20.16.0)</li>
  <li>PM2 (5.4.3)</li>
  <li>Let's Encrypt (Certbot)</li>
  <li>Namecheap</li>
  <li>Next.js (15.2.1)</li>
</ul>


<h2>Points of Interest</h2>
<ul>
  <li>Although this tutorial focuses on setting up a DigitalOcean droplet for a Next.js application, the principles and configurations can be applied to deploy any Node.js-based website or application.</li>
</ul>

<h2>References</h2>
<ul>
  <li>The Next.js application used in this tutorial, including its CI/CD workflow, can be found in the following repository: <a href="https://github.com/NathanKr/simple-ci-cd-pipeline-for-next.js-app">https://github.com/NathanKr/simple-ci-cd-pipeline-for-next.js-app</a></li>
</ul>

