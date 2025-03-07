# Configuración de Proxy Inverso con Nginx en Vagrant

Este tutorial te guiará paso a paso para configurar un **proxy inverso** con **Nginx** en **Vagrant**, utilizando dos máquinas virtuales Debian:

- **proxy** (192.168.57.10) → Recibe las peticiones y las redirige al servidor web.
- **web (w1)** (192.168.57.11) → Servidor web que responderá a las peticiones.

## 1. Configurar el entorno en Vagrant

### Crear y configurar el Vagrantfile

Crea un directorio para el proyecto y dentro de él un archivo `Vagrantfile`:

```sh
mkdir vagrant-proxy
cd vagrant-proxy
nano Vagrantfile
```

Copia y pega el siguiente contenido:

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "debian/bullseye64"
  
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "256"
  end

  config.vm.provision "shell", inline: <<-SHELL
    apt-get update && apt-get install -y nginx
  SHELL

  config.vm.define "proxy" do |proxy|
    proxy.vm.hostname = "www.example.test"
    proxy.vm.network "private_network", ip: "192.168.57.10"
  end

  config.vm.define "web" do |web|
    web.vm.hostname = "w1.example.test"
    web.vm.network "private_network", ip: "192.168.57.11"
  end  
end
```

Guarda y cierra el archivo.

### Levantar las máquinas virtuales

Ejecuta el siguiente comando en la terminal:

```sh
vagrant up
```

---

## 2. Configurar el Servidor Web (w1)

Accede a la máquina `web`:

```sh
vagrant ssh web
```

Edita la configuración de Nginx:

```sh
sudo nano /etc/nginx/sites-available/default
```

Modifica el contenido:

```nginx
server {
    listen 8080;
    listen [::]:8080;
    
    server_name w1;
    root /var/www/html;
    index index.html index.htm;
    
    location / {
        try_files $uri $uri/ =404;
    }
}
```

Reinicia Nginx:

```sh
sudo systemctl restart nginx
```

Crea una página web de prueba:

```sh
sudo nano /var/www/html/index.html
```

Copia el siguiente contenido:

```html
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>example.test</title>
</head>
<body>
    <h1>example.test</h1>
    <h2>Bienvenido</h2>
    <p>Servidor w1</p>
</body>
</html>
```

Verifica que el servidor web está funcionando:

```sh
curl http://localhost:8080
```

Sal de la máquina con `exit`.

---

## 3. Configurar el Proxy Inverso

Accede a la máquina `proxy`:

```sh
vagrant ssh proxy
```

Edita el archivo `/etc/hosts` para resolver `w1`:

```sh
sudo nano /etc/hosts
```

Agrega la siguiente línea:

```
192.168.57.11 w1
```

Edita la configuración de Nginx:

```sh
sudo nano /etc/nginx/sites-available/default
```

Modifica el contenido para configurar el proxy inverso:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name example.test www.example.test;

    location / {
        proxy_pass http://192.168.57.11:8080;
    }
}
```

Reinicia Nginx:

```sh
sudo systemctl restart nginx
```

Sal de la máquina con `exit`.

---

## 4. Comprobación del Proxy Inverso

### Acceder a la web desde el navegador

Abre el navegador y accede a:

```
http://www.example.test
```

### Revisar los logs en `proxy`

```sh
vagrant ssh proxy
sudo tail /var/log/nginx/access.log
exit
```

### Revisar los logs en `web`

```sh
vagrant ssh web
sudo tail /var/log/nginx/access.log
exit
```

---

## 5. Añadir Cabeceras Personalizadas

### En `proxy`

```sh
vagrant ssh proxy
sudo nano /etc/nginx/sites-available/default
```

Modifica el bloque `location /`:

```nginx
location / {
    proxy_pass http://192.168.57.11:8080;
    add_header X-friend "TuNombre";
}
```

Reinicia Nginx:

```sh
sudo systemctl restart nginx
exit
```

### En `web`

```sh
vagrant ssh web
sudo nano /etc/nginx/sites-available/default
```

Modifica el bloque `location /`:

```nginx
location / {
    add_header Host w1.example.test;
    try_files $uri $uri/ =404;
}
```

Reinicia Nginx:

```sh
sudo systemctl restart nginx
exit
```

### Verificar en el Navegador

1. Abre Firefox y presiona `Ctrl + Shift + I`.
2. Ve a la pestaña **Network** y marca **Disable cache**.
3. Recarga la página `http://www.example.test`.
4. Revisa los **Response Headers** y deberías ver:

```
X-friend: TuNombre
Host: w1.example.test
```

---
