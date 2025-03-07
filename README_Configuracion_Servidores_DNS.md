# 1. Configuración de los Servidores DNS

En este apartado configuramos los nombres de los servidores en Vagrant. La idea es que `atlas` y `ceo` tengan sus respectivos nombres de host bien definidos desde el inicio. No es complicado, pero hay que hacerlo bien para evitar problemas después.

## 1️.1 Configurar los nombres de los servidores en Vagrant

Para que cada servidor tenga su nombre de host correcto, editamos el archivo `Vagrantfile` y añadimos lo siguiente:

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "debian/bullseye64"

  config.vm.provider "virtualbox" do |vb|
    vb.memory = "256"
    vb.linked_clone = true
  end

  # Actualizar SO
  config.vm.provision "update", type: "shell", inline: <<-SHELL
    apt-get update 
  SHELL

  ######################################################################
  # Servidor 'atlas'
  ######################################################################
  config.vm.define "atlas" do |atlas|
    atlas.vm.hostname = "atlas"  # Asignamos el nombre de host
    atlas.vm.network "private_network", ip: "192.168.56.10"  # IP fija

    atlas.vm.provision "bind9-install", type: "shell", inline: <<-SHELL
        apt-get install -y bind9 bind9-utils bind9-doc
    SHELL
  end

  ######################################################################
  # Servidor 'ceo'
  ######################################################################
  config.vm.define "ceo" do |ceo|
    ceo.vm.hostname = "ceo"  # Asignamos el nombre de host
    ceo.vm.network "private_network", ip: "192.168.56.11"  # IP fija

    ceo.vm.provision "bind9-install", type: "shell", inline: <<-SHELL
        apt-get install -y bind9 bind9-utils bind9-doc
    SHELL
  end
end
```

## 1.2️ Iniciar las máquinas virtuales

Después de modificar el Vagrantfile, ejecutamos:

```bash
vagrant up
```

Si ya estaban encendidas, mejor reiniciarlas para que cojan bien los cambios:

```bash
vagrant reload --provision
```

## 1.3 Comprobar que los nombres están bien asignados

Para asegurarnos de que cada máquina tiene su nombre de host correcto, nos conectamos a cada una y ejecutamos:

```bash
vagrant ssh atlas #o ceo
hostname
exit
```

Si en cada caso aparece el nombre correcto (atlas o ceo), significa que todo está bien configurado.

# 2. Configurar la dirección IP de atlas y ceo
En este paso, se han asignado las direcciones IP a los servidores:

    Atlas: 192.168.56.10
    Ceo: 192.168.56.11

Para asegurarnos de que todo está bien, hemos comprobado las IPs con:

```bash
vagrant ssh atlas
ip a | grep 192.168.56
exit
```

```bash
vagrant ssh ceo
ip a | grep 192.168.56
exit
```
Si aparecen las IPs correctas, significa que la configuración está bien hecha.

# 3. Configurar la zona DNS olimpo.test
En este apartado configuramos la zona DNS olimpo.test para que atlas sea el servidor primario y ceo el secundario. Esto permitirá que atlas gestione los registros y que ceo reciba las actualizaciones de la zona cuando haya cambios.

## 3.1 Configurar las zonas en los servidores
### 3.1.1 En atlas
Editamos el archivo de configuración local de BIND:

```bash
sudo nano /etc/bind/named.conf.local
```

Añadimos:

```bash
zone "olimpo.test" {
    type master;
    file "/var/lib/bind/db.olimpo";
    allow-transfer { 192.168.56.3; };
    notify yes;
};

zone "56.168.192.in-addr.arpa" {
    type master;
    file "/var/lib/bind/db.olimpo.rev";
    allow-transfer { 192.168.56.3; };
    notify yes;
};
```
Guardamos y cerramos.

### 3.1.2 En ceo
Abrimos el mismo archivo:

```bash

sudo nano /var/lib/bind/named.conf.local
```

Añadimos:

```bash
zone "olimpo.test" {
    type slave;
    file "/var/cache/bind/db.olimpo";
    masters { 192.168.56.10; };
};

zone "56.168.192.in-addr.arpa" {
    type slave;
    file "/var/cache/bind/db.olimpo.rev";
    masters { 192.168.56.10; };
};
```

Guardamos y cerramos.

## 3.2 Crear los archivos de zona en atlas
### 3.2.1 Zona directa

```bash
sudo nano /var/lib/bind/db.olimpo
```

y añadimos:

```bash
$TTL 86400
@   IN  SOA atlas. hefestos.olimpo.test. (
        1        ; Serial
        1200     ; Refresh
        900      ; Retry
        2419200  ; Expire
        86400    ; Negative Cache TTL
)

@   IN  NS  atlas.olimpo.test.
atlas   IN  A   192.168.56.10
ceo     IN  A   192.168.56.11
```

### 3.2.2 Zona inversa
```bash
sudo nano /var/lib/bind/db.olimpo.rev
```

```bash
$TTL 86400
@   IN  SOA atlas. hefestos.olimpo.test. (
        1        ; Serial
        1200     ; Refresh
        900      ; Retry
        2419200  ; Expire
        86400    ; Negative Cache TTL
)

@   IN  NS  atlas.olimpo.test.
10  IN  PTR atlas.olimpo.test.
11  IN  PTR ceo.olimpo.test.
```
## 3.3 Reiniciar BIND y comprobar
### 3.3.1 Reiniciar BIND

En atlas:

```bash
sudo systemctl restart bind9
sudo systemctl status bind9
```

En ceo:

```bash
sudo systemctl restart bind9
sudo systemctl status bind9
```

### 3.3.2 Verificar la transferencia de zona

En ceo:

```bash
ls -l /var/cache/bind/
```

Si aparecen los archivos db.olimpo y db.olimpo.rev, significa que la transferencia ha funcionado.

# 4. Configurar el responsable de los servidores y los tiempos de refresco
En este paso, nos aseguramos de que el responsable de los servidores sea hefestos@olimpo.test y configuramos los tiempos de refresco y reintento en los archivos de zona.

## 4.1 Modificar los archivos de zona en atlas
Abrimos el archivo de zona directa:

```bash
sudo nano /etc/bind/db.olimpo
```

Nos aseguramos de que el bloque SOA tenga los valores correctos:

```bash
@   IN  SOA atlas. hefestos.olimpo.test. (
        2        ; Serial
        1200     ; Refresh (20 min)
        900      ; Retry (15 min)
        2419200  ; Expire
        86400    ; Negative Cache TTL
)
```

Hacemos lo mismo en la zona inversa:

```bash
sudo nano /etc/bind/db.olimpo.rev
```

Nos aseguramos de actualizar el número de serie y guardamos los cambios.

## 4.2 Recargar BIND y verificar
Después de modificar los archivos, reiniciamos BIND en atlas:

```bash
sudo systemctl restart bind9
```

Para asegurarnos de que ceo recibe la actualización, ejecutamos en ceo:

```bash
sudo rndc retransfer olimpo.test
sudo rndc reload
```

Luego verificamos si la actualización se ha propagado correctamente:

```bash
dig @192.168.56.11 SOA olimpo.test
```

Si el número de serie mostrado coincide con el de atlas, la configuración se ha aplicado correctamente.

# 5. Configurar los reenviadores de los servidores DNS
En este paso, configuramos los reenviadores en atlas y ceo, para que cuando no puedan resolver una consulta internamente, la envíen al servidor DNS de Cloudflare (1.1.1.1).

## 5.1 Modificar named.conf.options en atlas y ceo
Abrimos el archivo en atlas:

```bash
sudo nano /etc/bind/named.conf.options
```

Y añadimos la configuración de los reenviadores dentro del bloque options {}:

```bash
options {
    directory "/var/cache/bind";

    // Habilitar reenviadores
    forwarders {
        1.1.1.1;
    };

    dnssec-validation no;

    listen-on-v6 { any; };
};
```

Hacemos lo mismo en ceo:

```bash
sudo nano /etc/bind/named.conf.options
```

Y agregamos la misma configuración.

## 5.2 Reiniciar BIND en atlas y ceo
Para aplicar los cambios, reiniciamos el servicio en ambos servidores:

```bash
sudo systemctl restart bind9
sudo systemctl status bind9
```

Si el servicio está activo y sin errores, significa que la configuración ha sido aplicada correctamente.

## 5.3 Comprobar la resolución de nombres
Para verificar que los reenviadores funcionan, probamos a resolver un dominio externo desde ceo:

```bash
dig @192.168.56.11 google.com
```

Si obtenemos una respuesta con una dirección IP en la ANSWER SECTION, significa que ceo ha reenviado la consulta a Cloudflare (1.1.1.1) y ha recibido una respuesta válida.

# 6. Crear los registros necesarios en la zona directa e inversa
En este paso, añadimos los registros DNS para los servidores y equipos de la red en la zona directa e inversa.

## 6.1 Añadir registros en la zona directa (db.olimpo)
Editamos el archivo de la zona directa en atlas:

```bash
sudo nano /etc/bind/db.olimpo
```

Añadimos los registros A para los dispositivos de la red:

```bash
$TTL 86400
@   IN  SOA atlas. hefestos.olimpo.test. (
        3        ; Serial
        1200     ; Refresh (20 min)
        900      ; Retry (15 min)
        2419200  ; Expire
        86400    ; Negative Cache TTL
)

; Servidores de nombres
@   IN  NS  atlas.olimpo.test.

; Registros A (asociación de nombres a IPs)
atlas   IN  A   192.168.56.10
ceo     IN  A   192.168.56.11
atenea  IN  A   192.168.56.20
mercurio IN  A   192.168.56.30
ares    IN  A   192.168.56.40
dionisio IN  A   192.168.56.50
```

Guardamos y cerramos.

## 6.2 Añadir registros en la zona inversa (db.olimpo.rev)
Editamos el archivo de la zona inversa en atlas:

```bash
sudo nano /etc/bind/db.olimpo.rev
```

Añadimos los registros PTR para la resolución inversa:

```bash
$TTL 86400
@   IN  SOA atlas. hefestos.olimpo.test. (
        3        ; Serial
        1200     ; Refresh (20 min)
        900      ; Retry (15 min)
        2419200  ; Expire
        86400    ; Negative Cache TTL
)

; Servidores de nombres
@   IN  NS  atlas.olimpo.test.

; Registros PTR (IP a nombre)
10  IN  PTR atlas.olimpo.test.
11  IN  PTR ceo.olimpo.test.
20  IN  PTR atenea.olimpo.test.
30  IN  PTR mercurio.olimpo.test.
40  IN  PTR ares.olimpo.test.
50  IN  PTR dionisio.olimpo.test.
```

Guardamos y cerramos.

## 6.3 Reiniciar BIND en atlas y verificar
Después de actualizar los archivos, aumentamos el número de serie y reiniciamos el servicio en atlas:

```bash
sudo systemctl restart bind9
```

Verificamos que la configuración es válida con:

```bash
sudo named-checkzone olimpo.test /etc/bind/db.olimpo
sudo named-checkzone 56.168.192.in-addr.arpa /etc/bind/db.olimpo.rev
```

Si no hay errores, pasamos al siguiente paso.

## 6.4 Forzar la actualización en ceo
Para asegurarnos de que ceo recibe la nueva configuración, ejecutamos en ceo:

```bash
sudo rndc retransfer olimpo.test
sudo rndc reload
```

Verificamos la resolución de los nuevos registros con:

```bash
dig @192.168.56.11 atenea.olimpo.test
dig @192.168.56.11 -x 192.168.56.20
```

Si la respuesta es correcta, significa que la transferencia de zona ha funcionado.

# 7. Crear alias (CNAME) en la zona directa
En este paso, añadimos alias a ciertos equipos mediante registros CNAME. Estos alias permiten referirse a servicios con nombres más amigables dentro del dominio olimpo.test.

## 7.1 Añadir registros CNAME en atlas
Editamos el archivo de la zona directa en atlas:

```bash
sudo nano /etc/bind/db.olimpo
```

Añadimos los alias al final del archivo:

```bash
; Alias (CNAME)
ftp     IN  CNAME atenea.olimpo.test.
smtp    IN  CNAME mercurio.olimpo.test.
pop     IN  CNAME mercurio.olimpo.test.
www     IN  CNAME ares.olimpo.test.
```

Guardamos y cerramos.

Nota: Es normal tener más de un alias apuntando al mismo equipo, como es el caso de smtp y pop para mercurio.olimpo.test, ya que un mismo servidor puede manejar múltiples servicios, como el envío y recepción de correos.

## 7.2 Actualizar el número de serie y reiniciar BIND
Para asegurar que los cambios sean replicados, incrementamos el número de serie en el archivo de zona:

```bash
sudo nano /etc/bind/db.olimpo
```

Modificamos el número de serie en el SOA:

```bash
@   IN  SOA atlas. hefestos.olimpo.test. (
        4        ; Serial
```

Guardamos y reiniciamos BIND en atlas:

```bash
sudo systemctl restart bind9
```

Verificamos que no haya errores:

```bash
sudo named-checkzone olimpo.test /etc/bind/db.olimpo
```

## 7.3 Forzar la actualización en ceo
Para que ceo reciba los nuevos alias, ejecutamos:

```bash
sudo rndc retransfer olimpo.test
sudo rndc reload
```

Verificamos que los alias funcionan correctamente:

```bash
dig @192.168.56.11 ftp.olimpo.test
dig @192.168.56.11 smtp.olimpo.test
dig @192.168.56.11 www.olimpo.test
```

Si obtenemos respuestas válidas, los alias se han configurado correctamente. 

# 8. Añadir los registros de los servidores de correo
En este paso, configuramos los registros MX (Mail Exchange) para definir los servidores de correo de olimpo.test.

mercurio.olimpo.test será el servidor principal con prioridad 10.
dionisio.olimpo.test será el servidor de respaldo con prioridad 20.

Cuanto menor es el número de prioridad, mayor será la preferencia del servidor para recibir correos.

## 8.1 Añadir los registros MX en atlas
Editamos la zona directa en atlas:

```bash
sudo nano /etc/bind/db.olimpo
```

Añadimos los registros MX debajo de los registros existentes:

```bash
; Servidores de correo (MX)
@       IN  MX  10 mercurio.olimpo.test.
@       IN  MX  20 dionisio.olimpo.test.
```

Guardamos y cerramos.

## 8.2 Actualizar el número de serie y reiniciar BIND
Aumentamos el número de serie para que ceo detecte la actualización:

```bash
sudo nano /etc/bind/db.olimpo
```

Modificamos la línea del SOA:

```bash
@   IN  SOA atlas. hefestos.olimpo.test. (
        6        ; Serial
```

Guardamos y cerramos.

Reiniciamos BIND en atlas:

```bash
sudo systemctl restart bind9
```

Verificamos que la configuración sea válida:

```bash
sudo named-checkzone olimpo.test /etc/bind/db.olimpo
```

Si no hay errores, pasamos a ceo.

## 8.3 Forzar la actualización en ceo
Para que ceo reciba los registros MX, ejecutamos:

```bash
sudo rndc retransfer olimpo.test
sudo rndc reload
```

Comprobamos que los registros se han propagado correctamente:

```bash
dig @192.168.56.11 MX olimpo.test
```
Si la respuesta muestra los servidores con sus prioridades correctamente, la configuración ha sido exitosa. 