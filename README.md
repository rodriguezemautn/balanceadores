# 👨‍💻 Laboratorios de Balanceo de Carga con NGINX y HAProxy

Este repositorio está diseñado para la práctica de la materia Gestión Operativa y Seguridad en Redes (UTN FRLP), pero es aplicable a cualquier curso de SRE. El objetivo es implementar y comparar algoritmos de balanceo de carga estáticos (Round Robin, Weighted Round Robin, IP Hash) y dinámicos (Least Connections, Least Response Time) en un entorno de contenedores con Docker Compose. Usaremos tres backends web simples para simular un cluster de servidores, y dos balanceadores: NGINX (puerto 8080) y HAProxy (puerto 8081).

**Competencias Desarrolladas:**
- Diseño de sistemas distribuidos: Seleccionar y configurar algoritmos basados en necesidades (e.g., afinidad para sesiones, dinámica para cargas variables).
- Implementación práctica: Usar Docker para deploy reproducible.
- Monitoreo y troubleshooting: Analizar tráfico con curls, benchmarks (ab) y logs.
- Resiliencia: Simular fallos/latencia para observar adaptabilidad.
- Documentación: Registrar observaciones para informes profesionales.

## 1. Prerrequisitos (VM Rocky Linux 9)
Antes de comenzar, configura tu entorno base en una VM de Rocky Linux 9.

### 1.1. Descarga e Inicialización de la VM
1. Descarga la imagen de Rocky Linux 9 desde: https://www.linuxvmimages.com/images/rockylinux-9/
2. Importa en VirtualBox/VMWare/KVM.
3. Asegura acceso a internet (NAT/Bridged) y SSH.

### 1.2. Instalación de Docker y Docker Compose
Ejecuta estos comandos en la VM:

```bash
# Eliminar versiones antiguas (si existen)
sudo dnf remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-selinux docker-engine-selinux docker-engine

# Instalar herramientas necesarias
sudo dnf -y install dnf-utils httpd-tools

# Agregar repositorio de Docker
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# Instalar Docker y Compose
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

# Iniciar y habilitar Docker
sudo systemctl start docker
sudo systemctl enable docker

# Agregar usuario al grupo docker (cierra y abre sesión después)
sudo usermod -aG docker $USER

# Verificar
docker --version
docker compose version
```

### 1.3. Clonación del Repositorio
```bash
git clone https://github.com/tu-usuario/lb-sre-labs.git
cd lb-sre-labs
```

## 2. Comandos Base para el Entorno
| Comando | Descripción |
|---------|-------------|
| `docker compose up -d` | Inicia todos los servicios (backends, NGINX, HAProxy). |
| `docker compose down` | Detiene y elimina todo. **Úsalo después de cada lab para resetear.** |
| `curl -s localhost:8080` | Accede a NGINX (verifica con `| grep SERVER`). |
| `curl -s localhost:8081` | Accede a HAProxy. |
| `docker compose logs nginx` | Ve logs de NGINX (busca distribución con `| grep SERVER`). |
| `docker compose logs haproxy` | Ve logs de HAProxy. |

## 3. Descripción de los Backends
Tres servidores web estáticos (web1, web2, web3) que responden con "SERVER N (WEBN)". Construidos con NGINX alpine para ligereza. No exponen puertos públicos; solo accesibles vía balanceadores.

## 4. Laboratorios con NGINX (Puerto 8080)
Para cada lab, copia la config de conf/ a nginx.conf y reinicia NGINX:
```bash
cp conf/[ARCHIVO].conf ./nginx.conf
docker compose restart nginx
```

### 🧪 Lab 4.1: Round Robin (RR)
**Concepto:** Distribución cíclica igualitaria (1→2→3→1...).
**Config:** `nginx_rr.conf`
**Prueba:**
```bash
watch -n 0.5 "curl -s localhost:8080 | grep SERVER"
```
**Verificación:** Ciclo secuencial. Analiza: ¿Es predecible? ¿Bueno para cargas uniformes?

### 🧪 Lab 4.2: Round Robin Ponderado (WRR)
**Concepto:** Distribución basada en pesos (web1=3, web2=1, web3=1; ~60% a web1).
**Config:** `nginx_rr_ponderado.conf`
**Prueba:**
```bash
watch -n 0.5 "curl -s localhost:8080 | grep SERVER"
```
**Verificación:** Cuenta ~3/5 a SERVER 1. Analiza: Útil para servidores heterogéneos.

### 🧪 Lab 4.3: IP Hash (Afinidad de Sesión)
**Concepto:** Persistencia por IP cliente (mismo IP → mismo backend).
**Config:** `nginx_ip_hash.conf`
**Prueba:**
```bash
echo "--- Prueba de Afinidad ---"
for ip in 1.1.1.1 8.8.8.8 4.4.4.4 1.1.1.1; do
  echo -n "IP: $ip → "
  curl -s --header "X-Forwarded-For: $ip" localhost:8080 | grep SERVER
done
```
**Verificación:** IP repetida → mismo SERVER. Analiza: Ideal para sesiones stateful.

### 🧪 Lab 4.4: Least Connections (Concurrencia Dinámica)
**Concepto:** Envía a backend con menos conexiones activas.
**Config:** `nginx_least_conn.conf`
**Install:** `sudo dnf install -y httpd-tools`
**Prueba:**
```bash
ab -n 60 -c 10 http://localhost:8080/
docker compose logs nginx | grep "SERVER"
```
**Verificación:** Distribución adaptativa bajo carga. Analiza: Mejora rendimiento en concurrencia alta.

## 5. Laboratorio con HAProxy (Puerto 8081)
HAProxy usa Least Response Time por defecto (no requiere cambio de config).

### 🧪 Lab 5.1: Least Response Time (Latencia Dinámica)
**Concepto:** Selecciona backend con menor tiempo de respuesta promedio.
**Prueba Inicial:**
```bash
watch -n 0.5 "curl -s localhost:8081 | grep SERVER"
```
**Simular Latencia (en web3):**
```bash
# Edita index.html para simular delay (agrega sleep si usas un server dinámico; aquí aproximamos reiniciando)
echo '<h1>SERVER 3 (DELAYED)</h1><script>for(let i=0;i<1e9;i++);</script>' > backends/web3/index.html
docker compose restart web3
watch -n 0.5 "curl -s localhost:8081 | grep SERVER"
ab -n 60 -c 10 http://localhost:8081/
```
**Restaurar:**
```bash
echo '<h1>Respuesta desde: SERVER 3 (WEB3)</h1><p>Algoritmo de balanceo de carga en acción.</p>' > backends/web3/index.html
docker compose restart web3
```
**Verificación:** Menos tráfico a web3 lento. Analiza: Optimiza SLOs de latencia.

## 6. Finalización y Evaluación
- Detén siempre con `docker compose down`.
- **Tarea:** Registra observaciones en un informe (e.g., pros/contras de cada algoritmo, métricas de ab). ¿Cómo escalarías esto a producción (e.g., con Kubernetes)?
- **Extensión Avanzada:** Integra monitoreo con Prometheus o agrega health checks.