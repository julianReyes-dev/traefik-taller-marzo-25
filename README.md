# Manual de Uso

Este documento describe los pasos para clonar el repositorio, levantar los servicios con Docker y probar sus funcionalidades.

## 1. Clonar el Repositorio

```bash
# Clonar el repositorio
git clone https://github.com/julianReyes-dev/traefik-taller-marzo-25.git
```
```bash
cd traefik-taller-marzo-25
```
## 2. Configurar el archivo Hosts

Antes de ejecutar `docker-compose up`, es necesario agregar las siguientes configuraciones en el archivo `/etc/hosts`.

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

## 3. Levantar los Servicios

Ejecutar los siguientes comandos para levantar los servicios en segundo plano:

```bash
docker-compose up -d
```

## 4. Generar Tráfico

```bash
# Acceder a Nginx (público)
curl http://nginx.localhost

# Acceder a la API (con autenticación)
curl http://api.localhost/api -u test:test

# Acceder al dashboard de Traefik
curl http://traefik.localhost:8080
```

## 4. Ver Logs

### Ver todos los logs en formato JSON
```bash
docker logs traefik | grep -E '^{'
```

### Filtrar logs por servicio
```bash
docker logs traefik | grep -E '^{" | jq -c 'select(.RequestHost == "nginx.localhost")'
docker logs traefik | grep -E '^{" | jq -c 'select(.RequestHost == "api.localhost")'
```

### Formato resumido
```bash
docker logs traefik | grep -E '^{" | jq -c '[.time, .RequestHost, .RequestMethod, .RequestPath, .DownstreamStatus, .ClientUsername]'
```

## 5. Probar la Autenticación

### Prueba exitosa (usuario: test, contraseña: test)
```bash
curl -v http://api.localhost/api -u test:test
```
**Debe devolver:** `200 OK`

### Prueba fallida (credenciales incorrectas)
```bash
curl -v http://api.localhost/api -u test:wrongpassword
```
**Debe devolver:** `401 Unauthorized`

### Sin credenciales
```bash
curl -v http://api.localhost/api
```
**Debe devolver:** `401 Unauthorized`

### Ver los logs de autenticación
```bash
docker logs traefik | grep -E '^{" | jq -c 'select(.RequestHost == "api.localhost") | [.DownstreamStatus, .ClientUsername]'
```

**Método de autenticación:** HTTP estándar
- Las credenciales se envían en el header `Authorization: Basic <base64>`
- Traefik verifica contra la lista de usuarios configurada
- Las contraseñas se almacenan con hash (`APR1`, variante de MD5)

## 6. Comprobar la Limitación de Velocidad

### Ejecutar múltiples solicitudes rápidamente
```bash
for i in {1..15}; do
  curl -v http://nginx.localhost
  sleep 0.1
done
```

### Buscar respuestas `429 (Too Many Requests)`
```bash
docker logs traefik | grep -E '^{" | jq -c 'select(.DownstreamStatus == 429)'
```

### Estadísticas de Rate Limiting
```bash
docker logs traefik | grep -E '^{" | jq -c 'select(.RequestHost == "nginx.localhost") | [.time, .DownstreamStatus]'
```

**Interpretación de resultados:**
- **Si no hay salida:** No se ha excedido el límite de velocidad
- **Si hay resultados:** Cada línea mostrará un JSON con detalles de una solicitud rechazada por `rate limiting`
