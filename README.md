
### `# UTN FRLP - Laboratorios de Balanceo de Carga (SRE)`

Este repositorio contiene el entorno de práctica para la Sesión de Balanceo de Carga, implementando y comparando algoritmos estáticos y dinámicos en **NGINX** y **HAProxy**.

### 1\. Configuración del Entorno

Asegúrese de haber completado los pasos de instalación de Docker en su VM de Rocky Linux.

**Clonar el Repositorio:**

```bash
git clone https://github.com/tu-usuario/lb-sre-labs.git 
cd lb-sre-labs
```

**Comandos Base:**

| Comando | Descripción |
| :--- | :--- |
| `docker compose up -d` | Inicia todos los servicios (backends y balanceadores). |
| `docker compose down` | Detiene y elimina todos los contenedores. **Usar después de cada práctica.** |

### 2\. Backends de Aplicación

Todos los laboratorios usan tres servidores web (`web1`, `web2`, `web3`) que exponen el nombre del servidor para visualizar el balanceo.

  * **Contenido:** `backends/webN/index.html` (Muestra **`SERVER N`**).
  * **Servicio:** Definido en `docker-compose.yml`.

### 3\. Laboratorios con NGINX (Puerto 8080)

Para ejecutar cada laboratorio de NGINX, debe reemplazar el archivo de configuración activo:

```bash
# PASO 1: Reemplazar la configuración (ej: Round Robin)
sudo cp conf/nginx_rr.conf ./nginx.conf

# PASO 2: Reiniciar el contenedor NGINX para aplicar la nueva conf
docker compose restart nginx

# PASO 3: Ejecutar la prueba (usar el comando de prueba correspondiente)
```

#### 🧪 Lab 3.1: Round Robin (RR)

  * **Configuración:** `conf/nginx_rr.conf`
  * **Prueba (Verificación Cíclica):**

<!-- end list -->

```bash
watch -n 0.5 "curl -s localhost:8080 | grep SERVER"
# Verificación: El resultado debe ser 1 -> 2 -> 3 -> 1 -> ...
```

#### 🧪 Lab 3.2: Round Robin Ponderado (WRR)

  * **Configuración:** `conf/nginx_rr_ponderado.conf`
  * **Pesos:** `web1` (weight=3), `web2` (weight=1), `web3` (weight=1).
  * **Prueba (Verificación de Pesos):**

<!-- end list -->

```bash
watch -n 0.5 "curl -s localhost:8080 | grep SERVER"
# Verificación: De cada 5 peticiones, ~3 deben ir al SERVER 1.
```

#### 🧪 Lab 3.3: IP Hash (Afinidad de Sesión)

  * **Configuración:** `conf/nginx_ip_hash.conf`
  * **Prueba (Simulación de IPs):**
    Utilizamos la cabecera `X-Forwarded-For` para simular diferentes IPs de cliente.

<!-- end list -->

```bash
echo "--- Prueba de Persistencia de Sesión (Hash por IP) ---"
for ip in 1.1.1.1 8.8.8.8 4.4.4.4 1.1.1.1; do
  echo -n "$ip -> "
  curl -s --header "X-Forwarded-For: $ip" localhost:8080 | grep SERVER
done
# Verificación: El IP 1.1.1.1 debe ser enrutado al mismo SERVER cada vez.
```

#### 🧪 Lab 3.4: Least Connections (Concurrencia Dinámica)

  * **Configuración:** `conf/nginx_least_conn.conf`
  * **Prueba (Simulación de Carga Pesada):**
    Usaremos `ab` (Apache Benchmark - si no está instalado, `sudo dnf install httpd-tools -y`) para simular múltiples conexiones.

<!-- end list -->

```bash
# PASO 1: Ejecutar 60 peticiones concurrentes (C=10)
# Esto estresará el balanceador, haciendo que Least Connections redistribuya.
ab -n 60 -c 10 http://localhost:8080/

# PASO 2: Revisar los logs del balanceador (para ver la distribución)
docker compose logs nginx | grep "SERVER" 
# La distribución de solicitudes mostrará que los servidores con menos carga activa recibieron más solicitudes.
```

### 4\. Laboratorio con HAProxy (Puerto 8081)

**Nota:** HAProxy está preconfigurado para usar el algoritmo Least Response Time.

#### 🧪 Lab 4.1: Least Response Time (Latencia Dinámica)

  * **Configuración:** `conf/haproxy_least_resp.cfg`
  * **Prueba (Medición de Latencia):**
    Simularemos que el `web3` es más lento (agregando un *delay* de 1 segundo en su `index.html`).

<!-- end list -->

1.  **Hacer Lento el `web3` (Ejemplo avanzado):**

    ```bash
    # Agregar un delay de 1 segundo solo a web3 (para simular baja performance)
    echo '<p>SERVER 3 (DELAYED)</p>' > backends/web3/index.html
    docker compose restart web3
    ```

2.  **Prueba de Balanceo (Verificación de Distribución):**

    ```bash
    watch -n 0.5 "curl -s localhost:8081 | grep SERVER"
    # Verificación: HAProxy enviará la mayoría del tráfico a web1 y web2, priorizando el menor tiempo de respuesta, hasta que la carga se iguale.
    ```

3.  **Restaurar `web3`:**

    ```bash
    echo '<p>SERVER 3 (WEB3)</p>' > backends/web3/index.html
    docker compose restart web3
    ```

### 5\. Finalización del Laboratorio

**Recuerde siempre detener el entorno de contenedores al finalizar:**

```bash
docker compose down
```
