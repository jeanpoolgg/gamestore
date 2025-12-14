# Docker Compose - WordPress con MySQL y phpMyAdmin

Guía completa del archivo `docker-compose.yml` para entender cómo funcionan los contenedores, redes y volúmenes.

---

## 🎯 ¿Qué hace este archivo?

Crea un entorno completo con 3 servicios conectados:
- **MySQL**: Base de datos
- **WordPress**: Sitio web
- **phpMyAdmin**: Administrador visual de la base de datos

---

## 📦 Los 3 Servicios

### 1. MySQL (Base de Datos)

```yaml
mysql:
  image: mysql:8.0
  ports:
    - "3310:3306"
  environment:
    MYSQL_ROOT_PASSWORD: wordpress
    MYSQL_DATABASE: gamestore
    MYSQL_USER: wordpress
    MYSQL_PASSWORD: wordpress
```

**¿Qué hace?**
- Usa la imagen `mysql:8.0` (versión específica)
- **Puerto**: `3310:3306` → Tu PC usa puerto 3310, dentro del contenedor es 3306
- **Credenciales**: TÚ las defines aquí. MySQL lee estas variables al iniciar y crea:
  - Usuario: `wordpress`
  - Contraseña: `wordpress`
  - Base de datos: `gamestore`
- **Volumen**: `mysql_data:/var/lib/mysql` → Guarda los datos en tu disco para que persistan

### 2. WordPress (Sitio Web)

```yaml
wordpress:
  image: wordpress:latest
  ports:
    - "8000:80"
  environment:
    WORDPRESS_DB_HOST: mysql
    WORDPRESS_DB_USER: wordpress
    WORDPRESS_DB_PASSWORD: wordpress
    WORDPRESS_DB_NAME: gamestore
  depends_on:
    - mysql
```

**¿Qué hace?**
- Usa `wordpress:latest` (última versión disponible, sin especificar número)
- **Puerto**: `8000:80` → Accedes en `http://localhost:8000`
- **Credenciales**: Usa las MISMAS que definiste en MySQL para conectarse
- **WORDPRESS_DB_HOST: mysql** → Usa el nombre del servicio como hostname (DNS interno de Docker)
- **depends_on**: Espera a que MySQL esté listo antes de arrancar

### 3. phpMyAdmin (Administrador de BD)

```yaml
phpmyadmin:
  image: phpmyadmin/phpmyadmin
  ports:
    - "8181:80"
  environment:
    PMA_HOST: mysql
```

**¿Qué hace?**
- Sin versión especificada = usa `:latest` automáticamente
- **Puerto**: `8181:80` → Accedes en `http://localhost:8181`
- **PMA_HOST: mysql** → Se conecta al servicio MySQL usando su nombre

---

## 🔌 Puertos - ¿Por qué dos veces el puerto 80?

```yaml
wordpress:
  ports:
    - "8000:80"    # ← Puerto 80 DENTRO del contenedor WordPress

phpmyadmin:
  ports:
    - "8181:80"    # ← Puerto 80 DENTRO del contenedor phpMyAdmin
```

**NO hay conflicto** porque:
- Cada contenedor es una "computadora" independiente
- `8000:80` = Tu PC (puerto 8000) → Contenedor WordPress (puerto 80 interno)
- `8181:80` = Tu PC (puerto 8181) → Contenedor phpMyAdmin (puerto 80 interno)

Ambos tienen puerto 80 interno, pero Docker los mapea a puertos distintos en tu máquina.

---

## 🔄 restart: always

```yaml
restart: always
```

Si el contenedor se cae o crashea, Docker lo reinicia automáticamente.

**Otras opciones**:
- `no` → No reinicia
- `on-failure` → Solo si falla
- `unless-stopped` → Siempre, excepto si lo detuviste manualmente

---

## 🔗 Redes - ¿Por qué "mysql" como host?

```yaml
networks:
  - wpnet
```

Todos los servicios están en la red `wpnet`. Docker crea un **DNS interno** donde los contenedores se conocen por su nombre de servicio:

```yaml
services:
  mysql:      # ← WordPress puede conectarse haciendo "mysql:3306"
  wordpress:  # ← phpMyAdmin puede conectarse haciendo "mysql:3306"
```

Es como si cada servicio tuviera un nombre de dominio dentro de Docker.

---

## 💾 Volúmenes - Sincronización de archivos

### Tipo 1: Directorio (bidireccional)

```yaml
volumes:
  - ./.srv/wordpress:/var/www/html
```

**Comportamiento**:
1. Si `.srv/wordpress` está vacío → Docker **copia todo desde el contenedor a tu PC**
2. Luego es **bidireccional**:
   - Editas en tu PC → Se ve en el contenedor
   - WordPress crea archivos → Aparecen en tu PC

**Contiene**: Todos los archivos PHP de WordPress (themes, plugins, wp-config.php, etc.)

### Tipo 2: Archivo específico

```yaml
volumes:
  - ./.srv/custom.ini:/usr/local/etc/php/conf.d/custom.ini
```

**Comportamiento**:
- Si `.srv/custom.ini` NO existe → **ERROR, contenedor no arranca**
- Si existe → Lo monta y reemplaza el del contenedor

**Debes crearlo antes** con configuraciones PHP:
```ini
upload_max_filesize = 64M
post_max_size = 64M
memory_limit = 256M
```

### Volumen nombrado (persistente)

```yaml
volumes:
  mysql_data:
```

Docker crea un espacio en tu disco para guardar datos de MySQL. No lo ves fácilmente, pero persiste aunque borres los contenedores.

---

## 🔐 Flujo de Credenciales

1. **Defines** las credenciales en el servicio `mysql`
2. **MySQL lee** esas variables y crea el usuario/BD al iniciar
3. **WordPress usa** esas mismas credenciales para conectarse
4. **phpMyAdmin usa** las mismas para administrar

Todo debe coincidir o no funcionará.

---

## 🚀 Comandos Útiles

```bash
# Iniciar todos los servicios
docker-compose up -d

# Ver logs
docker-compose logs -f

# Detener todo
docker-compose down

# Detener y borrar volúmenes (¡cuidado, pierdes datos!)
docker-compose down -v

# Ver servicios corriendo
docker-compose ps
```

---

## 🌐 Accesos

- **WordPress**: http://localhost:8000
- **phpMyAdmin**: http://localhost:8181
  - Usuario: `wordpress`
  - Contraseña: `wordpress`
- **MySQL** (desde terminal): `mysql -h 127.0.0.1 -P 3310 -u wordpress -p`

---

## 📝 Notas Importantes

- `latest` como versión = última disponible (puede cambiar en el futuro)
- Los archivos específicos en volúmenes DEBEN existir antes de iniciar
- Los directorios vacíos se llenan automáticamente desde el contenedor
- Las credenciales aquí son de ejemplo, **cámbialas en producción**
- `depends_on` solo espera que el contenedor inicie, no que esté 100% listo

---

## 🐛 Troubleshooting

**Error: "bind source path does not exist"**
→ Crea el archivo que falta o comenta esa línea del volumen

**WordPress no se conecta a MySQL**
→ Verifica que las credenciales coincidan exactamente

**Puerto ya en uso**
→ Cambia el puerto externo (ej: `8001:80` en vez de `8000:80`)

---

¡Listo! Ahora entiendes cómo funciona cada parte del Docker Compose. 🎉