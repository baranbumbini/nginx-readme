# Práctica Tomcat y Maven:

## Instalación de OpenJDK

```bash
sudo apt install -y openjdk-11-jdk
```

## Instalación de Tomcat 9

1. Instalar paquete Tomcat:
```bash
sudo apt install -y tomcat9
```

2. Crear grupo de usuarios:
```bash
sudo groupadd tomcat9
```

3. Crear usuario de servicio:
```bash
sudo useradd -s /bin/false -g tomcat9 -d /etc/tomcat9 tomcat9
```

4. Iniciar y verificar servicio:
```bash
sudo systemctl start tomcat9
sudo systemctl status tomcat9
```

## Configuración de Administración

1. Editar configuración de usuarios (`/etc/tomcat9/tomcat-users.xml`):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<tomcat-users>
  <role rolename="admin"/>
  <role rolename="admin-gui"/>
  <role rolename="manager"/>
  <role rolename="manager-gui"/>
  <user username="alumno" 
        password="1234" 
        roles="admin,admin-gui,manager,manager-gui"/>
</tomcat-users>
```

2. Instalar administrador web:
```bash
sudo apt install -y tomcat9-admin
```

## Instalación de Maven

```bash
sudo apt-get update
sudo apt-get -y install maven
```

## Configuración de Maven para Despliegue

1. Actualizar `tomcat-users.xml` con roles adicionales:
```xml
<role rolename="manager-script"/>
<user username="deploy" password="1234" roles="manager-script"/>
```

2. Configurar `settings.xml`:
```xml
<servers>
  <server>
    <id>Tomcat</id>
    <username>deploy</username>
    <password>1234</password>
  </server>
</servers>
```

## Crear Proyecto Maven de Ejemplo

```bash
mvn archetype:generate -DgroupId=org.zaidinvergeles \
-DartifactId=tomcat-war-deployment \
-DarchetypeArtifactId=maven-archetype-webapp \
-DinteractiveMode=false
```

## Configurar POM para Despliegue Tomcat

En `pom.xml`, añadir:
```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.apache.tomcat.maven</groupId>
      <artifactId>tomcat7-maven-plugin</artifactId>
      <version>2.2</version>
      <configuration>
        <url>http://localhost:8080/manager/text</url>
        <server>Tomcat</server>
        <path>/despliegue</path>
      </configuration>
    </plugin>
  </plugins>
</build>
```

## Comandos de Despliegue Maven

- Desplegar: `mvn tomcat7:deploy`
- Redesplegar: `mvn tomcat7:redeploy`
- Retirar: `mvn tomcat7:undeploy`

## Capturas de Pantalla

- Se encuentran todas en la carpeta de imagenes

