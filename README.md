# Manual de Uso

Este documento describe los pasos para clonar el repositorio, configurar el entorno y probar las funcionalidades del sistema utilizando Docker y Traefik además de demostrar su funcionalidad.

## Compatibilidad con navegadores  
Este sistema ha sido probado en los navegadores Brave y Opera, donde funciona correctamente. Sin embargo, en Firefox pueden presentarse problemas con la resolución de nombres de dominio locales.  

✅ Funciona correctamente en:
- Brave
- Opera

⚠️ Problemas conocidos en:
- Firefox (puede requerir configuración adicional)  
    

## 1. Clonar el Repositorio

Para obtener el código fuente del proyecto, ejecute los siguientes comandos en la terminal:

```bash
git clone https://github.com/julianReyes-dev/traefik-taller-marzo-25.git
```
```bash
cd traefik-taller-marzo-25
```

## 2. Configurar el archivo Hosts

Para que los nombres de dominio locales (como `nginx.localhost` y `api.localhost`) funcionen correctamente, es necesario agregarlos al archivo `/etc/hosts`. Esto permite que las solicitudes dirigidas a estos dominios sean resueltas correctamente en el entorno local.

### Método 1: Usando terminal con nano

1. Abre una terminal y ejecuta:
   ```bash
   sudo nano /etc/hosts
   ```
2. Agrega estas líneas al final del archivo:
   ```bash
   127.0.0.1 traefik.localhost
   127.0.0.1 nginx.localhost
   127.0.0.1 api.localhost
   ```
3. Guarda los cambios:
   - Presiona `Ctrl+O` para guardar
   - Presiona `Enter` para confirmar
   - Presiona `Ctrl+X` para salir

### Método 2: Usando terminal con Vim

1. Abre una terminal y ejecuta:
   ```bash
   sudo vim /etc/hosts
   ```
2. Presiona `i` para entrar en modo edición.
3. Agrega estas líneas al final del archivo:
   ```bash
   127.0.0.1 traefik.localhost
   127.0.0.1 nginx.localhost
   127.0.0.1 api.localhost
   ```
4. Guarda los cambios y sal de Vim:
   - Presiona `Esc`
   - Escribe `:wq` y presiona `Enter`

Verifica que los cambios se aplicaron correctamente con:

```bash
cat /etc/hosts | grep localhost
```

Este paso es fundamental para que los contenedores de Docker puedan ser accedidos mediante nombres de dominio en lugar de direcciones IP.

## 3. Levantar los Servicios

Ahora, inicie los servicios utilizando `docker-compose`:

```bash
docker-compose up -d
```

Este comando inicia los contenedores en segundo plano (`-d` significa "detached"), permitiendo que el sistema funcione sin bloquear la terminal.
![imagen](https://github.com/user-attachments/assets/7d6b5711-b0ca-45fb-a217-5ddc13ea71d6)


## 4. Probar los servicios

Para verificar que los servicios están funcionando correctamente, realice las siguientes pruebas:

### Desde terminal
```bash
# Acceder a Nginx (público)
curl http://nginx.localhost

# Acceder a la API (con autenticación)
curl http://api.localhost/api -u test:test

# Acceder al dashboard de Traefik
curl http://traefik.localhost:8080
```
- Este comando verifica que el servidor web Nginx esté activo y accesible.  
- Aquí se está accediendo a la API, que requiere autenticación básica con usuario `test` y contraseña `test`.  
- Traefik proporciona un panel de administración que permite visualizar el estado de las conexiones y las reglas de enrutamiento.

![imagen](https://github.com/user-attachments/assets/b71fc92a-e5eb-4ef6-adb8-160d43a0a5c6)

### Desde navegador
Nginx
```bash
http://nginx.localhost
```
![imagen](https://github.com/user-attachments/assets/fe7d18fd-6b31-47c9-ad14-de5a13ddc6be)


API (whoami) con credenciales test/test
```bash
http://api.localhost/api
```
![imagen](https://github.com/user-attachments/assets/8000a032-b133-4210-ba39-93c943b3d1d6)


Traefik
```bash
http://traefik.localhost:8080
```
![imagen](https://github.com/user-attachments/assets/d0a1a5e5-baed-4c4a-a8a4-0cb01cb5dc73)



## 5. Ver Logs

Para diagnosticar problemas o analizar el tráfico de los servicios, se pueden consultar los logs de Traefik.

### Ver todos los logs en formato JSON
```bash
docker logs traefik | grep -E '^{'
```
![imagen](https://github.com/user-attachments/assets/8d9718aa-5bab-4930-9a3f-de8a1ceca18b)

Esto muestra los registros generados por Traefik en formato JSON.

### Filtrar logs por servicio
```bash
docker logs traefik | grep -E '^{"' | jq 'select(.RequestHost == "nginx.localhost")'
docker logs traefik | grep -E '^{"' | jq 'select(.RequestHost == "api.localhost")'
```
![imagen](https://github.com/user-attachments/assets/75ef134e-d87b-4f36-8163-84f26e91ba07)

Estos comandos filtran los registros por servicio, permitiendo analizar el tráfico de `nginx.localhost` o `api.localhost`.

### Formato resumido
```bash
docker logs traefik | grep -E '^{"' | jq -c '[.time, .RequestHost, .RequestMethod, .RequestPath, .DownstreamStatus, .ClientUsername]'
```
![imagen](https://github.com/user-attachments/assets/9c206c09-743a-466d-b460-1af0e4536dd9)

Este comando muestra una versión simplificada de los logs, resaltando la información más relevante y comprobando que la conexión es satisfactoria.  

>Los logs se generan gracias a --accesslog=true y --accesslog.format=json  
>Se almacenan en el contenedor Traefik y se acceden con docker logs

## 6. Probar la Autenticación

El sistema utiliza autenticación HTTP básica para proteger el acceso a la API.

### Desde terminal

#### Prueba exitosa (usuario: test, contraseña: test)
```bash
curl -v http://api.localhost/api -u test:test
```
**Debe devolver:** `200 OK`, lo que indica un acceso exitoso.
![imagen](https://github.com/user-attachments/assets/a1e737de-773d-41b0-acfb-29d509d93699)


#### Prueba fallida (credenciales incorrectas)
```bash
curl -v http://api.localhost/api -u test:wrongpassword
```
**Debe devolver:** `401 Unauthorized`, lo que significa que las credenciales son incorrectas.
![imagen](https://github.com/user-attachments/assets/acd7d905-10f5-403b-963d-48ae0518f057)


#### Sin credenciales
```bash
curl -v http://api.localhost/api
```
**Debe devolver:** `401 Unauthorized`, ya que la API requiere autenticación.
![imagen](https://github.com/user-attachments/assets/5b15d9d4-5b1a-4675-8a46-89861bf3cc98)


#### Ver los logs de autenticación
```bash
docker logs traefik | grep -E '^{"' | jq -c 'select(.RequestHost == "api.localhost") | [.DownstreamStatus, .ClientUsername]'
```
Este comando permite verificar los intentos de autenticación en los registros.
![imagen](https://github.com/user-attachments/assets/9c7c8a28-fd1d-4125-8e60-2b63c8d189e7)


### Desde navegador
Al acceder a ```http://api.localhost/api``` se mostrará un diálogo de autenticación:


**Explicación de la autenticación:**
- Se utiliza autenticación HTTP estándar.
- Las credenciales se envían en el header `Authorization: Basic <base64>`.
- Traefik compara los datos con la lista de usuarios configurados.
- Las contraseñas se almacenan con un hash seguro (`APR1`, variante de MD5).

## 7. Comprobar la Limitación de Velocidad

El sistema implementa restricciones para evitar el abuso de solicitudes excesivas.

### Ejecutar múltiples solicitudes rápidamente
```bash
for i in {1..15}; do
  curl -v http://nginx.localhost
  sleep 0.1
done
```
Este comando realiza 15 solicitudes en rápida sucesión para verificar la limitación de velocidad.

### Buscar respuestas `429 (Too Many Requests)`
```bash
docker logs traefik | grep -E '^{"' | jq -c 'select(.DownstreamStatus == 429)'
```
![imagen](https://github.com/user-attachments/assets/267c204d-d430-4771-b3c3-1ec24446cb95)

Esto permite detectar si el sistema ha bloqueado solicitudes por exceder el límite permitido.

### Estadísticas de Rate Limiting
```bash
docker logs traefik | grep -E '^{"' | jq -c 'select(.RequestHost == "nginx.localhost") | [.time, .DownstreamStatus]'
```
![imagen](https://github.com/user-attachments/assets/1f7e45cc-4807-4b92-bfe4-fa8849f039f2)
- Las respuestas 429 son "Too Many Requests" y se dan cuando se excede el límite del rate limiting

**Interpretación de resultados:**
- **Si no hay salida:** No se ha excedido el límite de velocidad.
- **Si hay resultados:** Cada línea mostrará un JSON con detalles de una solicitud rechazada por `rate limiting`.

Este mecanismo protege el sistema contra posibles ataques o uso excesivo de recursos.
