# 🗒️ Registro de Trabajo en Clase — Taller 4: Mapa de Infraestructura y Diagnóstico Técnico

## 📆 Fecha de la sesión
7/03/2026

## 👥 Integrantes presentes
- Andrea Sosa
- Samuel Rodriguez
- Juan Gomez

---

## 🧠 Actividades realizadas en clase

### ¿Qué se discutió con el equipo?

El equipo analizó la arquitectura de **RedExpress**, una plataforma de logística híbrida que combina servicios en la nube pública, servidores regionales on-premise y dispositivos móviles en campo (mensajeros). La discusión se centró en entender los flujos principales del sistema:

1. **Flujo de rastreo de paquetes:** los mensajeros actualizan el estado desde sus dispositivos → el evento viaja por el API Gateway → el Tracking Service lo procesa y publica en Kafka → la BD y los clientes reciben la actualización.
2. **Flujo de optimización de rutas:** cuando se asigna una entrega, el Motor de Rutas calcula la ruta óptima de forma asíncrona usando una cola (RabbitMQ/SQS) para no bloquear la respuesta al usuario.
3. **Flujo de temporada alta (ej. Navidad):** escenario donde el volumen de paquetes puede multiplicarse por 5-10x, lo que expone cuellos de botella en la BD centralizada y en el servicio de rastreo.

### ¿Qué decisiones de modelado se tomaron?

- Se organizó la infraestructura en **7 capas horizontales** para facilitar la lectura y el análisis por zona de responsabilidad:

  | Capa | Descripción |
  |------|-------------|
  | 1. Acceso — Clientes | App móvil, plataforma web, panel admin, CDN, DNS |
  | 2. Seguridad y Entrada | WAF, protección DDoS, Load Balancer global, API Gateway, Auth Service |
  | 3. Microservicios (Nube) | Tracking, Rutas, Notificaciones, Paquetes, Pagos, Usuarios, Reportes |
  | 4. Mensajería Asíncrona | Apache Kafka, Event Bus (SNS), Cola de Rutas (RabbitMQ) |
  | 5. Datos y Almacenamiento | PostgreSQL (master/réplica), Redis, MongoDB, S3, Elasticsearch |
  | 6. Regional — Híbrido | Nodos Norte, Sur, Costa + Centros de distribución + Dispositivos |
  | 7. Monitoreo y CI/CD | Prometheus/Grafana, ELK, PagerDuty, Jaeger, GitHub Actions, Kubernetes |

- Se decidió **marcar visualmente en rojo/naranja** todos los componentes identificados como críticos (SPOF, cuellos de botella o con latencia elevada) para facilitar el diagnóstico a primera vista.
- Se utilizó **línea sólida** para comunicación síncrona (REST/HTTPS) y **línea punteada** para comunicación asíncrona (eventos/mensajería).

### ¿Qué herramientas se usaron?

- **draw.io** (diagrams.net) — para construir el mapa lógico de infraestructura de forma colaborativa.
- Discusión en equipo con pizarra conceptual para distribuir capas y relaciones.
- Exportación a PNG para facilitar visualización y entrega del diagrama sin necesidad de abrir draw.io.

### Nota sobre la organización visual del diagrama

Al pie del mapa se ubicaron dos bloques separados lado a lado:
- **Leyenda de diagnóstico** (izquierda): indica el significado de los colores (rojo/naranja = crítico, verde = estable) y el tipo de línea (sólida = síncrono, punteada = asíncrono).
- **Diagnóstico técnico** (derecha): lista los 4 puntos críticos identificados, cada uno numerado con su descripción concisa.

Esta separación evita confusión visual entre los dos bloques y permite leer cada uno de forma independiente.

### ¿Qué parte del trabajo se alcanzó a desarrollar?

- ✅ Mapa de infraestructura preliminar completo (`mapa-borrador.drawio`) con las 7 capas modeladas.
- ✅ Exportación del mapa como imagen PNG (`mapa-borrador.png`) para entrega y presentación.
- ✅ Identificación de 4 zonas críticas con justificación técnica.
- ✅ Definición del flujo de comunicación síncrona y asíncrona entre servicios.
- ✅ Leyenda de diagnóstico y tabla de puntos críticos incluidas en el propio diagrama, lado a lado en la parte inferior.

---

## 🧩 Boceto inicial del modelo

El diagrama se construyó directamente en draw.io (archivo adjunto `mapa-borrador.drawio`). La estructura lógica es la siguiente:

```
[Clientes: App / Web / Admin]
          ↓ HTTPS
[WAF + DDoS + Load Balancer Global ⚠️ SPOF]
          ↓
[API Gateway → Auth Service]
          ↓ REST
[Microservicios: Tracking ⚠️ | Rutas | Notif | Paquetes | Pagos | Usuarios | Reportes ⚠️]
          ↓ Kafka / SQS / RabbitMQ (asíncrono)
[Datos: PostgreSQL ⚠️ (master) | Réplica | Redis | MongoDB | S3 | Elasticsearch]
          ↕ VPN / Sync
[Nodos Regionales: Norte | Sur | Costa ⚠️ SPOF ISP | Centros Distribución | Dispositivos]
          ↓ métricas/logs
[Monitoreo: Prometheus | ELK | PagerDuty | Jaeger | CI/CD | Kubernetes]
```

---

## 🔍 Zonas sensibles identificadas

### ❶ Base de datos PostgreSQL centralizada — Cuello de botella principal

**Problema:** Toda la escritura transaccional pasa por un único nodo master de PostgreSQL. En temporadas de alto volumen (Navidad, campañas promocionales), el número de inserciones de estados de paquetes puede saturar la BD.

**Impacto:** Degradación general del sistema — los servicios de rastreo, gestión de paquetes y reportes compiten por los mismos recursos.

**Posible solución:** Implementar sharding por zona geográfica o migrar a una arquitectura de BD distribuida (CockroachDB, AWS Aurora Global); separar la escritura de la lectura con réplicas de lectura dedicadas.

---

### ❷ Nodo Regional Costa — SPOF geográfico por enlace ISP único

**Problema:** El nodo de la zona Costa (Barranquilla/Cartagena) depende de un único enlace a internet de un solo proveedor ISP. Si ese enlace falla, todos los centros de distribución y mensajeros de esa zona quedan desconectados de la plataforma central.

**Impacto:** Pérdida total de visibilidad de entregas en la región costa; incumplimiento de SLA para clientes de esa zona.

**Posible solución:** Contratar un segundo ISP (failover activo-pasivo o active-active) o usar un enlace de respaldo LTE/satelital para garantizar continuidad mínima.

---

### ❸ Tracking Service sin caché — Alta latencia en rastreo en tiempo real

**Problema:** El servicio de rastreo consulta directamente la BD para cada solicitud de estado en tiempo real (vía WebSocket). Con miles de usuarios simultáneos consultando el estado de sus paquetes, esto genera carga masiva tanto en el servicio como en la BD.

**Impacto:** Latencia elevada en la app y web; experiencia degradada para el cliente final; riesgo de caídas del servicio en picos.

**Posible solución:** Introducir una capa de caché Redis entre el Tracking Service y la BD (ya modelada en el diagrama). Los estados de paquete tienen una frecuencia de actualización baja (minutos), por lo que un TTL corto en caché reduce drásticamente la carga.

---

### ❹ Load Balancer Global sin alta disponibilidad (HA)

**Problema:** El Load Balancer de capa 7 actúa como único punto de entrada para todo el tráfico externo. Si no está configurado en modo HA (activo-activo o activo-pasivo con failover automático), su caída implica la caída total de la plataforma para todos los usuarios.

**Impacto:** Interrupción completa del servicio — ningún cliente (app, web, mensajeros) puede conectarse.

**Posible solución:** Desplegar el LB en al menos dos zonas de disponibilidad (AZ) distintas con health checks activos; usar un servicio gestionado como AWS ALB o Google Cloud Load Balancing que ofrecen HA nativa.

---

### ❺ (Adicional) Servicio de Reportes sin escalabilidad horizontal

**Problema:** El servicio de reportes ejecuta consultas analíticas pesadas sobre la misma BD transaccional, lo que interfiere con las operaciones del negocio en tiempo real.

**Impacto:** Degradación del rendimiento transaccional durante la generación de reportes; tiempos de respuesta elevados para el equipo de operaciones.

**Posible solución:** Separar el workload analítico del transaccional mediante un data warehouse (Amazon Redshift, BigQuery) o OLAP store alimentado asíncronamente por el Event Bus.

---

## 🔁 Tareas definidas para complementar el taller

| Tarea asignada | Responsable | Fecha estimada |
|---|---|---|
| Modelado final en draw.io (cliente real) | Juan Gomez | 14/03/2026 |
| Redacción del informe de diagnóstico técnico | Samuel Rodriguez | 15/03/2026 |
| Investigación de buenas prácticas y referencias | Andrea Sosa | 15/03/2026 |
| Revisión y consolidación de la entrega final | Todos | 16/03/2026 |

---

## 📌 Conclusiones de la sesión

La sesión permitió visualizar que RedExpress enfrenta riesgos arquitectónicos típicos de plataformas que escalan rápidamente sin rediseñar su núcleo de datos. Los puntos más urgentes son:

1. **La centralización de la BD** es el riesgo técnico más alto a mediano plazo.
2. **La falta de redundancia regional** en el nodo Costa representa un riesgo operativo inmediato.
3. **El Tracking Service** necesita optimización antes de la próxima temporada alta para evitar degradación de experiencia de usuario.

El mapa generado servirá de base para la adaptación al cliente real en la Parte 2 del taller.

---

_Este documento resume el trabajo colaborativo realizado durante la sesión del Taller 4 en el curso AREM — Universidad de La Sabana._
