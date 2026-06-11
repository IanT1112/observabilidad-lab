# Observabilidad

## Preguntas

**1. ¿Por qué necesitamos Loki además de Prometheus si ya tenemos /metrics?**  
Loki sirve par almacenar logs, prometheus solo guarda métricas. Requerimos de ambos para tener mejor contexto de lo que sucede detrás, ya que, las métricas te dicen que falló y los logs te dicen el porque falló

**2. ¿Qué ventaja aporta que las fuentes de datos de Grafana estén aprovisionadas como código y no creadas a mano?**  
De que si el entorno se replica en otra máquina, las fuentes de datos aparecen solas al levantar el stack, ya que están definidas en un archivo versionado en Git

**3. El panel "CPU contenedor" y el panel "CPU host" pueden mostrar valores muy distintos. ¿Por qué? ¿Cuál usarías para alertar sobre una aplicación concreta?**  
CPU contenedor: Mide los recursos utilizados exclusivamente por la aplicación  
CPU host: Muestra la carga agregada de todas las máquinas virtuales o contenedores  
Los valores distintos son porque el CPU del contenedor suele medirse como el porcentaje del límite asignado, mientras que el CPU del host mide el porcentaje sobre el total físico o virtual de la máquina. Para alertar sobre una aplicación usaría CPU del contenedor, para saber si la aplicación se va a caer

**4. ¿Qué diferencia hay entre el evaluation interval y el pending period de una alarma?**  
Evaluation interval (intervalo de evaluación) define cada cuánto tiempo la alarma revisa si se cumple la condición, mientras que el pending period (periodo de espera) determina cuánto tiempo debe mantenerse esa condición activa antes de disparar la notificación

---

## Validación

**1. Levantar el stack**
```bash
docker compose up -d --build
docker compose ps
```
Todos los servicios deben aparecer en estado `Up`.

**2. Verificar servicios**

| Servicio | URL |
|---|---|
| Frontend | http://localhost:8080 |
| Backend métricas | http://localhost:3001/metrics |
| Grafana | http://localhost:3000 |
| Prometheus | http://localhost:9090 |
| Alloy | http://localhost:12345 |

**3. Verificar targets en Prometheus**  
Ir a http://localhost:9090/targets y confirmar que todos los targets están en estado `UP`.

**4. Verificar dashboards en Grafana**  
Entrar con `admin` / `admin`, ir a Dashboards y abrir el dashboard creado. Los 4 paneles deben mostrar datos.

> **Nota:** Las consultas PromQL usan `id="/docker"` en lugar de `name="lab-backend"` porque en Windows, cAdvisor identifica los contenedores por ID y no por nombre. En Linux funcionaría con `name="lab-backend"`.

**5. Probar la alarma**  
Abrir http://localhost:8080 y pulsar **"Generar carga de CPU (30s)"**. En **Alerting → Alert rules** la regla debe pasar de `Normal` → `Pending` → `Firing`.

**6. Verificar el ciclo alarma → log**  
En el panel **"Logs de aplicación"** filtrar con:

```logql
{tier="application"} | json | msg="grafana_alert_received"
```

Debe aparecer un log con `alert_status: firing`.

---

## Explicación de componentes del stack

**Prometheus:** Guarda métricas sobre el estado de servidores, bd y microservicios. Las consulta cada 5 segundos desde los servicios.

**Node-exporter:** Herramienta de Prometheus. Usado para supervisar y exponer el estado de los servidores en un formato que Prometheus pueda leer fácilmente.

**cAdvisor:** Monitorea rendimiento y uso de recursos de contenedores en tiempo real, para permitir a los administradores y desarrolladores entender cómo interactúan los contenedores con los recursos del sistema.

**Loki:** Conocido como sistema de agregación de registros (logs). Se usa para recopilar, almacenar y con ello analizar mensajes generados por servidores, aplicaciones o servidores.

**Grafana Alloy:** Envía datos sobre el rendimiento de aplicaciones y servidores hacia plataformas de análisis como Grafana, Prometheus o Loki.

**Grafana:** La interfaz visual, permite convertir métricas, registros y eventos en dashboards, en este laboratorio se usó también su función de alertas.

---

## Secuencia de desarrollo

Una vez que tenemos los archivos bases del laboratorio, usamos `docker compose up -d --build` y `docker compose ps`. Me percaté de un archivo llamado .DS_Store (que al buscar era un archivo generado por las propias laptops macOS).

Abrí cada puerto para verificar que todo esté corriendo correctamente y así era, entré a Grafana con user y password admin/admin y generamos tráfico con "Saludar(API)".

Verificamos fuentes de datos en Data Sources para verificar que Prometheus y Loki aparecen en el estado correcto y empezamos a construir los dashboards requeridos.

- CPU contenedor backend (%) - Fuente: Prometheus
- CPU del host (%) - Fuente: Prometheus
- Logs de aplicación (API + frontend) - Fuente: Loki
- Logs de infraestructura - Fuente: Loki

Configuración de alarma de CPU>50% y probamos al hacer click sobre el botón del frontend "Generar carga de CPU (30s)" y verificamos que la alarma pase de Normal -> Pending -> Firing.

Configuración de un contact point de tipo Webhook y llamado así. Registrándose el evento como log y visualizándose en el panel "Logs de aplicación".

---

## Comentarios

Se siguieron los pasos señalados en la documentación, no se hicieron cambios más que en el código base proporcionado, se generaron los dashboards correspondientes, me topé con errores en el camino, se pudieron resolver y funcionó.

### Problemas que se tuvieron

**Grafana no permitía realizar dashboard con Loki**, salía error en la parte de consulta de query. Solución: Se cambió la versión de Grafana.

**Estado "caído" de node-exporter en el target de Prometheus.** Solución: Se comentaron algunas líneas del código inicial, debido a que Windows no permitía ese tipo de acceso desde Docker, por ello el contenedor fallaba y no se levantaba. En lugar de ver la CPU real de mi laptop, veía la VM que usa Docker Desktop. Valores distintos, pero mismo comportamiento.

**Las consultas PromQL usaban name="lab-backend" pero no devolvían datos**. Solución: Se cambió el filtro a id="/docker". Debido a que, en Linux cAdvisor identifica los contenedores por nombre (name="lab-backend"). En Windows los identifica por ID (id="/docker"). Por eso se cambió la consulta, para usar la forma en que Windows expone esa información.
