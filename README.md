# Deploying a Flask Application with Nginx and Gunicorn

This guide provides a step-by-step process for deploying a Flask application using Gunicorn as the WSGI server and Nginx as the reverse proxy on an Ubuntu server.

## Prerequisites

- Ubuntu server
- Basic knowledge of terminal commands
- A Flask application ready for deployment

## 1. Create a New User

1. **Add a new user:**

    ```bash
    sudo adduser hussain
    ```

2. **Grant sudo privileges to the user:**

    ```bash
    sudo usermod -aG sudo hussain
    ```

3. **Switch to the new user:**

    ```bash
    su - hussain
    ```

## 2. Install Required Packages

1. **Update package lists:**

    ```bash
    sudo apt update
    ```

2. **Install Python packages and other dependencies:**

    ```bash
    sudo apt install python3-pip python3-dev build-essential libssl-dev libffi-dev python3-setuptools python3-venv ufw
    ```

## 3. Set Up the Flask Application

1. **Create a project directory:**

    ```bash
    mkdir /var/www/moviesburn.com
    cd /var/www/moviesburn.com
    ```

2. **Set up a virtual environment:**

    ```bash
    python3 -m venv .movies-env
    source .movies-env/bin/activate
    ```

3. **Install Flask and Gunicorn:**

    ```bash
    pip install wheel gunicorn flask
    ```

4. **Create your Flask application (`main.py`):**

    ```python
    from flask import Flask
    app = Flask(__name__)

    @app.route("/")
    def hello():
        return "<h1 style='color:blue'>Hello There!</h1>"

    if __name__ == "__main__":
        app.run(host='0.0.0.0')
    ```

5. **Test the Flask application:**

    ```bash
    python main.py
    ```

    Access your application at `http://your_ip:5000`.

## 4. Set Up Gunicorn

1. **Create a WSGI file (`wsgi.py`):**

    ```python
    from main import app

    if __name__ == "__main__":
        app.run()
    ```

2. **Run Gunicorn:**

    For port forwarding:
    ```bash
    gunicorn --bind 192.168.1.5:80 wsgi:app
    ```

    For a default setup:
    ```bash
    gunicorn --bind 0.0.0.0:5000 wsgi:app
    ```

3. **Deactivate the virtual environment:**

    ```bash
    deactivate
    ```

## 5. Configure Systemd for Gunicorn

1. **Create a systemd service file (`/etc/systemd/system/moviesburn.service`):**

    ```ini
    Description=Gunicorn instance to serve MoviesBurn
    After=network.target
    
    [Service]
    User=root
    Group=www-data
    WorkingDirectory=/var/www/moviesburn.com/MoviesBurn
    Environment="PATH=/var/www/moviesburn.com/MoviesBurn/.movies-env/bin"
    ExecStart=/var/www/moviesburn.com/MoviesBurn/.movies-env/bin/gunicorn --workers 12 --bind unix:/var/www/moviesburn.com/MoviesBurn/moviesburn.sock -m 007 wsgi:app
    ##ExecStart=/var/www/moviesburn.com/MoviesBurn/.movies-env/bin/gunicorn --workers=16 --threads=8 --worker-class=gthread --bind unix:/var/www/moviesburn.com/MoviesBurn/moviesburn.sock -m 007 wsgi:app
    Restart=always
    RestartSec=5
    
    [Install]
    WantedBy=multi-user.target
    ```

2. **Start and enable the Gunicorn service:**

    ```bash
    sudo systemctl start moviesburn.service
    sudo systemctl enable moviesburn.service
    sudo systemctl status moviesburn.service
    ```

## 6. Configure Nginx

1. **Create an Nginx configuration file (`/etc/nginx/sites-available/moviesburn`):**

    ```nginx
    server {
        listen 80;
        listen [::]:80;
        server_name moviesburn.com www.moviesburn.com;
    
        # Redirect all HTTP requests to HTTPS and www subdomain
        return 301 https://moviesburn.com$request_uri;
    }

    server {
        listen 443 ssl;
        listen [::]:443 ssl;
        server_name moviesburn.com www.moviesburn.com;
    
        root /var/www/html;
    
        # Hide server version information
        server_tokens off;
        add_header X-Frame-Options "SAMEORIGIN";
        add_header X-XSS-Protection "1; mode=block";
        add_header X-Content-Type-Options nosniff;
    
        # Handle requests to /phpmyadmin
       location /phpmyadmin {
            alias /usr/share/phpmyadmin/;
            index index.php;
            try_files $uri $uri/ =404;
    
            location ~ \.php$ {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/run/php/php8.3-fpm.sock;
                fastcgi_param SCRIPT_FILENAME $request_filename;
                include fastcgi_params;
            }
        }
        # SSL certificate paths
        ssl_certificate /var/www/moviesburn.com/moviesburn-ssl/movies-burn-ssl.crt;
        ssl_certificate_key /var/www/moviesburn.com/moviesburn-ssl/movies-burn-ssl.key;
    
        # Redirect www HTTPS requests to non-www HTTPS
        if ($host = www.moviesburn.com) {
            return 301 https://moviesburn.com$request_uri;
        }
    
        # Proxy other requests to your Flask app
        location / {
            include proxy_params;
            proxy_pass http://unix:/var/www/moviesburn.com/MoviesBurn/moviesburn.sock;
        }
    
        # Serve static files with caching and gzip compression
        location /static/ {
            root /var/www/moviesburn.com/MoviesBurn/;
            expires 1y;
            add_header Cache-Control "public, max-age=31536000";
            include /etc/nginx/mime.types;
            gzip on;
            gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
        }
    }
    ```

2. **Enable the Nginx site and remove default configuration:**

    ```bash
    sudo ln -s /etc/nginx/sites-available/moviesburn /etc/nginx/sites-enabled
    sudo rm /etc/nginx/sites-enabled/default
    sudo rm /etc/nginx/sites-available/default
    ```

3. **Test and restart Nginx:**

    ```bash
    sudo nginx -t
    sudo systemctl restart nginx
    ```

4. **Update UFW firewall rules:**

    ```bash
    sudo ufw delete allow 5000
    sudo ufw allow 'Nginx Full'
    ```

## Conclusion

Your Flask application should now be up and running, served by Gunicorn and Nginx. Access your site using `http://moviesburn.com`.

## References

- [DigitalOcean Tutorial](https://www.digitalocean.com/community/tutorials/how-to-serve-flask-applications-with-gunicorn-and-nginx-on-ubuntu-18-04)
- [Code With Harry Blog Post](https://www.codewithharry.com/blogpost/flask-app-deploy-using-gunicorn-nginx/)
