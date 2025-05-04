# From Serverless to AWS: Deploying Node.js Apps on EC2

Transitioning from serverless platforms like Vercel or Render to virtual machines (VMs) on AWS EC2 gives you greater control over your infrastructure, potentially reduces costs, and allows for custom configurations. This guide walks you through deploying a Node.js project (such as an Express or Next.js app) on an AWS EC2 instance, covering project structure, GitHub setup, domain and SSL configuration with Cloudflare, deployment strategies, process management with PM2, and securing the app with Nginx.

![AWS EC2 Deployment](https://raw.githubusercontent.com/XyonX/portfolio-content/refs/heads/main/Blogs/serverless-to-aws/images/awsec2.png)


## 1. Proper Project Structure

A well-organized project structure is crucial for maintainability and deployment. Below is a recommended structure for a Node.js project (e.g., an Express or Next.js app):

```
my-node-app/
├── src/                    # Source code
│   ├── routes/             # Express routes or Next.js pages
│   ├── middleware/         # Custom middleware
│   ├── controllers/        # Business logic
│   └── app.js              # Main Express app or Next.js entry point
├── public/                 # Static assets (images, CSS, etc.)
├── package.json            # Node.js dependencies and scripts
├── ecosystem.config.js     # PM2 configuration file
├── .gitignore              # Ignore node_modules, logs, etc.
├── README.md               # Project documentation
└── next.config.js          # (Optional) Next.js configuration
```

- **For Express**: Place your routes, middleware, and controllers in `src/`. The `app.js` file initializes the Express server.
- **For Next.js**: Use the `pages/` or `app/` directory for routing (depending on the Next.js version). Include `next.config.js` for custom configurations.
- **Key Files**:
  - `package.json`: Define dependencies and scripts like `"start": "node src/app.js"` for Express or `"start": "next start"` for Next.js.
  - `.gitignore`: Ignore `node_modules/`, `.env`, and build artifacts like `dist/` or `.next/`.

## 2. Create a GitHub Repository

Version control is essential for collaboration and deployment. Follow these steps to set up a GitHub repository:

1. **Initialize the Repository Locally**:
   ```bash
   git init
   git add .
   git commit -m "Initial commit"
   ```

2. **Create a Repository on GitHub**:
   - Go to [GitHub](https://github.com) and create a new repository (e.g., `my-node-app`).
   - Choose public or private based on your needs.
   - Do not initialize with a README or `.gitignore` if you already have them locally.

3. **Push to GitHub**:
   ```bash
   git remote add origin https://github.com/your-username/my-node-app.git
   git branch -M main
   git push -u origin main
   ```

This repository will serve as the source for deploying your code to the AWS VM.

## 3. Set Up a Custom Domain and SSL with Cloudflare

To make your app accessible via a custom domain (e.g., `example.com`) and secure it with SSL, use Cloudflare for DNS management and certificate generation. Cloudflare’s Origin CA certificate provides free SSL/TLS encryption between Cloudflare and your EC2 instance.[](https://www.digitalocean.com/community/tutorials/how-to-host-a-website-using-cloudflare-and-nginx-on-ubuntu-20-04)[](https://support.cloudways.com/en/articles/5130554-how-to-configure-cloudflare-origin-certificate)[](https://geekrewind.com/how-to-setup-cloudflare-origin-certificates-with-nginx-on-ubuntu-16-04-18-04/)

### Step 1: Register a Domain
- **Purchase a Domain**: Use a registrar like Cloudflare Registrar, Namecheap, or GoDaddy to buy a domain (e.g., `example.com`). Prices vary; less common TLDs like `.party` can cost under $5/year.[](https://medium.com/%40life-is-short-so-enjoy-it/homelab-nginx-proxy-manager-setup-ssl-certificate-with-domain-name-in-cloudflare-dns-732af64ddc0b)
- **Note**: If you already own a domain, skip to the next step.

### Step 2: Configure Cloudflare DNS
1. **Add Your Domain to Cloudflare**:
   - Sign up for a free Cloudflare account at [cloudflare.com](https://www.cloudflare.com).
   - Click “Add a Site” and enter your domain (e.g., `example.com`).
   - Select the free plan and confirm.
2. **Update Nameservers**:
   - Cloudflare will provide custom nameservers (e.g., `ns1.cloudflare.com`, `ns2.cloudflare.com`).
   - Log in to your domain registrar and update the nameservers to Cloudflare’s. This change may take up to 24 hours to propagate.
3. **Set Up DNS Records**:
   - In the Cloudflare dashboard, go to **DNS** > **Records**.
   - Add an **A record**:
     - **Name**: `@` (for the root domain, e.g., `example.com`) or a subdomain (e.g., `www`).
     - **IPv4 address**: Your EC2 instance’s public IP address.
     - **Proxy status**: Enable “Proxied” to use Cloudflare’s CDN and security features.
   - Add a **CNAME record** (optional) for `www`:
     - **Name**: `www`
     - **Target**: `example.com`
     - **Proxy status**: Proxied
   - Verify DNS propagation using a tool like `dig` or Cloudflare’s dashboard (status should show “Active”).

### Step 3: Generate a Cloudflare Origin CA Certificate
1. **Access the SSL/TLS Section**:
   - In the Cloudflare dashboard, select your domain and go to **SSL/TLS** > **Origin Server**.
2. **Create a Certificate**:
   - Click **Create Certificate**.
   - Choose **Generate private key and CSR with Cloudflare** (default).
   - **Hostnames**: Enter your domain (e.g., `example.com`) and subdomains (e.g., `*.example.com` for wildcards). Include the root domain and any subdomains you plan to use (e.g., `www.example.com`).
   - **Certificate Validity**: Select 15 years (default) or a shorter period for better security.[](https://developers.cloudflare.com/ssl/origin-configuration/origin-ca/)[](https://support.cloudways.com/en/articles/5130554-how-to-configure-cloudflare-origin-certificate)
   - Click **Create**.
3. **Save the Certificate and Key**:
   - Copy the **Origin Certificate** and **Private Key** displayed in the dashboard.
   - On your EC2 instance, create files to store them:
     ```bash
     sudo mkdir -p /etc/ssl/certs
     sudo mkdir -p /etc/ssl/private
     sudo nano /etc/ssl/certs/cloudflare_origin.pem
     ```
     Paste the Origin Certificate, save, and exit.
     ```bash
     sudo nano /etc/ssl/private/cloudflare_key.pem
     ```
     Paste the Private Key, save, and exit.
   - Set permissions for security:
     ```bash
     sudo chmod 644 /etc/ssl/certs/cloudflare_origin.pem
     sudo chmod 600 /etc/ssl/private/cloudflare_key.pem
     ```
   - **Warning**: Ensure no blank lines are in the certificate or key files, as Nginx treats them as invalid.[](https://www.digitalocean.com/community/tutorials/how-to-host-a-website-using-cloudflare-and-nginx-on-ubuntu-20-04)
4. **Set SSL/TLS Encryption Mode**:
   - In Cloudflare’s **SSL/TLS** > **Overview**, set the encryption mode to **Full (strict)** to ensure secure connections between Cloudflare and your EC2 instance.[](https://developers.cloudflare.com/ssl/origin-configuration/origin-ca/)

### Step 4: Verify Certificate Installation
- The certificate is now ready to be used by Nginx (configured in the next section). If you pause Cloudflare or disable proxying, the Origin CA certificate may cause untrusted certificate errors, as it’s only trusted by Cloudflare.[](https://www.digitalocean.com/community/tutorials/how-to-host-a-website-using-cloudflare-and-nginx-on-ubuntu-20-04)



![AWS EC2 Deployment](https://raw.githubusercontent.com/XyonX/portfolio-content/refs/heads/main/Blogs/serverless-to-aws/images/techstack.webp)

## 4. Deploying to the AWS VM

You can deploy your project to the AWS EC2 instance using either **CI/CD** (e.g., GitHub Actions) or **rsync** for manual file transfer. Below are both approaches, including how to store the `.pem` file and set permissions.

### Option 1: Set Up CI/CD with GitHub Actions

CI/CD automates building and deploying your app, similar to Vercel’s auto-build feature.

1. **Launch an EC2 Instance**:
   - In the AWS Console, create an EC2 instance (e.g., Ubuntu 22.04, t2.micro for testing).
   - Configure a security group to allow SSH (port 22), HTTP (port 80), and HTTPS (port 443).
   - Download the `.pem` key file (e.g., `your-key.pem`) for SSH access.

2. **Store and Secure the `.pem` File**:
   - **On Linux/Mac**:
     - Save the `.pem` file in a secure directory, e.g., `~/.ssh/`:
       ```bash
       mv ~/Downloads/your-key.pem ~/.ssh/your-key.pem
       ```
     - Set restrictive permissions to prevent unauthorized access:
       ```bash
       chmod 400 ~/.ssh/your-key.pem
       ```
     - This ensures only the file owner can read it.
   - **On Windows**:
     - Save the `.pem` file in a secure directory, e.g., `C:\Users\YourUser\.ssh\`.
     - Set permissions using `icacls` to restrict access:
       ```cmd
       icacls "C:\Users\YourUser\.ssh\your-key.pem" /inheritance:r
       icacls "C:\Users\YourUser\.ssh\your-key.pem" /grant:r "%username%:R"
       ```
     - Alternatively, use Windows Explorer: Right-click the file > Properties > Security > Edit, and allow only your user account read access.
     - If using Git Bash or WSL, you can also use `chmod 400` as in Linux.
   - **Security Note**: Never commit the `.pem` file to Git or share it publicly. Add it to `.gitignore` if stored in your project directory.

3. **Set Up the VM**:
   - SSH into the VM using the `.pem` file:
     ```bash
     ssh -i ~/.ssh/your-key.pem ubuntu@<vm-public-ip>
     ```
   - Install Node.js:
     ```bash
     sudo apt update
     curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
     sudo apt install -y nodejs
     ```
   - Install Git:
     ```bash
     sudo apt install -y git
     ```

4. **Create a GitHub Actions Workflow**:
   - In your repository, create a file: `.github/workflows/deploy.yml`.
   - Add the following configuration:

     ```yaml
     name: Deploy to AWS EC2
     on:
       push:
         branches:
           - main
     jobs:
       deploy:
         runs-on: ubuntu-latest
         steps:
           - name: Checkout code
             uses: actions/checkout@v3
           - name: Set up Node.js
             uses: actions/setup-node@v3
             with:
               node-version: '20'
           - name: Install dependencies
             run: npm install
           - name: Build (for Next.js)
             run: npm run build
             if: contains(github.event.head_commit.message, 'next')
           - name: Deploy to EC2
             env:
               EC2_HOST: ${{ secrets.EC2_HOST }}
               EC2_USER: ${{ secrets.EC2_USER }}
               EC2_KEY: ${{ secrets.EC2_KEY }}
             run: |
               echo "$EC2_KEY" > deploy_key.pem
               chmod 600 deploy_key.pem
               rsync -avz --delete -e "ssh -i deploy_key.pem -o StrictHostKeyChecking=no" ./ $EC2_USER@$EC2_HOST:/home/$EC2_USER/my-node-app
               ssh -i deploy_key.pem -o StrictHostKeyChecking=no $EC2_USER@$EC2_HOST << 'EOF'
                 cd /home/$EC2_USER/my-node-app
                 npm install
                 npm run build
                 pm2 restart ecosystem.config.js
               EOF
     ```

5. **Add Secrets to GitHub**:
   - Go to your repository’s Settings > Secrets and variables > Actions.
   - Add:
     - `EC2_HOST`: Your EC2 public IP.
     - `EC2_USER`: `ubuntu` (or your VM user).
     - `EC2_KEY`: The contents of your `.pem` key file (open it in a text editor and copy the entire text).

6. **Explanation**:
   - The workflow triggers on pushes to `main`.
   - It builds the app (e.g., `npm run build` for Next.js) and uses `rsync` to transfer files to the VM.
   - It installs dependencies and restarts the app using PM2.

### Option 2: Manual Deployment with rsync

For simpler setups, use `rsync` to transfer files manually.

1. **Store and Secure the `.pem` File** (same as above):
   - **Linux/Mac**: Save to `~/.ssh/your-key.pem` and run `chmod 400 ~/.ssh/your-key.pem`.
   - **Windows**: Save to `C:\Users\YourUser\.ssh\your-key.pem` and use `icacls` or Explorer to restrict access.

2. **Set Up the VM** (same as above).

3. **Transfer Files with rsync**:
   - From your local machine:
     ```bash
     rsync -avz -e "ssh -i ~/.ssh/your-key.pem" ./ ubuntu@<vm-public-ip>:/home/ubuntu/my-node-app
     ```
   - On Windows, if using Git Bash or WSL, the command is the same. If using PowerShell, ensure `rsync` is installed (e.g., via WSL or Cygwin).

4. **SSH and Run the App**:
   - SSH into the VM:
     ```bash
     ssh -i ~/.ssh/your-key.pem ubuntu@<vm-public-ip>
     ```
   - Navigate to the app directory:
     ```bash
     cd /home/ubuntu/my-node-app
     npm install
     npm run build  # For Next.js
     ```

## 5. Process Management with PM2

PM2 is a process manager that keeps your Node.js app running, restarts it on crashes, and supports clustering.

1. **Install PM2 Globally**:
   On the VM:
   ```bash
   sudo npm install -g pm2
   ```

2. **Create a PM2 Ecosystem File**:
   In your project root, create `ecosystem.config.js`:

   ```javascript
   module.exports = {
     apps: [
       {
         name: 'my-node-app',
         script: 'src/app.js', // For Express
         // script: 'node_modules/next/dist/bin/next', // For Next.js
         // args: 'start', // For Next.js
         instances: 'max', // Use all CPU cores
         exec_mode: 'cluster', // Cluster mode for scalability
         env: {
           NODE_ENV: 'production',
           PORT: 3000,
         },
       },
     ],
   };
   ```

   - **For Express**: Set `script` to your main file (e.g., `src/app.js`).
   - **For Next.js**: Uncomment the Next.js lines and set `script` to the Next.js binary.

3. **Start the App with PM2**:
   ```bash
   pm2 start ecosystem.config.js
   ```

4. **Save PM2 Configuration**:
   Ensure PM2 restarts on VM reboot:
   ```bash
   pm2 save
   pm2 startup
   ```

5. **Monitor the App**:
   Check logs or status:
   ```bash
   pm2 logs
   pm2 list
   ```

## 6. Secure and Serve with Nginx

Nginx acts as a reverse proxy, improves security, and simplifies access by routing HTTP requests to your Node.js app. Use the Cloudflare Origin CA certificate for SSL.

1. **Install Nginx**:
   On the VM:
   ```bash
   sudo apt update
   sudo apt install -y nginx
   ```

2. **Configure Nginx with Cloudflare SSL**:
   Create a configuration file:
   ```bash
   sudo nano /etc/nginx/sites-available/my-node-app
   ```
   Add the following, using the Cloudflare certificate and key:
   ```nginx
   server {
       listen 80;
       server_name example.com www.example.com; # Replace with your domain
       return 301 https://$host$request_uri; # Redirect HTTP to HTTPS
   }
   
   server {
       listen 443 ssl;
       server_name example.com www.example.com; # Replace with your domain
   
       ssl_certificate /etc/ssl/certs/cloudflare_origin.pem;
       ssl_certificate_key /etc/ssl/private/cloudflare_key.pem;
   
       ssl_protocols TLSv1.2 TLSv1.3;
       ssl_ciphers EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH;
       ssl_prefer_server_ciphers on;
   
       location / {
           proxy_pass http://localhost:3000; # Your Node.js app port
           proxy_http_version 1.1;
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection 'upgrade';
           proxy_set_header Host $host;
           proxy_cache_bypass $http_upgrade;
       }
   }
   ```
   - Replace `example.com www.example.com` with your domain.
   - The `ssl_certificate` and `ssl_certificate_key` paths point to the Cloudflare certificate files created earlier.[](https://www.digitalocean.com/community/tutorials/how-to-host-a-website-using-cloudflare-and-nginx-on-ubuntu-20-04)[](https://geekrewind.com/how-to-setup-cloudflare-origin-certificates-with-nginx-on-ubuntu-16-04-18-04/)
   - The HTTP block redirects all traffic to HTTPS.

3. **Enable the Configuration**:
   ```bash
   sudo ln -s /etc/nginx/sites-available/my-node-app /etc/nginx/sites-enabled/
   sudo nginx -t # Test configuration
   sudo systemctl restart nginx
   ```

4. **Verify SSL**:
   - Visit `https://example.com` in a browser. You should see a secure connection with no certificate errors.
   - If errors occur, ensure the certificate and key files have no blank lines and the Nginx configuration points to the correct paths.[](https://www.digitalocean.com/community/tutorials/how-to-host-a-website-using-cloudflare-and-nginx-on-ubuntu-20-04)

## 7. Automating Builds for Next.js (Vercel-Like Experience)

To replicate Vercel’s auto-build feature, integrate the build step into your deployment pipeline:

- **In CI/CD**: The GitHub Actions workflow above includes `npm run build` for Next.js projects. This runs automatically on each push.
- **Manual Deployment**: Run `npm run build` on the VM after transferring files with `rsync`.
- **Watch for Changes**: For development, you can use a tool like `nodemon` to watch for changes and rebuild:
  ```bash
  npm install -g nodemon
  nodemon --exec "npm run build && pm2 restart ecosystem.config.js" src/
  ```

## 8. Additional Tips

- **Environment Variables**: Store sensitive data (e.g., API keys) in a `.env` file and use `dotenv` in your app. Exclude `.env` from Git with `.gitignore`.
- **Backups**: Regularly back up your VM using AWS snapshots or export your app directory.
- **Scaling**: For high traffic, consider AWS Auto Scaling or load balancers.
- **Monitoring**: Use PM2’s monitoring tools (`pm2 monit`) or integrate with AWS CloudWatch.
- **Cloudflare Security**: Enable Cloudflare’s WAF (Web Application Firewall) and DDoS protection for added security.[](https://support.cloudways.com/en/articles/5130554-how-to-configure-cloudflare-origin-certificate)

## Conclusion

Moving from serverless to AWS EC2 gives you flexibility and control. By structuring your project, setting up a custom domain with Cloudflare, securing it with an Origin CA certificate, deploying with CI/CD or `rsync`, managing processes with PM2, and configuring Nginx with SSL, you can achieve a robust, secure deployment. For Next.js projects, automate builds in your pipeline to mimic Vercel’s simplicity. With this setup, your Node.js app will be production-ready on AWS, accessible via your custom domain with end-to-end encryption.

For further customization, explore AWS services like RDS for databases or CloudFront for CDN capabilities, and leverage Cloudflare’s performance features like CDN and caching. Happy hosting!
