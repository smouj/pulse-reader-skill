---
name: pulse-reader
version: 1.2.0
description: Monitorea e interpreta patrones, métricas y ritmos de eventos del sistema en tiempo real. Proporciona detección de anomalías, análisis de tendencias y perspectivas predictivas para la salud y el rendimiento del sistema.
author: OpenClaw Team
license: MIT
min_openclaw_version: "2.1.0"
tags: [pulse, systems, patterns, insights, monitoring, analysis]
category: observability
dependencies:
  - name: htop
    optional: true
  - name: iostat
    optional: true
  - name: netstat
    optional: true
  - python:
    version: "3.8+"
    packages: [numpy,pandas,scipy]
  - name: prometheus-node-exporter
    optional: true
---

# Pulse Reader Skill

## Propósito

Pulse Reader establece un sistema continuo de monitoreo de latidos que captura, correlaciona e interpreta ritmos a nivel de sistema en CPU, memoria, disco I/O, tráfico de red y logs de aplicación. A diferencia del monitoreo básico que reporta métricas estáticas, Pulse Reader identifica patrones emergentes, comportamientos cíclicos y anomalías en flujos en tiempo real.

### Casos de Uso Reales

1. **Detectar Progresión de Fuga de Memoria**: Monitorear patrones de crecimiento de heap a través de ciclos de 24-72 horas para identificar acumulación gradual de memoria que los umbrales estándar pasan por alto.

2. **Análisis de Ritmo de Tráfico de Red**: Identificar patrones diarios en tiempos de respuesta de API correlacionados con picos de volumen de solicitudes, distinguiendo carga normal de degradación.

3. **Detección de Anomalías de I/O de Almacenamiento**: Reconocer cuando la latencia de escritura en disco diverge de su línea base establecida de 5 minutos por más de 3 desviaciones estándar, indicando problemas de hardware emergentes.

4. **Correlación de Salud de Aplicación**: Correlacionar patrones entre logs de error, picos de CPU y pausas de GC para identificar causas raíz de problemas intermitentes.

5. **Pronóstico de Capacidad**: Analizar tendencias históricas de utilización de recursos para predecir cuándo la infraestructura actual excederá el 80% de uso sostenido.

## Alcance

### Comandos

`pulse start [--duration <seconds>] [--interval <ms>] [--channels <list>]`

Inicia monitoreo continuo de pulso con parámetros especificados. Por defecto: 60s duración, 1000ms intervalo, todos los canales.

`pulse status`

Muestra sesiones de monitoreo activas, estado de establecimiento de línea base y salud actual del pulso.

`pulse baseline [--calc-from <minutes>] [--set-manual <json>]`

Establece o actualiza los patrones de línea base. Calcula automáticamente desde los últimos 60 minutos por defecto.

`pulse anomalies [--since <duration>] [--severity <level>] [--channel <name>]`

Consulta anomalías detectadas con opciones de filtrado. Niveles de severidad: low, medium, high, critical.

`pulse streams [--format {json,table,chart}] [--live]`

Muestra flujos de métricas en bruto o visualización en vivo. Requiere dependencias opcionales para formato chart.

`pulse correlate [--with <skill_name>] [--over <duration>]`

Correlaciona datos de pulso con salidas de habilidades externas. Requiere configuración de integración.

`pulse export [--type {insights,raw,patterns}] [--output <path>]`

Exporta resultados de análisis en formato especificado. Las exportaciones raw incluyen historiales completos de métricas.

### Variables de Entorno

- `PULSE_INTERVAL_MS`: Intervalo de muestreo por defecto (default: 1000)
- `PULSE_STORAGE_BACKEND`: Dónde almacenar datos de pulso (sqlite, memory, prometheus)
- `PULSE_ANOMALY_SENSITIVITY`: Umbral sigma para detección de anomalías (default: 2.5)
- `PULSE_MAX_HISTORY_DAYS`: Período de retención para datos históricos (default: 30)

## Proceso de Trabajo

### 1. Fase de Inicialización

- La habilidad verifica la disponibilidad de métricas del sistema (procfs, sysfs o endpoint node-exporter)
- Carga línea base anterior si existe; de lo contrario entra en modo de establecimiento de línea base
- Inicializa ventanas móviles: 5 minutos (corto plazo), 1 hora (medio plazo), 24 horas (largo plazo)

### 2. Establecimiento de Línea Base (Primeros 60 minutos)

- Recoge métricas a intervalo configurado sin detección de anomalías
- Calcula media, mediana, desviación estándar para cada métrica por cubo de hora del día
- Identifica patrones cíclicos usando FFT (Fast Fourier Transform) en agregados de 5 minutos
- Almacena línea base en base de datos SQLite local en `~/.openclaw/pulse_baseline.db`

### 3. Fase de Monitoreo Continuo

- Cada ciclo de muestreo:
  - Recoge métricas actuales de canales configurados
  - Compara contra cubo de línea base apropiado para la hora
  - Aplica control estadístico de procesos: si valor > media + (σ × sensibilidad), marcar como anomalía
  - Actualiza arrays móviles de z-score para detección de tendencias
  - Detecta cambios de patrón usando prueba de Mann-Kendall en ventanas recientes de 30 minutos
  - Genera perspectivas solo cuando la anomalía persiste por >3 muestras consecutivas

### 4. Generación de Perspectivas

- **Anomalía**: Una única métrica se desvía de la línea base (ej. promedio de CPU de 5 min > percentil 95)
- **Cambio de Patrón**: La correlación entre métricas cambia significativamente (ej. uso de memoria ya no se correlaciona con CPU)
- **Disrupción de Ritmo**: La amplitud de ciclo establecida cambia >40% (ej. patrón de limpieza nocturno se debilita)
- **Cruzado entre Canales**: Dos o más canales muestran anomalías sincronizadas indicando problema sistémico

### 5. Formateo de Salida

- Legible para humanos: tabla coloreada con indicadores de severidad
- JSON: datos completos incluyendo valores en bruto, líneas base, z-scores
- Resumen: una línea de perspectiva por problema detectado con acción sugerida

## Reglas de Oro

1. **Nunca activar por picos únicos**: Requerir 3+ muestras de anomalía consecutivas antes de reportar. Picos breves son ruido.

2. **Líneas base de hora del día son sagradas**: Separar líneas base para entre semana/fin de semana, horas laborales/no laborales. No mezclar.

3. **Correlación ≠ causalidad**: Cuando múltiples canales muestran anomalías, reportar fuerza de correlación pero evitar afirmar causa raíz sin evidencia adicional.

4. **Respetar carga del sistema**: Si CPU del host >90%, reducir automáticamente frecuencia de muestreo en 2x hasta que la carga se normalice.

5. **Contexto histórico tiene prioridad**: Si el patrón actual coincide con una anomalía previamente permitida (ej. respaldo programado), suprimir alerta.

6. **Sin pérdida de estado al reiniciar**: Siempre persistir estado de pulso en disco. Nunca comenzar línea base fresca en cada invocación.

7. **Disciplina señal-ruido**: Si tasa de anomalías excede 20% del total de muestras durante cualquier hora, aumentar automáticamente umbral de sensibilidad y registrar advertencia.

8. **Degradación elegante de dependencias**: Si faltan bibliotecas de gráficos opcionales, retroceder a salida de texto/tabla sin error.

## Ejemplos

### Ejemplo 1: Iniciar Pulso Continuo con Canales Personalizados

```bash
pulse start --duration 1800 --interval 500 --channels cpu,memory,disk
```

Salida:
```
Starting Pulse Reader...
Duration: 1800s, Interval: 500ms, Channels: cpu,memory,disk
Baseline: Established (52 days)
Session ID: p-20260305-143022
```

### Ejemplo 2: Consultar Anomalías Críticas de las Últimas 24 Horas

```bash
pulse anomalies --since 24h --severity high
```

Salida (formato tabla):
```
TIME                 CHANNEL   METRIC        VALUE  BASELINE  Z-SCORE  INSIGHT
2026-03-04 08:15:22  memory   used_percent  94.2%   67.1%     3.8      Memory usage rhythm disrupted: sustained high during off-hours
2026-03-04 12:43:11  cpu      load_5min     4.8     1.9       2.9      CPU pattern shift: increased volatility detected
2026-03-05 02:00:05  disk     await_ms      42.3    15.2      3.2      I/O latency anomaly: 3x baseline deviation
```

### Ejemplo 3: Establecer Nueva Línea Base después de Despliegue Mayor

```bash
pulse baseline --calc-from 120
```

Salida:
```
Recalculating baseline from last 120 minutes of data...
Processing 14,400 samples across 4 channels...
Baseline updated: 2026-03-05 14:22:18 UTC
FFT detected cycles: daily (1440 min), weekly (10080 min)
Stored to ~/.openclaw/pulse_baseline.db
```

### Ejemplo 4: Correlacionar Pulso con Logs de Aplicación

```bash
pulse correlate --with log-analyzer --over 6h
```

Salida:
```
Correlation window: last 6 hours
pattern-error-rate: r=0.87 (strong positive)
pattern-response-time vs memory-pressure: r=0.72 (moderate)
Detected: Memory pressure increases precede error rate spikes by ~45 seconds
```

### Ejemplo 5: Exportar Flujo en Bruto para Análisis Offline

```bash
pulse export --type raw --output /tmp/pulse_dump.json
```

Salida:
```
Exported 1,284,000 raw samples (CPU, Memory, Disk, Network)
 compressed size: 47.3 MB
Format: JSON Lines (one sample per line)
File: /tmp/pulse_dump.json
```

## Comandos de Rollback

```bash
# Restore baseline from backup (if corrupted)
pulse baseline --set-manual "$(cat ~/.openclaw/backups/pulse_baseline_20260304.json)"

# Disable anomaly detection temporarily (maintenance window)
pulse config set anomaly_detection false

# Clear current session state (start fresh)
rm -f ~/.openclaw/pulse_session_*.db && pulse reset

# Revert to previous baseline version (if auto-update caused false positives)
pulse baseline --restore previous

# Disable specific noisy channel
pulse config set channels_exclude [disk]

# Rollback correlation settings
pulse correlate --disconnect --all
```

## Solución de Problemas

**Error: "No metric sources available"**
- Asegurar que `/proc` esté montado y legible
- Verificar node-exporter ejecutándose si está configurado: `systemctl status prometheus-node-exporter`
- Revisar `PULSE_STORAGE_BACKEND` apunte a ubicación escribible

**Línea base nunca se estabiliza (tasa de anomalías >30%)**
- El sistema puede ser inherentemente inconsistente. Ejecutar `pulse baseline --calc-from 1440` usando ciclo completo de 24h.
- Aumentar `PULSE_ANOMALY_SENSITIVITY` a 3.5 o 4.0 temporalmente.
- Revisar tareas cron recurrentes causando picos predecibles.

**Uso alto de CPU del propio pulse**
- Reducir intervalo de muestreo a >=2000ms
- Limitar canales a solo métricas esenciales
- Cambiar backend de almacenamiento a solo memoria (efímero): `PULSE_STORAGE_BACKEND=memory`

**Falsos positivos durante ventanas de mantenimiento conocidas**
- Agregar períodos de mantenimiento via `pulse whitelist add --window "Sat 02:00-04:00"`
- O temporalmente establecer `PULSE_ANOMALY_SENSITIVITY` más alto durante mantenimiento.

**Correlación cruzada no funciona**
- Asegurar que la habilidad dependiente esté generando salida activamente
- Verificar sincronización de tiempo entre logs de habilidad y marcas de tiempo de pulso
- Revisar `pulse correlate --validate` para estado de conexión.

**Errores de base de datos bloqueada**
- Otra sesión de pulse está ejecutándose. Detener todas: `pulse stop --all`
- Remover bloqueo obsoleto: `rm ~/.openclaw/pulse_*.lock`
```