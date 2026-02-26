# 🌾 S.I.G. Riego Pro v1.0 (API Connect – RDC Edition)

**Sistema de Información Geográfica para la Gestión Integral de Recursos Hídricos**, orientado al diseño, planificación y evaluación estacional del riego agrícola mediante:

* Climatología histórica oficial (AEMET OpenData)
* Cálculo físico riguroso de ET<sub>o</sub> (FAO-56 Penman–Monteith)
* Balance hídrico agronómico mensual
* Redistribución operativa semanal con control hidráulico
* Sistema resiliente de estaciones con fallback inteligente

---

# 🎯 Objetivo del sistema

Proporcionar una **estimación robusta, reproducible y auditable** de las necesidades hídricas de un cultivo para campañas presentes o futuras, basada en:

* Series históricas reales (~36 meses efectivos AEMET)
* Modelización física estandarizada FAO-56
* Separación explícita entre:

  * Demanda evaporativa (capa física)
  * Gobernanza hidráulica (capa operativa)

El sistema está diseñado para defender técnicamente su uso en:

* Planificación estacional de dotaciones
* Estudios comparativos
* Evaluación de escenarios de campaña
* Diseño de turnos de riego

---

# 📍 1. Selección y validación de estaciones

## 📏 Cálculo de distancia real

Se utiliza la fórmula de Haversine para calcular la distancia geodésica entre:

* Coordenadas de parcela introducidas por el usuario
* Todas las estaciones AEMET disponibles

Se establece:

* 🔵 Estación principal → la más cercana
* 🟢 Hasta 5 estaciones de apoyo → ordenadas por distancia

---

## 🧪 Diagnóstico de cobertura por variable

Para cada estación candidata se evalúa la cobertura histórica (~36 meses):

* Temperatura (tm_mes o tm_max + tm_min)
* Humedad relativa (hr)
* Viento (w_med)
* Radiación (glo / inso)

La selección para el cálculo no descarta una estación completa, sino que trabaja:

> 🔎 Por variable y por mes.

---

# 🛰️ 2. Motor climático histórico (Fallback mensual por variable)

## 📅 Ventana temporal

Se utilizan los últimos 3 años completos disponibles en AEMET:

* Año N-2
* Año N-1
* Año N (último completo)

Resultado típico: 36–39 meses efectivos.

No se fuerzan meses inexistentes.

---

## 🔁 Fallback inteligente por variable (no por estación)

Para cada mes natural del ciclo:

| Variable    | Prioridad                  |
| ----------- | -------------------------- |
| Temperatura | Principal → hasta 5 apoyos |
| HR          | Principal → hasta 5 apoyos |
| Viento      | Principal → hasta 5 apoyos |
| Radiación   | Principal → hasta 5 apoyos |

El sistema:

* Resuelve mensualmente
* Registra estación usada
* Mantiene trazabilidad completa

Esto evita perder meses completos por fallo parcial de una variable.

---

# 🌡️ 3. Cálculo de Evapotranspiración de Referencia (ET<sub>o</sub>)

## 📐 Método: FAO-56 Penman–Monteith

Se aplica la formulación estándar FAO-56:

## 📐 Ecuación FAO-56 Penman–Monteith

![Ecuación FAO-56 Penman-Monteith](docs/img/pm_fao56.svg)

Fuente: Allen et al. (1998). FAO Irrigation and Drainage Paper No. 56.

Donde:

* Δ = pendiente de la curva de presión de vapor
* Rn = radiación neta
* G = flujo de calor del suelo (≈ 0 en mensual)
* γ = constante psicrométrica
* u2 = velocidad del viento a 2 m
* es − ea = déficit de presión de vapor

---

## 🔄 Ajuste del viento

El viento AEMET (w_med) se convierte a u2 mediante la ecuación FAO-56:

u2 = uz × 4.87 / ln(67.8 z − 5.42)

Asumiendo altura estándar de medición ≈ 10 m.

---

## ☀ Radiación

Prioridad de cálculo:

1. Radiación global (glo) si está disponible
2. Insolación (inso) mediante Angström-Prescott
3. Fallback a estaciones de apoyo

---

## 📅 Día representativo mensual

Se utiliza el día 15 del mes como día juliano representativo.

* En mensual, el flujo de calor del suelo G ≈ 0.
* Método estándar en estudios de riego estacionales.

---

# 🌱 4. Balance Hídrico Agronómico

## 🔹 Evapotranspiración del cultivo

ET<sub>c</sub> = ET<sub>o</sub> × K<sub>c</sub>

---

## 🔹 Precipitación efectiva (P<sub>e</sub>)

Modelo tipo USDA/SCS:

* Si P < 70 mm → 0.6 P − 10
* Si P ≥ 70 mm → 0.8 P − 24
* Nunca negativa
* Prorrateada en meses parciales

---

## 🔹 Necesidades Hídricas Netas

NH<sub>n</sub> = (ET<sub>c</sub> − P<sub>e</sub>) × 10

Unidad: m³/ha

---

# 💧 5. Sistema RDC (Redistribución de Dotación por Cultivo)

Nueva capa implementada en la versión actual.

## 🎛 Ajuste mensual porcentual

Para cada mes:

* El usuario introduce %RDC (0–100)
* Se calcula:

RDC (m³/ha) = NH<sub>n</sub> × (RDC% / 100)

El sistema distingue:

* 🔵 Asignado proporcional por recursos disponibles
* 🟢 Ajuste RDC mensual
* 🔴 Total mensual resultante

No recalcula AEMET ni ETo.

---

# 📅 6. Programación Semanal

## 🔵 Capa física: ETo-weighted intra-mensual

Dentro de cada mes:

1. Se construye la lista de días activos.
2. Se interpola ETo entre mes actual y siguiente:

ETo_d = ETo_m + (ETo_m+1 − ETo_m) × t

donde:

t = (día − 1) / (días_mes − 1)

3. Se reparte el volumen mensual proporcional a los pesos diarios.

Resultado:

* Curva suave
* Continuidad estacional
* Conservación exacta del total mensual

---

## 🟠 Capa hidráulica: clamp semanal ±10 %

Una vez agregados días por semana natural:

Para cada semana parcial dentro del mes:

Uniforme = Volumen_mensual × (días_semana / días_mes)

Mínimo = 0.9 × Uniforme
Máximo = 1.1 × Uniforme

Si el valor semanal generado excede esos límites:

* Se ajusta al rango permitido
* Se redistribuye el residuo
* Se conserva el total mensual exacto

---

## 🔎 Semanas mixtas (entre dos meses)

Si una semana contiene días de dos meses:

* Cada mes aplica su propio clamp parcial
* La semana final es suma de ambas partes
* No se aplica un segundo clamp global

---

# 🧠 Fundamento científico del modelo

El sistema separa explícitamente:

| Capa         | Naturaleza | Justificación            |
| ------------ | ---------- | ------------------------ |
| ETo-weighted | Física     | Demanda evaporativa real |
| Clamp ±10%   | Operativa  | Estabilidad hidráulica   |

El ±10 %:

* No altera ETo
* No modifica física atmosférica
* Actúa como amortiguador operativo

---

# 🧾 Trazabilidad completa

El sistema registra por mes:

* Estación usada por variable
* Si fue principal o apoyo
* Fuente exacta de radiación
* Variables imprescindibles validadas

Permite auditoría técnica y validación externa (SIAR u otros).

---

# 💻 Stack Tecnológico

* Datos climáticos: AEMET OpenData (REST)
* Frontend: HTML5 + ES6 Vanilla JavaScript
* Visualización: Chart.js
* Exportación: SheetJS (XLSX)
* Arquitectura: Cliente puro (sin backend intermedio)

---

# 🔐 Configuración API Key

En el código:

```javascript
const API_KEY = "TU_AEMET_API_KEY";
```

Requiere registro previo en AEMET OpenData.

---

# ⚖ Decisiones de diseño

1. Uso de climatología histórica, no predicción meteorológica.
2. Fallback por variable, no por estación única.
3. Separación física (demanda) / hidráulica (operatividad).
4. Resolución mensual coherente con planificación estacional.
5. Conservación estricta del volumen mensual y total campaña.

---

# 🚫 Limitaciones

* No modela balance dinámico de suelo.
* No incorpora eficiencia de aplicación.
* No sustituye sensores de humedad.
* No captura eventos extremos diarios.
* No es un modelo de predicción meteorológica.

---

# 📚 Referencias técnicas

* Allen, R.G. et al. (1998). FAO Irrigation and Drainage Paper 56.
* USDA Soil Conservation Service – Effective Rainfall.
* Angström (1924), Prescott (1940).

---

# 📌 Filosofía del sistema

Modelo físico sólido + capa hidráulica prudente.
Transparencia y trazabilidad antes que complejidad opaca.
Robustez agronómica para planificación estacional real.

---
