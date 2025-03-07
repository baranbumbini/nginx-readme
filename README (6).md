# Instalación y Configuración de una Aplicación Flask con Gunicorn y Nginx

## Instalamos el gestor de paquetes de Python pip:
```sh
sudo apt-get update && sudo apt-get install -y python3-pip
```

## Instalamos el paquete pipenv para gestionar los entornos virtuales:
```sh
pip3 install pipenv
```

Y comprobamos que está instalado correctamente mostrando su versión:
```sh
USER/.local/bin
pipenv --version
```

Después instalaremos el paquete `python-dotenv` para cargar las variables de entorno.
```sh
pip3 install python-dotenv
```

## Creación del Directorio del Proyecto

Creamos el directorio en el que almacenaremos nuestro proyecto:

> Sustituir `app` por el nombre de la aplicación
```sh
sudo mkdir -p /var/www/app
```

Al crearlo con `sudo`, los permisos pertenecen a root, por lo que hay que cambiarlo para que el dueño sea nuestro usuario y pertenezca al grupo `www-data`, el usuario usado por defecto por el servidor web para ejecutarse:
```sh
sudo chown -R $USER:www-data /var/www/app
```

Establecemos los permisos adecuados a este directorio, para que pueda ser leído por todo el mundo:
```sh
sudo chmod -R 775 /var/www/app
```

> **Advertencia**
> Es indispensable asignar estos permisos, de otra forma obtendríamos un error al acceder a la aplicación cuando pongamos en marcha Nginx.

## Creación del Archivo `.env`

Dentro del directorio de nuestra aplicación, creamos un archivo oculto `.env` que contendrá las variables de entorno necesarias.

**Contenido del fichero `/var/www/app/.env`**
```sh
FLASK_APP=wsgi.py
FLASK_ENV=production
```

## Iniciar el Entorno Virtual
```sh
pipenv shell
```

Usamos `pipenv install` para instalar las dependencias necesarias para nuestro proyecto:
```sh
pipenv install flask gunicorn
```

## Creación de la Aplicación Flask

```sh
touch application.py wsgi.py
```

**Contenido de `/var/www/app/application.py`**
```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def index():
    '''Index page route'''
    return '<h1>App desplegada</h1>'
```

**Contenido de `/var/www/app/wsgi.py`**
```python
from application import app

if __name__ == '__main__':
   app.run(debug=False)
```

Ejecutamos la aplicación para comprobar su funcionamiento:
```sh
flask run --host '0.0.0.0'
```

Ahora podremos acceder a la aplicación desde nuestro ordenador, nuestra máquina anfitrión, introduciendo en un navegador web: http://IP-maq-virtual:5000 aparecerá App desplegada.

Tras la comprobación, paramos el servidor con CTRL+C. Comprobemos ahora que Gunicorn funciona correctamente también. Si os ha funcionado el servidor de desarrollo de Flask, podéis usar el comando gunicorn --workers 4 --bind 0.0.0.0:5000 wsgi:app para probar que la alicación funciona correctamente usando Gunicorn, accediendo con vuestro navegador de la misma forma que en el paso anterior:

## Ejecutar Gunicorn
```sh
gunicorn --workers 4 --bind 0.0.0.0:5000 wsgi:app
```

Para conocer el path de `gunicorn`, ejecutamos:
```sh
which gunicorn
```

ahora instalamos nginx
sudo apt install nginx
sudo systemctl start nginx
sudo systemctl status nginx

## Configuración de `systemd` para Gunicorn

**Fichero `/etc/systemd/system/flask_app.service`**
```ini
[Unit]
Description=flask app service - App con flask y Gunicorn
After=network.target

[Service]
User=vagrant
Group=www-data
Environment="PATH=/home/vagrant/.local/share/virtualenvs/app-1lvW3LzD/bin"
WorkingDirectory=/var/www/app
ExecStart=/home/vagrant/.local/share/virtualenvs/app-1lvW3LzD/bin/gunicorn --workers 3 --bind unix:/var/www/app/app.sock wsgi:app

[Install]
WantedBy=multi-user.target
```

Informamos a `systemd` sobre el nuevo servicio:
```sh
sudo systemctl daemon-reload
systemctl enable flask_app
systemctl start flask_app
```

## Configuración de Nginx

**Fichero `/etc/nginx/sites-available/app.conf`**
```nginx
server {
  listen 80;
  server_name app.izv www.app.izv;

  access_log /var/log/nginx/app.access.log;
  error_log /var/log/nginx/app.error.log;

  location / {
    include proxy_params;
    proxy_pass http://unix:/var/www/app/app.sock;
  }
}
```

Creamos un enlace simbólico para habilitar la configuración:
```sh
sudo ln -s /etc/nginx/sites-available/app.conf /etc/nginx/sites-enabled/
```

Comprobamos la sintaxis de la configuración y reiniciamos Nginx:
```sh
sudo nginx -t
sudo systemctl restart nginx
sudo systemctl status nginx
```

## Edición del Archivo `/etc/hosts`

Editamos `/etc/hosts` (Linux) o `C:\Windows\System32\drivers\etc\hosts` (Windows) y agregamos:
```sh
192.168.X.X app.izv www.app.izv
```

Finalmente, accedemos desde el navegador a:
```sh
http://app.izv/ o http://www.app.izv/
```
Y debería mostrarse:
```html
<h1>App desplegada</h1>
```


 
