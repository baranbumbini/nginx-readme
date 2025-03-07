# **Despliegue de una Aplicación en Clúster con Node.js y Express**

## **Introducción**
En este tutorial, aprenderemos a desplegar una aplicación Node.js con Express en un entorno clúster, aprovechando múltiples núcleos de la CPU para mejorar el rendimiento. También utilizaremos PM2 para la gestión de procesos y realizaremos pruebas de carga con `loadtest`.

---

## **Requisitos Previos**
- Tener instalado **Vagrant** y **VirtualBox**.
- Tener instalado **Node.js** y **npm** en la VM.
- Conocimientos básicos de **Linux y SSH**.

---

## **Paso 1: Configurar el Entorno en Vagrant**

### **1. Iniciar la máquina virtual**
```sh
vagrant up
vagrant ssh
```

### **2. Actualizar el sistema y verificar Node.js**
```sh
sudo apt update && sudo apt install -y nodejs npm
node -v
npm -v
```

---

## **Paso 2: Crear la Aplicación sin Clúster**

### **1. Crear el proyecto**
```sh
mkdir mi_proyecto && cd mi_proyecto
```

### **2. Inicializar un proyecto Node.js**
```sh
npm init -y
```

### **3. Instalar Express**
```sh
npm install express
```

### **4. Crear `app.js`**
```javascript
const express = require("express");
const app = express();
const port = 3000;
const limit = 5000000000;

app.get("/", (req, res) => {
    res.send("Hello World!");
});

app.get("/api/:n", function (req, res) {
    let n = parseInt(req.params.n);
    let count = 0;
    if (n > limit) n = limit;
    for (let i = 0; i <= n; i++) {
        count += i;
    }
    res.send(`Final count is ${count}`);
});

app.listen(port, () => {
    console.log(`App listening on port ${port}`);
});
```

### **5. Ejecutar la Aplicación**
```sh
node app.js
```

Prueba en el navegador:
- `http://192.168.33.10:3000`
- `http://192.168.33.10:3000/api/50`
- `http://192.168.33.10:3000/api/5000000000`

---

## **Paso 3: Implementar Clúster en la Aplicación**

### **1. Crear `app_cluster.js`**
```javascript
const express = require("express");
const cluster = require("cluster");
const os = require("os");
const port = 3000;
const totalCPUs = os.cpus().length;

if (cluster.isMaster) {
    console.log(`Master ${process.pid} en ejecución`);
    for (let i = 0; i < totalCPUs; i++) {
        cluster.fork();
    }
    cluster.on("exit", (worker) => {
        console.log(`Worker ${worker.process.pid} murió, reiniciando...`);
        cluster.fork();
    });
} else {
    const app = express();
    app.get("/", (req, res) => res.send("Hello World!"));
    app.get("/api/:n", (req, res) => {
        let n = parseInt(req.params.n);
        let count = 0;
        if (n > 5000000000) n = 5000000000;
        for (let i = 0; i <= n; i++) count += i;
        res.send(`Final count is ${count}`);
    });
    app.listen(port, () => console.log(`Worker ${process.pid} escuchando en ${port}`));
}
```

### **2. Ejecutar la Aplicación con Clúster**
```sh
node app_cluster.js
```

---

## **Paso 4: Pruebas de Carga con Loadtest**

### **1. Instalar Loadtest**
```sh
npm install -g loadtest
```

### **2. Prueba en la aplicación sin clúster**
```sh
loadtest http://localhost:3000/api/500000 -n 1000 -c 100
```

### **3. Prueba en la aplicación con clúster**
```sh
loadtest http://localhost:3000/api/500000 -n 1000 -c 100
```

---

## **Paso 5: Automatización con PM2**

### **1. Instalar PM2**
```sh
npm install -g pm2
```

### **2. Ejecutar aplicación sin modificar código**
```sh
pm2 start app.js -i 0
```

### **3. Verificar procesos**
```sh
pm2 ls
```

### **4. Detener los procesos**
```sh
pm2 stop all
pm2 delete all
```

---

## **Paso 6: Configuración con `ecosystem.config.js`**

### **1. Generar el archivo de configuración**
```sh
pm2 ecosystem
```

### **2. Modificar `ecosystem.config.js`**
```javascript
module.exports = {
    apps: [{
        name: "mi_app",
        script: "app.js",
        instances: 0,
        exec_mode: "cluster"
    }]
};
```

### **3. Ejecutar aplicación con PM2**
```sh
pm2 start ecosystem.config.js
```

### **4. Comprobaciones**
```sh
pm2 ls
pm2 logs
pm2 monit
```

---

