# DeployTemplateVPS

This is a template for deploying your web application on HTTPS to a VPS with CI/CD. It uses Docker, Docker Compose, and GitHub Actions.

## Prerequisites

### VPS

You need a VPS with Docker and Docker Compose installed. As of today, here is how you can install them on ubuntu:

```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/v2.6.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

To handle SSL certificates, we use Certbot. For the first time, you need to generate a certificate. It will be renewed automatically. 

To validate a certificate, Certbot needs to access a specific route on your server. We will need a very simple nginx web server to serve this route. Here is how to create it:

```bash
sudo mkdir -p /var/www/certbot/DeployTemplateVps

sudo apt update
sudo apt install -y nginx

sudo nano /etc/nginx/sites-available/default
```

On the file opened by nano, replace the content with the following:

```nginx
server {
   listen 80;
   server_name test.amercier.fr; # Replace with your domain

   location /.well-known/acme-challenge/ {
      root /var/www/certbot/DeployTemplateVps;
   }
}
```

Then, restart nginx:

```bash
sudo systemctl restart nginx
```

Finally, you can generate the certificate with Certbot and then uninstall nginx on the VPS:

```bash
sudo docker run -it --rm \
   -v /etc/letsencrypt:/etc/letsencrypt \
   -v /var/www/certbot/DeployTemplateVps:/var/www/certbot/DeployTemplateVps \
   certbot/certbot certonly --webroot -w /var/www/certbot/DeployTemplateVps -d test.amercier.fr

sudo apt remove --purge nginx
```

It will then we renewed automatically by the certbot container.

Lastly, you need to generate a private key for the Github Actions to access your VPS.

### Docker Hub

You need a Docker Hub account to store your Docker images. Create a new repository for your project.

You also need to create a personal access token to allow GitHub Actions to push images to your repository. As of today, you can create a new token on your "Docker Hub Account Setting" > "Security" > "Personal Access Token".

If you want to use a private repository, you need to login to Docker Hub on your VPS:

```bash
sudo docker login --username=yourhubusername
```

It will ask for a password, you can use the access token you just created.

### GitHub

Clone this repository and create a new one from it. Then, go to the settings of your repository, and in the "Secrets" section, add the following secrets:

- `VPS_HOST`: The IP address of your VPS
- `VPS_USERNAME`: The username to connect to your VPS
- `VPS_PRIVATE_KEY`: The private key to connect to your VPS
- `VPS_PORT`: The SSH port of your VPS
- `DOCKER_HUB_USERNAME`: Your Docker Hub username
- `DOCKER_HUB_ACCESS_TOKEN`: Your Docker Hub access token

You also need a `production` branch. This branch will be used to deploy your application. The `main` branch is used for development. When you want to deploy your application, merge the `main` branch into the `production` branch.

### Overwrite the template

You need to replace all occurences of `test.amercier.fr` with your domain name.

You need to replace all occurences of `deploy-template-vps` with your Docker Hub repository name.

If your VPS is not an Ubuntu server, you may need to adjust this template (for example in the `ci-cd.yml` the `/home/ubuntu/deploy-template-vps` path).

## Deploy

To deploy your application, merge the `main` branch into the `production` branch. The GitHub Actions will build your Docker image, push it to Docker Hub, and deploy it to your VPS.
