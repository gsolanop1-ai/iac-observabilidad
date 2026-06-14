# Informe — Laboratorio de Observabilidad (Grafana + Prometheus + Loki)

**Curso:** Infraestructura como Código
**Estudiante:** Solano Pérez Gabriel
**Fecha:** 2026-06-13

---

## 0. Resumen ejecutivo

Este documento describe el desarrollo técnico del laboratorio de observabilidad: qué se hizo en el repositorio, qué errores se encontraron y por qué se tomaron las decisiones que se tomaron. La intención es complementar el informe formal (con capturas) con el detalle de implementación, los cambios versionados en el código y la justificación de las desviaciones respecto a la guía.

Decisiones tomadas que se desvían intencionalmente de la guía y se justifican explícitamente:

1. **Umbral de alarma a 30%** (no 50%) — respondiendo a la duda del docente sobre si el umbral debía ser obligatoriamente 50%.
2. **Query de CPU del contenedor sustituida** por la métrica del proceso Node.js (`backend_process_cpu_seconds_total`) por fallo de cAdvisor en Docker Desktop.
3. **Dos dashboards en vez de uno** — atendiendo la indicación del docente "no solo 1 dashboard".

---

## 1. Stack levantado y todos los servicios accesibles

El stack se levantó con:

```bash
docker compose up -d --build
```

Tras corregir el bug del `rslave` (sección 4.2), los 8 contenedores arrancaron correctamente. Verificación realizada vía `docker compose ps` y consulta a la API de Prometheus (`/api/v1/targets`) confirmando que los 5 targets están `up`.

**Servicios accesibles confirmados:**

| Servicio | URL | Resultado |
|---|---|---|
| Frontend | http://localhost:8080 | 200 OK (Hello World) |
| Backend `/metrics` | http://localhost:3001/metrics | 200 OK (formato Prometheus) |
| Grafana | http://localhost:3000 | 200 OK (login admin/admin) |
| Prometheus | http://localhost:9090 | 200 OK |
| Loki `/ready` | http://localhost:3100/ready | 200 OK |
| Alloy | http://localhost:12345 | 200 OK |
| cAdvisor | http://localhost:8081 | 200 OK (con caveat — ver §4.4) |
| node-exporter | http://localhost:9100/metrics | 200 OK |

---

## 2. Generación de tráfico y validación de pipeline

Tras levantar el stack se generó tráfico desde el navegador (10 clics en "Saludar (API)"), validado consultando el contador en Prometheus:

```promql
http_requests_total{route="/api/hello"}
```

Resultado: 10 peticiones en `job=backend` y 10 en `job=frontend` — coincide exactamente con los clics manuales. Esto demostró que el pipeline `app → /metrics → Prometheus` funciona end-to-end.

Paralelamente, los logs en Loki contenían 10 eventos `saludo_generado` del backend, confirmando que el pipeline `app stdout → Alloy → Loki` también funciona.

---

## 3. Dashboard principal — "Observabilidad"

Dashboard creado con 4 paneles según especificación de la guía. **Ningún dashboard fue importado desde JSON** — todos los paneles se construyeron a mano para demostrar comprensión.

### 3.1 Panel "CPU proceso backend (%)"

**Query final (modificada respecto a la guía):**

```promql
rate(backend_process_cpu_seconds_total[1m]) * 100
```

**Justificación del cambio:** la query original de la guía (`sum(rate(container_cpu_usage_seconds_total{name="lab-backend"}[1m])) * 100`) depende de cAdvisor, que en Docker Desktop **no logra exponer métricas por contenedor** (ver §4.4). Por tanto se sustituyó por la métrica que el propio backend expone vía `prom-client`. Mide la CPU del proceso Node.js dentro del contenedor — en este lab es equivalente, ya que cada contenedor corre un único proceso de aplicación.

**Threshold:** configurado a `50` con color rojo, como pide la guía. Durante el estado idle (línea base ~0.4%) el threshold queda fuera del eje Y visible; durante el test de carga (§6) la línea cruza el threshold y es claramente visible porque el worker thread del backend satura un core completo (~100%).

### 3.2 Panel "CPU del host (%)"

**Query:**

```promql
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)
```

**Observación:** el valor reportado oscila entre 0.7% y 13% en momentos de carga, lo cual es bajo en términos absolutos. **Esto se debe a que node-exporter, corriendo dentro de Docker Desktop, monitoriza el VM de WSL2, no el host Windows real.** Es un matiz importante sobre dónde está montado el exporter (ver respuesta a la Pregunta 3).

### 3.3 Panel "Logs de aplicación (API + frontend)"

**Query base:**

```logql
{tier="application"} | json
```

**Query con filtro por nivel:**

```logql
{tier="application"} | json | level="ERROR"
```

El parser `| json` extrae los campos JSON estructurados del log (level, service, msg, order_id, user_id, etc.), permitiendo filtrar después por cualquiera de ellos. Esto demuestra el valor de emitir logs en formato estructurado desde la aplicación: los chips de filtro aparecen automáticamente en la UI de Grafana, sin configuración adicional.

### 3.4 Panel "Logs de infraestructura"

**Query:**

```logql
{tier="infrastructure"}
```

**Sin el parser `| json`** — los logs de Loki, Prometheus, Grafana, Alloy, node-exporter y cAdvisor no son JSON sino texto plano con sus propios formatos. Aplicar `| json` produciría errores de parseo.

**Hallazgo importante en este panel:** son visibles los logs de cAdvisor reportando errores continuos del tipo:

```
E0613 22:03:57.506979 1 manager.go:1116] Failed to create existing container:
/docker/<hash>: failed to identify the read-write layer ID for container ...
open /rootfs/var/lib/docker/image/overlayfs/layerdb/mounts/<hash>/mount-id:
no such file or directory
```

Esto confirma visualmente el bug detectado en el análisis estático (ver §4.4) y refuerza por qué se pivotó a la métrica del proceso Node.js.

---

## 4. Errores encontrados en el proyecto entregado

Durante el análisis estático y la ejecución se identificaron **cinco bugs**. Tres requirieron corrección en archivos versionados; dos son comportamientos derivados de la plataforma (Docker Desktop en Windows) que no se corrigen pero sí se documentan.

### 4.1 Bug — Loki sin volumen persistente (intencional, severidad alta)

**Archivo:** `docker-compose.yml`

El servicio `loki` no declaraba volumen para `/loki/chunks`, donde Loki almacena los chunks de logs. Mientras `prometheus-data` y `grafana-data` sí tenían volúmenes nombrados, Loki escribía en el filesystem efímero del contenedor. **Cualquier `docker compose restart loki` o recreación del contenedor implicaría perder todo el histórico de logs**, dejando los paneles de logs en blanco.

**Fix aplicado:** se añadió volumen nombrado `loki-data:/loki` al servicio y se declaró en la sección `volumes:` del compose.

```yaml
loki:
  ...
  volumes:
    - ./loki/loki-config.yaml:/etc/loki/loki-config.yaml:ro
    - loki-data:/loki      # <-- añadido
...
volumes:
  prometheus-data:
  grafana-data:
  loki-data:               # <-- añadido
```

### 4.2 Bug — `node-exporter` con propagación `rslave` (de plataforma, severidad alta)

**Archivo:** `docker-compose.yml`

El mount `/:/host:ro,rslave` del servicio `node-exporter` falla en Docker Desktop con el error:

> `Error response from daemon: path / is mounted on / but it is not a shared or slave mount`

El flag `rslave` requiere que el host tenga propagación de mounts compartida, lo cual no expone el VM de Docker Desktop. En Linux nativo funcionaría sin problema. **Como consecuencia, en el primer intento de levantar el stack, el `docker compose up` se detuvo a mitad de camino y dejó alloy, frontend y grafana en estado `Created` sin arrancar.**

**Fix aplicado:** se eliminó el flag `rslave`, quedando solo `:ro`. Trade-off: node-exporter no detecta automáticamente nuevos mountpoints añadidos en caliente — irrelevante para el lab.

### 4.3 Bug — Retención de Loki declarada pero no aplicada (de configuración, severidad media)

**Archivo:** `loki/loki-config.yaml`

El bloque `limits_config.retention_period: 168h` define una retención de 7 días, pero **para que Loki ejecute la eliminación real se necesita activar el `compactor` con `retention_enabled: true`**. Sin esa configuración, la línea de retención es ignorada y los logs se acumulan indefinidamente.

**Fix aplicado:** se añadió el bloque `compactor`:

```yaml
compactor:
  working_directory: /loki/compactor
  retention_enabled: true
  delete_request_store: filesystem
```

### 4.4 Bug — cAdvisor no expone métricas por contenedor en Docker Desktop (de plataforma, severidad alta)

cAdvisor (`gcr.io/cadvisor/cadvisor:v0.49.1`) arranca correctamente y expone `/metrics`, pero sus logs muestran fallos masivos al intentar leer la metadata de cada contenedor del host:

> `Failed to identify the read-write layer ID for container "<hash>". - open /rootfs/var/lib/docker/image/overlayfs/layerdb/mounts/<hash>/mount-id: no such file or directory`

Esto es una incompatibilidad entre cAdvisor y la forma en que Docker Desktop almacena los layers de los contenedores en el VM de WSL2 (path `overlay2` distinto de `overlayfs/layerdb`).

**Consecuencia:** cAdvisor solo expone métricas a nivel de cgroups raíz (`/`, `/docker`, `/docker/buildkit`), pero **NO por contenedor individual**. La query original de la guía `container_cpu_usage_seconds_total{name="lab-backend"}` devuelve siempre 0 resultados. Esto se verificó directamente contra la API de Prometheus.

**Fix aplicado:** se pivota la query del Panel 8.1 a `rate(backend_process_cpu_seconds_total[1m]) * 100` (métrica que el propio backend expone vía `prom-client`). Mide CPU del proceso, equivalente al contenedor en este lab. **El fallo de cAdvisor sigue siendo observable en el Panel 8.4 (logs de infraestructura) como evidencia del bug detectado.**

### 4.5 Bug menor — discrepancia entre guía y código sobre dónde aparece el log del webhook

La guía dice (Paso 11):

> *"el backend registrará un log de la alerta, que volverá a aparecer en tu panel **Logs de infraestructura**"*

Sin embargo, el archivo `alloy/config.alloy` marca explícitamente al contenedor `lab-backend` con `tier="application"`:

```alloy
rule {
  source_labels = ["__meta_docker_container_name"]
  regex         = "/(lab-backend|lab-frontend)"
  target_label  = "tier"
  replacement   = "application"
}
```

Por tanto, el log del webhook (`grafana_alert_received`) aparece en el panel **Logs de aplicación**, no en infraestructura. La guía está desactualizada o tiene este error intencional para que el estudiante verifique los criterios reales del tagging.

**No se modifica el código** — se documenta la discrepancia: el log se busca en el panel 8.3, no en el 8.4.

---

## 5. Alarma de CPU configurada

**Nombre:** `CPU backend > 30%`
**Query:** `rate(backend_process_cpu_seconds_total[1m]) * 100`
**Threshold:** `IS ABOVE 30`
**Evaluation interval:** `10s`
**Pending period:** `30s`
**Labels:** `severity=warning`
**Contact point:** webhook a `http://backend:3001/alerts`

### Justificación del umbral (responde directamente la duda del docente)

El docente planteó la duda de si el umbral debía ser obligatoriamente 50% o podía ser otro porcentaje. **El umbral correcto depende del baseline del sistema y la sensibilidad deseada.** El perfil de este sistema es:

| Estado | CPU del proceso backend |
|---|---|
| Idle (sin carga) | ~0.4% |
| Bajo carga del worker thread | ~100% (un core entero) |

Cualquier umbral entre 10% y 80% es defendible:

| Umbral | Detecta | Caso de uso |
|---|---|---|
| 20% | Cualquier desviación del baseline | Producción con tráfico real predecible — detección temprana de anomalías |
| 30% (elegido) | CPU pressure clara | Balance entre reactividad y falsos positivos |
| 50% (sugerido por guía) | CPU pressure obvio | Lo que sugiere la guía como ejemplo |
| 80% | Saturación inminente | Solo cuando ya es crítico |

Se eligió **30% como demostración de criterio**, no por copiar el valor de la guía. Conceptualmente la alarma cumple la misma función: detectar uso anormal de CPU que sostenido durante 30s indica problema real.

---

## 6. Evidencia de la alarma en estado Firing

Se generó carga pulsando el botón "Generar carga de CPU (30s)" del frontend. Cronología observada:

| Tiempo | Estado de la alarma | Valor CPU panel |
|---|---|---|
| T+0s | Normal | ~0.4% |
| T+10s | Pending (chip amarillo) | subiendo a ~80% |
| T+40s | **Firing** (chip rojo) | ~100-150% (un core saturado) |
| T+60s | Aún Firing | comenzando a bajar |
| T+90s | Vuelve a Normal | ~0.4% |

El panel 8.1 mostró claramente la línea verde cruzando el threshold rojo de 50%.

---

## 7. Ciclo cerrado alarma → log vía webhook

El Contact Point `webhook-backend` fue configurado con URL `http://backend:3001/alerts`. Cuando la alarma pasó a Firing, Grafana POSTeó el payload de alerta al backend, que lo procesó en el handler de [apps/backend/server.js:106-116](apps/backend/server.js#L106-L116) y emitió el log:

```json
{
  "timestamp": "2026-06-14T00:03:30.019Z",
  "level": "ERROR",
  "service": "backend",
  "msg": "grafana_alert_received",
  "alert_status": "firing",
  "alert_count": 1,
  "alertname": "CPU backend > 30%"
}
```

Este log fue capturado por Alloy y enviado a Loki con `tier="application"`. Por tanto **aparece en el Panel 8.3 (Logs de aplicación)**, no en el Panel 8.4 como sugiere erróneamente la guía (ver §4.5).

El ciclo cerrado completo demostrado:

```
métrica de CPU sube
   ↓
Prometheus la scrappea cada 5s
   ↓
Grafana evalúa la regla cada 10s
   ↓
Threshold cruzado 30s sostenido → Firing
   ↓
Grafana POST → http://backend:3001/alerts
   ↓
Backend logea "grafana_alert_received"
   ↓
Alloy lo recoge de stdout
   ↓
Loki lo indexa con tier="application"
   ↓
Panel 8.3 lo muestra en Grafana
```

---

## 8. Dashboard secundario — "Tráfico y errores HTTP"

Se creó un **segundo dashboard** atendiendo la indicación del docente "no solo 1 dashboard". Razonamiento: separar dashboards por propósito mejora la legibilidad.

| Dashboard | Propósito | Audiencia ideal |
|---|---|---|
| **Observabilidad** (principal) | Vista de salud del sistema (CPU, logs por capa) | Operador/SRE buscando diagnóstico rápido |
| **Tráfico y errores HTTP** (secundario) | Vista de comportamiento del producto (tráfico, errores aplicativos) | Producto/QA buscando entender uso real |

**Panel "Tasa de peticiones HTTP por servicio":**

```promql
sum by (job) (rate(http_requests_total[1m]))
```

Demuestra el uso de **agregación por label** (`by`), técnica fundamental de PromQL no presente en el dashboard principal.

**Panel "Errores recientes":**

```logql
{tier="application"} | json | level="ERROR"
```

Concentra todos los errores aplicativos del backend, frontend e incluso de subsistemas internos como `service="inventory"` (logueado desde el backend cuando simula fallo del microservicio de inventario downstream).

---

## 9. Respuestas a las preguntas de la guía

### 9.1 ¿Por qué necesitamos Loki además de Prometheus si ya tenemos `/metrics`?

Porque **Prometheus solo almacena números en el tiempo**, no texto ni contexto. Una métrica `http_requests_total{status=500}` te dice *cuántos* errores hubo, pero no *qué* error, *qué* usuario, *qué* orden lo causó.

Loki almacena las **líneas de log completas** con su contexto estructurado (`order_id=4521`, `user_id=132`, `trace_id=a8f3e2d1`, `gateway=stripe`). Esa información es imposible de representar como serie temporal porque introduciría una explosión de cardinalidad en Prometheus (un nuevo label por cada combinación posible de valores).

**Conclusión:** Prometheus responde *cuántos*; Loki responde *qué exactamente*. Son complementarios, no sustitutos.

### 9.2 ¿Qué ventaja aporta que las fuentes de datos de Grafana estén aprovisionadas como código?

Tres ventajas concretas demostradas en este lab:

1. **Reproducibilidad:** levantar el stack en otra máquina produce la misma configuración exacta sin acciones manuales — visible en `grafana/provisioning/datasources/datasources.yml`.
2. **Versionable:** la configuración vive en Git. Cambios revisables vía PR, retrocedibles con `git revert`.
3. **No depende de que un humano recuerde hacer clic.** El día que se rota una credencial o se reinstala Grafana, las datasources se recrean automáticamente sin sesgo de memoria.

Esto es la esencia de "Infraestructura como Código": **la configuración de la infraestructura tiene el mismo nivel de rigor (versionada, revisable, reproducible) que el código de la aplicación.**

### 9.3 Panel "CPU contenedor" vs Panel "CPU host" — ¿por qué son distintos? ¿Cuál usar para alertar?

**Por qué son distintos:**

- **CPU contenedor** (panel 8.1) mide únicamente el proceso del backend Node.js (~0.4% en idle, ~100% bajo carga).
- **CPU host** (panel 8.2) mide el VM de WSL2 entero (~0.7-13%), que incluye TODOS los contenedores del stack además de cualquier otro proceso del VM.

Pueden mostrar valores muy distintos porque miden universos distintos: uno es un proceso, el otro es un sistema completo con N procesos.

**Cuál usar para alertar sobre una aplicación concreta:**

El de **contenedor (o proceso)**. Razón: si quiero saber si el backend tiene problemas de CPU, alertar sobre el host me daría falsos positivos (otro proceso del host puede consumir CPU sin que mi app tenga problema) y también falsos negativos (mi backend puede estar saturado mientras el host todavía tiene CPU libre por los otros cores).

**Caveat encontrado en este lab:** el `host` aquí es realmente el VM de WSL2, no el host Windows. Esto refuerza que **observabilidad depende críticamente de dónde está montado el exporter**.

### 9.4 Diferencia entre *evaluation interval* y *pending period*

| Concepto | Significado | Valor en este lab |
|---|---|---|
| **Evaluation interval** | Cada cuánto Grafana re-ejecuta la query de la regla y compara con el threshold | `10s` |
| **Pending period** | Tiempo que la condición debe mantenerse cumplida antes de pasar a Firing | `30s` |

**Ejemplo aplicado:** con `evaluation=10s` y `pending=30s`, si la CPU cruza el 30% en T+0, Grafana lo detecta en T+10 (siguiente eval). La regla pasa a `Pending`. Si en T+20, T+30 y T+40 la condición sigue cumpliéndose (tres evaluaciones consecutivas durante 30s), entonces pasa a `Firing`. Si en T+15 la CPU baja de 30%, vuelve a Normal sin haber disparado nada.

**Por qué importa esta separación:**

- *Evaluation interval* muy alto (ej. 1 min) → la alarma reacciona lento.
- *Evaluation interval* muy bajo (ej. 1s) → genera carga innecesaria.
- *Pending period* en 0 → cualquier pico transitorio (un scrape pesado, una métrica con jitter) dispara la alarma. **Falsos positivos.**
- *Pending period* muy alto → la alarma llega tarde, cuando el incidente ya escaló.

El balance `10s/30s` evita falsos positivos sin sacrificar reactividad.

---

## 10. Cambios versionados en el repositorio

Resumen de los archivos modificados durante el desarrollo del lab y por qué:

| Archivo | Cambio | Motivo |
|---|---|---|
| `docker-compose.yml` | Añadido volumen `loki-data:/loki` y declaración en `volumes:` | Fix bug §4.1 (Loki sin persistencia) |
| `docker-compose.yml` | Eliminado `rslave` del mount de node-exporter | Fix bug §4.2 (incompatibilidad Docker Desktop) |
| `loki/loki-config.yaml` | Añadido bloque `compactor` con `retention_enabled: true` | Fix bug §4.3 (retención no aplicada) |
| `.gitignore` | Añadido `.DS_Store` y `**/.DS_Store` | Limpieza de archivos macOS no necesarios |
| `.DS_Store` (×3) | Eliminados del repo | Limpieza |
| `informe.md` | Creado | Este documento |

Validación: `docker compose config --quiet` pasa sin errores en cada cambio.

---

## 11. Cierre

Los entregables del rubro se cumplieron al 100%. Adicionalmente:

- Se diagnosticaron y corrigieron 3 bugs en archivos versionados (`docker-compose.yml`, `loki-config.yaml`).
- Se documentaron 2 bugs derivados de la plataforma (cAdvisor en Docker Desktop, propagación de mounts).
- Se atendió la duda del docente sobre el umbral de alarma con justificación técnica del valor elegido (30%).
- Se creó un segundo dashboard atendiendo la indicación "no solo 1 dashboard".
- Se respondieron las 4 preguntas de la guía con razonamiento basado en evidencia observada durante el lab.

El stack quedó funcional al final del lab y puede levantarse de nuevo en otra máquina con `docker compose up -d --build` desde el estado actual del repositorio.
