# ReactJS-Flask-app-VPS-deploy
## 0. Prerequisite 

You should have 2 projects:
- front-project
- back-project

We also assume that your have created a ssh key to connect to your github repo from your VPS.

### Back
- contains requirements.txt
- Endpoints start with `/api`
```python
# Example
@app.route('/api/my-route', methods=['GET'])
```
- You have a start.py file
```python
from app import app

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=5000)
```

### Front

This tuto has been tested with ReactJs + Typescript + Vite. You may need to do some adjustements if your stack is different.
- Your are using env file to store your base url

`.env` file in your local repository:
```
VITE_API_URL=http://localhost:5000/api
```

Example of use
```javascript
const response = await fetch
     `${import.meta.env.VITE_API_URL}/my-route}`,
);
```

## 1. Prepare your VPS

Connect to the VPS
```bash
ssh your_username@your_vps_ip
```

Install Node and Python if necessary
```bash
sudo apt update
sudo apt upgrade -y
sudo apt install git python3 python3-pip nodejs npm -y
```
## 2. Clone your projects

```bash
git clone https://github.com/your_username/front-project.git
git clone https://github.com/your_username/back-project.git
```
## 3. Configure your Flask app
### Create a virtual environment for your project

Import Virtual env package.
```bash
cd back-project
sudo apt install python3-venv
```

Create a virtualenv called venv.
```bash
sudo apt install python3-venv
python3 -m venv venv
source venv/bin/activate
```

Install your dependencies.
```bash
pip install -r requirements.txt
deactivate
```

Back to your root.
```
cd ..
```
### Create your environment variables
Create `.env` file.
```bash
touch .env
```
Add necessary variables to `.env`.
```bash
echo "FLASK_APP=app.py" >> .env
echo "FLASK_ENV=production" >> .env
```
## 3. Configure your React App

Install your dependencies
```bash
cd front-project
npm i
```

Create a `.env` file.
```bash
touch .env
```

If your don't have domain name:
```bash
echo "VITE_API_URL=http://your_vps_ip:5000/api" >> .env
```

Else (domain.com for this example):
```bash
echo "VITE_API_URL=https://subdomain.domain.com/api" >> .env
```
Back to the root.
```bash
cd ..
```

## 3. Configure background deployment with pm2

Install `pm2`.
```bash
sudo npm install -g pm2
```

You can create a `deploy.sh` file to launch every command at once. It could be useful if you want to add github action for automatic deployment.

```bash
sudo nano deploy.sh
```
```bash
# deploy the backend side
cd back-project
source venv/bin/activate
pm2 start python3 --name "movie-app-back" start_prod.py

# deploy the frontend side
cd ../front-project
pm2 start npm --name "movie-app-front" -- run dev -- --host 0.0.0.0 --port 5173

# Save the pm2 processes to start on reboot
pm2 save
pm2 startup
```

## 4. (If you have a domain) Access the application from your domain

### Subdomain creation

Go to your DNS control panel

Create a new subdomain:
- Type: A
- Name: subdomain name you want
- Value: your_vps_ip
- TTL: default or 3600

### Nginx installation and configuration

Install Nginx.
```bash
sudo apt update
sudo apt install nginx -y
```

Create a new conf file for your subdomain.
```bash
sudo nano /etc/nginx/sites-available/subdomain.domaine.com
```

Add the following content to the file.
```nginx
server {
    listen 80;
    server_name subdomain.domaine.com;

    location / {
        proxy_pass http://localhost:5173;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /api {
        proxy_pass http://localhost:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Create a symbolic link in `site-enabled`.
```bash
sudo ln -s /etc/nginx/sites-available/subdomain.domaine.com /etc/nginx/sites-enabled/
```

Test the new config.
```bash
sudo nginx -t
```

Restart to apply modifications
```bash
sudo systemctl restart nginx
```

### SSL configuration

Install certbot.
```bash
sudo apt install certbot python3-certbot-nginx -y
```

Obtain and install a SSL certificat for your subdomain.
```bash
sudo certbot --nginx -d subdomain.domaine.com
```

Follow the instructions. Certbot will automatically configure Nginx to use SSL. You can verify in the conf file.
```bash
sudo nano /etc/nginx/sites-available/subdomain.domaine.com
```

Restart to apply modifications.
```bash
sudo systemctl restart nginx
```

## Deploy

Just launch your `deploy.sh` file
```
bash deploy.sh
```

## More

- Use github action to automatocally update your app
- Use Gunicorn for Flask deployment
