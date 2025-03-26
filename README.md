# Manual de Uso

Este documento describe los pasos para clonar el repositorio, configurar el entorno y probar las funcionalidades del sistema utilizando Docker y Traefik.

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

## 4. Generar Tráfico

Para verificar que los servicios están funcionando correctamente, realice las siguientes pruebas:

### Acceder a Nginx (público)
```bash
curl http://nginx.localhost
```
Este comando verifica que el servidor web Nginx esté activo y accesible.

### Acceder a la API (con autenticación)
```bash
curl http://api.localhost/api -u test:test
```
Aquí se está accediendo a la API, que requiere autenticación básica con usuario `test` y contraseña `test`.

### Acceder al dashboard de Traefik
```bash
curl http://traefik.localhost:8080
```
Traefik proporciona un panel de administración que permite visualizar el estado de las conexiones y las reglas de enrutamiento.

## 5. Ver Logs

Para diagnosticar problemas o analizar el tráfico de los servicios, se pueden consultar los logs de Traefik.

### Ver todos los logs en formato JSON
```bash
docker logs traefik | grep -E '^{'
```
Esto muestra los registros generados por Traefik en formato JSON.

### Filtrar logs por servicio
```bash
docker logs traefik | grep -E '^{"' | jq -c 'select(.RequestHost == "nginx.localhost")'
docker logs traefik | grep -E '^{"' | jq -c 'select(.RequestHost == "api.localhost")'
```
Estos comandos filtran los registros por servicio, permitiendo analizar el tráfico de `nginx.localhost` o `api.localhost`.

### Formato resumido
```bash
docker logs traefik | grep -E '^{"' | jq -c '[.time, .RequestHost, .RequestMethod, .RequestPath, .DownstreamStatus, .ClientUsername]'
```
Este comando muestra una versión simplificada de los logs, resaltando la información más relevante.

## 6. Probar la Autenticación

El sistema utiliza autenticación HTTP básica para proteger el acceso a la API.

### Prueba exitosa (usuario: test, contraseña: test)
```bash
curl -v http://api.localhost/api -u test:test
```
**Debe devolver:** `200 OK`, lo que indica un acceso exitoso.

### Prueba fallida (credenciales incorrectas)
```bash
curl -v http://api.localhost/api -u test:wrongpassword
```
**Debe devolver:** `401 Unauthorized`, lo que significa que las credenciales son incorrectas.

### Sin credenciales
```bash
curl -v http://api.localhost/api
```
**Debe devolver:** `401 Unauthorized`, ya que la API requiere autenticación.

### Ver los logs de autenticación
```bash
docker logs traefik | grep -E '^{"' | jq -c 'select(.RequestHost == "api.localhost") | [.DownstreamStatus, .ClientUsername]'
```
Este comando permite verificar los intentos de autenticación en los registros.

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
Esto permite detectar si el sistema ha bloqueado solicitudes por exceder el límite permitido.

### Estadísticas de Rate Limiting
```bash
docker logs traefik | grep -E '^{"' | jq -c 'select(.RequestHost == "nginx.localhost") | [.time, .DownstreamStatus]'
```

**Interpretación de resultados:**
- **Si no hay salida:** No se ha excedido el límite de velocidad.
- **Si hay resultados:** Cada línea mostrará un JSON con detalles de una solicitud rechazada por `rate limiting`.

Este mecanismo protege el sistema contra posibles ataques o uso excesivo de recursos.
