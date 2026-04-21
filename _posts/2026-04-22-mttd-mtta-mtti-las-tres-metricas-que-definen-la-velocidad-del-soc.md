---
layout: post
title: "MTTD, MTTA y MTTI: Las Tres Métricas que Definen la velocidad del SOC"
date: 2026-04-22
categories: [blueteam, incident-response]
tags: [mttd, mtta, mtti, metricas, soc, dfir, deteccion]
mermaid: true
---

## La carrera contra el reloj empieza aquí

En la entrada anterior https://tnagata.com/posts/dwell-time-La-Metrica-que-Separa-la-Contencion-de-la-Cataastrofe/ se explicó el Dwell Time como la métrica estratégica que mide cuánto tiempo permanece un atacante sin ser detectado. Pero el Dwell Time es un resultado — no se puede mejorar directamente. Lo que sí se puede mejorar son los tres segmentos temporales que lo componen: **detectar** la amenaza, **reconocer** la alerta y **investigar** el incidente.

Estas tres métricas conforman el grupo de **Detección y Triaje**, y juntas determinan cuánto tiempo pasa desde que un atacante ejecuta su primera acción maliciosa hasta que el equipo de seguridad comprende lo que está pasando y puede actuar.

```mermaid
graph TD
    T0["COMPROMISO INICIAL<br/>El atacante ejecuta su primera accion"]
    T0 --> MTTD_S["MTTD<br/>Mean Time To Detect"]
    MTTD_S --> T1["ALERTA GENERADA<br/>El SIEM, EDR o IDS dispara una alerta"]
    T1 --> MTTA_S["MTTA<br/>Mean Time To Acknowledge"]
    MTTA_S --> T2["ALERTA RECONOCIDA<br/>Un analista acepta y comienza a trabajar"]
    T2 --> MTTI_S["MTTI<br/>Mean Time To Investigate"]
    MTTI_S --> T3["INVESTIGACION COMPLETA<br/>Alcance, causa raiz e impacto definidos"]
    T3 --> NEXT["Siguiente fase: MTTC - Contencion"]

    style T0 fill:#1a0a0a,stroke:#ff3355,color:#ff3355
    style T1 fill:#0d1117,stroke:#00b4ff,color:#00b4ff
    style T2 fill:#0d1117,stroke:#00e5ff,color:#00e5ff
    style T3 fill:#0d1117,stroke:#aa44ff,color:#aa44ff
    style NEXT fill:#0d1117,stroke:#1a2a3a,color:#556677
    style MTTD_S fill:#0d1117,stroke:#00b4ff,color:#00b4ff
    style MTTA_S fill:#0d1117,stroke:#00e5ff,color:#00e5ff
    style MTTI_S fill:#0d1117,stroke:#aa44ff,color:#aa44ff
```

La relación matemática fundamental es:

```
Tiempo_hasta_respuesta = MTTD + MTTA + MTTI
```

Si esa suma es mayor que el Breakout Time del atacante (29 minutos en promedio para eCrime en 2025, según CrowdStrike), el adversario completa su movimiento lateral **antes** de que el SOC siquiera entienda lo que está pasando.

---

## MTTD — Mean Time To Detect

### Definición técnica

MTTD mide el tiempo promedio entre el momento en que ocurre la actividad maliciosa y el momento en que el stack de seguridad de la organización **genera una alerta** o un analista identifica la anomalía. Es la métrica que refleja la capacidad de detección pura — la calidad de los sensores, las reglas, la telemetría y la cobertura de monitoreo.

Un MTTD bajo significa que las herramientas de seguridad (SIEM, EDR, NDR, UEBA) están capturando la actividad maliciosa rápidamente. Un MTTD alto indica puntos ciegos: endpoints sin agente, segmentos de red sin visibilidad, reglas de detección insuficientes, o simplemente que el atacante está usando técnicas que evaden las detecciones existentes.

### Fórmula

```
MTTD = Suma(Timestamp_Alerta - Timestamp_Compromiso) / Numero_de_incidentes
```

> **Nota importante**: determinar el `Timestamp_Compromiso` es el desafío real. Se establece retroactivamente durante el análisis forense buscando la evidencia más temprana de actividad maliciosa en logs, artefactos de disco y capturas de red. Si la telemetría es insuficiente, T0 no se puede determinar con precisión — y el MTTD reportado será artificialmente bajo.
{: .prompt-info }

### Benchmarks de industria

No existe un benchmark universal porque el MTTD depende del tamaño de la organización, la industria y el modelo de amenazas. Pero las fuentes más confiables proporcionan rangos de referencia:

| Fuente | Dato | Contexto |
|---|---|---|
| **SANS 2023 IR Survey** | Top 25% detecta en menos de 60 minutos | Medido a nivel SOC en incidentes activos |
| **SANS 2023 IR Survey** | Mas del 50% detecta en menos de 5 horas | Incluye organizaciones medianas |
| **IBM Cost of a Data Breach 2025** | Promedio de 158 dias para identificar brechas | Medido a nivel de brecha completa, no alertas individuales |
| **Prophet Security 2026** | Top performers: 30 min a 4 horas | Rango objetivo para SOCs maduros |
| **Industrias de alto riesgo** | Objetivo: menos de 1 hora | Finanzas, salud, infraestructura critica |

La diferencia entre el dato de SANS (minutos a horas) y el de IBM (158 dias) no es una contradicción — refleja dos cosas distintas. El MTTD a nivel de SOC mide alertas en tiempo real contra actividad conocida. El dato de IBM mide brechas completas, incluyendo intrusiones lentas y sigilosas que evadieron toda la detección inicial durante meses.

### Que mide y que no mide

```mermaid
graph TD
    MTTD_N["MTTD"]
    MTTD_N --> SI["SI mide"]
    MTTD_N --> NO["NO mide"]

    SI --> S1["Calidad de reglas de deteccion<br/>en SIEM, EDR, IDS"]
    SI --> S2["Cobertura de telemetria<br/>y logging centralizado"]
    SI --> S3["Capacidad de correlacion<br/>entre fuentes de datos"]
    SI --> S4["Efectividad de deteccion<br/>basada en comportamiento"]

    NO --> N1["Si alguien vio la alerta<br/>Eso es MTTA"]
    NO --> N2["Si se investigo correctamente<br/>Eso es MTTI"]
    NO --> N3["Si se contuvo la amenaza<br/>Eso es MTTC"]
    NO --> N4["La severidad o impacto<br/>del incidente"]

    style MTTD_N fill:#0d1117,stroke:#00b4ff,color:#00b4ff
    style SI fill:#0d1117,stroke:#00ff88,color:#00ff88
    style NO fill:#0d1117,stroke:#ff3355,color:#ff3355
    style S1 fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style S2 fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style S3 fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style S4 fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style N1 fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style N2 fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style N3 fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style N4 fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
```

### Tres escenarios comparativos: mismo ataque, diferente MTTD

El mismo ataque produce resultados radicalmente distintos según el stack de detección disponible. Escenario: un empleado de finanzas recibe un phishing dirigido a las 09:00 del lunes. Hace clic en el enlace y se instala un loader de Cobalt Strike. El beacon inicia callbacks C2 cada 60 segundos.

```mermaid
graph TD
    ATK["MISMO ATAQUE<br/>Phishing - Cobalt Strike beacon - Callbacks C2 cada 60s"]

    ATK --> A
    ATK --> B
    ATK --> C

    A["ESCENARIO A: EDR MODERNO"]
    A --> A1["09:00 - Empleado ejecuta el payload"]
    A1 --> A2["09:03 - EDR detecta process hollowing<br/>Alerta automatica generada"]
    A2 --> A3["MTTD = 3 minutos"]

    B["ESCENARIO B: AV LEGACY + SIEM BASICO"]
    B --> B1["09:00 Lunes - Payload ejecutado<br/>AV no detecta: crypter personalizado"]
    B1 --> B2["09:15 Martes - SIEM detecta beaconing<br/>Necesito 24h de datos estadisticos"]
    B2 --> B3["MTTD = 24 horas 15 minutos"]

    C["ESCENARIO C: SIN EDR NI REGLAS"]
    C --> C1["09:00 Lunes - Cobalt Strike operando"]
    C1 --> C2["Dia 10 - Ransomware desplegado<br/>Helpdesk: no puedo abrir mis archivos"]
    C2 --> C3["MTTD = 10 dias"]

    style ATK fill:#0d1117,stroke:#00b4ff,color:#00b4ff
    style A fill:#0a1a0a,stroke:#00ff88,color:#00ff88
    style B fill:#1a1a0a,stroke:#ff8800,color:#ff8800
    style C fill:#1a0a0a,stroke:#ff3355,color:#ff3355
    style A3 fill:#0a1a0a,stroke:#00ff88,color:#00ff88
    style B3 fill:#1a1a0a,stroke:#ff8800,color:#ff8800
    style C3 fill:#1a0a0a,stroke:#ff3355,color:#ff3355
    style A1 fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style A2 fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style B1 fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style B2 fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style C1 fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style C2 fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
```

La diferencia entre un MTTD de 3 minutos y uno de 10 dias es la diferencia entre contener un incidente en fase inicial o sufrir un evento catastrófico. Y la variable que determina esa diferencia no es suerte — es la inversión en el stack de detección.

### Factores que inflan el MTTD

```mermaid
graph TD
    HIGH["MTTD ALTO<br/>El atacante opera sin ser visto"]

    HIGH --> F1["GAPS DE VISIBILIDAD"]
    F1 --> F1a["Endpoints sin EDR"]
    F1 --> F1b["Segmentos de red sin captura"]
    F1 --> F1c["Servidores sin Sysmon"]
    F1 --> F1d["Cloud sin logging habilitado"]

    HIGH --> F2["REGLAS INSUFICIENTES"]
    F2 --> F2a["Solo deteccion basada en IOCs<br/>hashes, IPs, dominios"]
    F2 --> F2b["Sin reglas de comportamiento<br/>para TTPs reales"]
    F2 --> F2c["Reglas no mapeadas a<br/>MITRE ATT&CK"]

    HIGH --> F3["RUIDO EXCESIVO"]
    F3 --> F3a["73% de equipos citan falsos<br/>positivos como desafio principal<br/>(SANS 2025)"]
    F3 --> F3b["40% de alertas nunca se<br/>investigan (AI SOC Market 2025)"]
    F3 --> F3c["La alerta real se pierde<br/>entre 500 alertas diarias"]

    HIGH --> F4["COBERTURA TEMPORAL"]
    F4 --> F4a["SOC opera solo 8x5"]
    F4 --> F4b["16 horas nocturnas sin<br/>nadie mirando alertas"]

    style HIGH fill:#1a0a0a,stroke:#ff3355,color:#ff3355
    style F1 fill:#0d1117,stroke:#ff8800,color:#ff8800
    style F2 fill:#0d1117,stroke:#ff8800,color:#ff8800
    style F3 fill:#0d1117,stroke:#ff8800,color:#ff8800
    style F4 fill:#0d1117,stroke:#ff8800,color:#ff8800
    style F1a fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style F1b fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style F1c fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style F1d fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style F2a fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style F2b fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style F2c fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style F3a fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style F3b fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style F3c fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style F4a fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style F4b fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
```

### Estrategias de reduccion

| Estrategia | Detalle técnico | Impacto |
|---|---|---|
| **Cobertura de logging** | Habilitar Sysmon (config SwiftOnSecurity) en todos los endpoints. Centralizar eventos de AD: 4624, 4625, 4672, 4768, 4769. | Elimina puntos ciegos donde el atacante opera sin dejar trazas. |
| **Reglas basadas en TTPs** | Implementar reglas Sigma mapeadas a MITRE ATT&CK. Priorizar las técnicas que los threat actors relevantes para la industria realmente usan. | Detecta comportamientos en lugar de IOCs. Los IOCs cambian diariamente; los comportamientos son estables. |
| **Detección por comportamiento** | UEBA para identificar anomalias: un usuario que nunca accede a servidores de desarrollo ejecuta `whoami` y `net group "Domain Admins"` a las 3 AM. | Detecta lo que las firmas no pueden: credenciales legítimas usadas de forma ilegítima. |
| **Tuning agresivo de FP** | Reducir falsos positivos hasta que al menos el 30% de las alertas sean accionables. SOCs maduros apuntan a mas del 50%. | Menos ruido hace que las alertas reales sean visibles. Impacto directo en MTTA también. |
| **Cobertura 24x7** | Extender el SOC de 8x5 a 24x7, aunque sea con MDR externo. | Elimina las 16 horas nocturnas donde nadie mira las alertas. |

### Perspectiva Red Team a Blue Team

Desde el lado ofensivo, el MTTD es exactamente lo que el atacante intenta maximizar. Cada técnica de evasión (ofuscación de payloads, uso de herramientas legítimas, tunneling DNS, beaconing con jitter aleatorio) tiene un solo propósito: **evitar que el MTTD sea bajo**. El atacante sabe que si el EDR detecta su beacon en 3 minutos, el engagement falló. Pero si puede evadir la detección durante días, tiene tiempo para completar la kill chain.

Desde el lado defensivo, el MTTD es la primera barrera. Si falla, todo lo demás (MTTA, MTTI, MTTC) es irrelevante — no se puede reconocer una alerta que nunca se generó, ni investigar un incidente que nunca se detectó.

---

## MTTA — Mean Time To Acknowledge

### Definición técnica

MTTA mide el tiempo promedio entre el momento en que **se genera una alerta** y el momento en que **un analista humano la reconoce** y comienza a trabajar en ella. Es el "tiempo muerto" entre la detección automatizada y la intervención humana.

Una alerta sin reconocer es un incidente sin gestionar. El MTTA captura un problema estructural de los SOC modernos: las alertas se generan pero nadie las mira durante minutos, horas o incluso días.

### Fórmula

```
MTTA = Suma(Timestamp_Acknowledge - Timestamp_Alerta) / Numero_de_incidentes
```

### Benchmarks

| Severidad | Objetivo MTTA | Fuente |
|---|---|---|
| **Critica (SEV-1)** | Menos de 5 minutos | MetricFire 2026, Prophet Security |
| **Alta (SEV-2)** | Menos de 10 minutos | MetricFire 2026 |
| **Media (SEV-3)** | Menos de 20 minutos | MetricFire 2026 |
| **Baja (SEV-4)** | Menos de 1 hora | General |
| **Top performers general** | 10 minutos a 1 hora | Prophet Security 2026 |

> **Dato alarmante**: un MTTA de 36 minutos ya se considera demasiado lento para gestión moderna de incidentes según MetricFire. Si el Breakout Time promedio de eCrime es de 29 minutos, incluso un MTTA de 30 minutos significa que el atacante ya hizo movimiento lateral antes de que un analista siquiera abriera la alerta.
{: .prompt-danger }

### La crisis del alert fatigue

El MTTA no se infla por incompetencia de los analistas. Se infla por un problema sistémico que tiene nombre propio: **alert fatigue**.

```mermaid
graph TD
    AF["ALERT FATIGUE<br/>El problema central del MTTA"]

    AF --> D1["960 alertas diarias promedio<br/>por organizacion<br/>(AI SOC Market Landscape 2025)"]
    AF --> D2["Empresas de +20,000 empleados<br/>reciben mas de 3,000 alertas diarias"]
    AF --> D3["40% de alertas nunca se investigan"]
    AF --> D4["61% de equipos admitieron ignorar<br/>alertas que resultaron ser criticas"]
    AF --> D5["66% de equipos no pueden mantener<br/>el ritmo del volumen de alertas<br/>(SANS SOC Survey 2025)"]

    D1 --> R["RESULTADO"]
    D3 --> R
    D5 --> R

    R --> R1["MTTA se infla de minutos a horas"]
    R --> R2["Analistas desensibilizados"]
    R --> R3["Incidentes reales pasan<br/>desapercibidos en el ruido"]
    R --> R4["80% de analistas se sienten<br/>constantemente atrasados<br/>(Osterman Research)"]

    style AF fill:#1a0a0a,stroke:#ff3355,color:#ff3355
    style R fill:#0d1117,stroke:#ff8800,color:#ff8800
    style D1 fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style D2 fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style D3 fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style D4 fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style D5 fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style R1 fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style R2 fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style R3 fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style R4 fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
```

### Ejemplo comparativo: mismo SOC, diferente contexto

Para demostrar que el MTTA es sensible al contexto operativo, se presenta la misma alerta en dos momentos distintos del mismo SOC:

**Alerta**: "Suspicious PowerShell execution — encoded command detected" en un servidor de base de datos de producción.

| Condición | Hora de alerta | Hora de acknowledge | MTTA | Por que |
|---|---|---|---|---|
| **Turno nocturno, baja carga** | 02:30 AM | 02:47 AM | 17 minutos | Pocas alertas en cola. El analista pudo atenderla rápido. |
| **Martes post-Patch Tuesday** | 10:15 AM | 11:30 AM | 1 hora 15 min | 200+ alertas de cambios de configuración. La alerta real se enterró en el backlog. |

La diferencia no es la calidad del equipo — es la relación señal/ruido. El mismo analista, con las mismas habilidades, produce un MTTA 4 veces peor cuando el ruido de fondo es alto.

### Estrategias de reduccion

```mermaid
graph TD
    RED["REDUCIR MTTA"]

    RED --> S1["AUTOMATIZACION DE TRIAGE"]
    S1 --> S1a["SOAR enriquece alertas automaticamente:<br/>consulta threat intel, reputacion<br/>de IPs y dominios, actividad previa<br/>del usuario"]
    S1 --> S1b["Cuando el analista abre la alerta<br/>ya tiene todo el contexto para decidir"]

    RED --> S2["NIVELES DE SEVERIDAD CLAROS"]
    S2 --> S2a["P1 Critica: atencion inmediata<br/>Ej: ransomware activo"]
    S2 --> S2b["P2 Alta: atencion en menos de 10 min<br/>Ej: lateral movement detectado"]
    S2 --> S2c["P3 Media: atencion en menos de 20 min<br/>Ej: politica de contrasenas violada"]
    S2 --> S2d["P4 Baja: proximo turno<br/>Ej: software no autorizado"]

    RED --> S3["REGLA DE LOS 30 DIAS"]
    S3 --> S3a["Si nadie actua sobre un tipo de<br/>alerta en 30 dias, eliminarla.<br/>No ajustarla. Eliminarla.<br/>Equipos reportan reduccion del<br/>40% en MTTA solo con esto."]

    RED --> S4["ROTACION DE ON-CALL"]
    S4 --> S4a["Equipos que no rotan queman<br/>a sus analistas. Alert fatigue<br/>es un problema de personas,<br/>no solo de tecnologia."]

    style RED fill:#0d1117,stroke:#00ff88,color:#00ff88
    style S1 fill:#0d1117,stroke:#00b4ff,color:#00b4ff
    style S2 fill:#0d1117,stroke:#00b4ff,color:#00b4ff
    style S3 fill:#0d1117,stroke:#00b4ff,color:#00b4ff
    style S4 fill:#0d1117,stroke:#00b4ff,color:#00b4ff
    style S1a fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style S1b fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style S2a fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style S2b fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style S2c fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style S2d fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style S3a fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style S4a fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
```

> **La regla de los 30 dias** merece atención especial: si un tipo de alerta lleva 30 dias sin que nadie actúe sobre ella, se debe eliminar del pipeline. No ajustar. No reducir severidad. Eliminar. Suena radical, pero equipos que lo implementaron reportan reducciones del 40% en MTTA porque el volumen de ruido baja drásticamente.
{: .prompt-tip }

---

## MTTI — Mean Time To Investigate

### Definición técnica

MTTI mide el tiempo promedio desde que un analista **reconoce una alerta** hasta que la investigación se **completa**: se comprende que ocurrió, que sistemas están afectados, cuál es el alcance del compromiso y cuál es la respuesta apropiada. Es el tiempo de análisis puro — la fase donde la experiencia del analista, la calidad de las herramientas y la disponibilidad de telemetría hacen la diferencia.

### Fórmula

```
MTTI = Suma(Timestamp_Fin_Investigacion - Timestamp_Acknowledge) / Numero_de_incidentes
```

El desafío práctico es definir cuándo "termina" la investigación. En la práctica, se marca cuando el analista escala el incidente con un assessment completo o cuando se toma la decisión de contener.

### Benchmarks

| Nivel | Objetivo MTTI | Contexto |
|---|---|---|
| **Top performers** | 10 minutos a 1 hora | SOCs con SOAR y enriquecimiento automatizado |
| **Promedio aceptable** | 1 a 4 horas | SOCs con herramientas integradas |
| **Organizaciones con gaps** | 4 a 24 horas | Herramientas fragmentadas, investigación manual |

> **Advertencia**: para organizaciones con altas tasas de falsos positivos, hacer tuning de esas alertas puede aumentar artificialmente el MTTI porque los FP que antes se cerraban en 2 minutos (sin investigar realmente) desaparecen del cálculo, y solo quedan los verdaderos positivos que requieren investigación real. Eso no es malo — es una representación más honesta de la métrica.
{: .prompt-warning }

### Que incluye la investigación

```mermaid
graph TD
    INV["INVESTIGACION COMPLETA<br/>Las 4 fases del MTTI"]

    INV --> V["1. VALIDACION"]
    V --> V1["Confirmar que la alerta es un<br/>verdadero positivo y no un FP.<br/>Puede ser trivial: ese hash es Cobalt Strike.<br/>O complejo: esta conexion a un dominio<br/>legitimo es tunnel DNS o uso normal?"]

    INV --> A["2. DETERMINACION DE ALCANCE"]
    A --> A1["Cuantos sistemas estan afectados?<br/>El atacante se movio lateralmente?<br/>Hay otros beacons o implantes?<br/>Aqui el EDR con threat hunting,<br/>SIEM con correlacion cross-host,<br/>y NDR son fundamentales."]

    INV --> CR["3. ANALISIS DE CAUSA RAIZ"]
    CR --> CR1["Como entro el atacante?<br/>Que vector de acceso inicial uso?<br/>Que vulnerabilidad exploto?<br/>Que credenciales comprometio?"]

    INV --> IM["4. ASSESSMENT DE IMPACTO"]
    IM --> IM1["Se exfiltro informacion?<br/>Que datos estan en riesgo?<br/>Hay implicaciones regulatorias?<br/>Datos personales bajo GDPR o LGPD?"]

    style INV fill:#0d1117,stroke:#aa44ff,color:#aa44ff
    style V fill:#0d1117,stroke:#00b4ff,color:#00b4ff
    style A fill:#0d1117,stroke:#00e5ff,color:#00e5ff
    style CR fill:#0d1117,stroke:#ff8800,color:#ff8800
    style IM fill:#0d1117,stroke:#ff3355,color:#ff3355
    style V1 fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style A1 fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style CR1 fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style IM1 fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
```

### Walkthrough de investigación paso a paso

Para hacer tangible lo que ocurre durante el MTTI, se presenta un caso real completo. El SOC recibe una alerta del NDR: "Anomalous SMB traffic from WS-FIN-023 to FS-01 at 03:00 — 47GB transferred".

```mermaid
sequenceDiagram
    participant AN as Analista SOC
    participant SIEM as SIEM
    participant EDR as EDR
    participant AD as Active Directory
    participant VT as VirusTotal

    Note over AN,VT: MTTI comienza: 08:15

    AN->>SIEM: Paso 1 Validacion: historial de juan.perez accediendo a FS-01
    SIEM-->>AN: Sin historial de acceso fuera de horario. Verdadero Positivo.

    AN->>EDR: Paso 2 Endpoint: procesos sospechosos en WS-FIN-023
    EDR-->>AN: rundll32.exe ejecutando DLL desde AppData Local Temp
    AN->>VT: Hash de la DLL
    VT-->>AN: 0 detecciones. Muestra nueva o desconocida.

    AN->>SIEM: Paso 3 Pivoting: buscar hash o IP de C2 en otros hosts
    SIEM-->>AN: 2 hosts adicionales: WS-HR-011 y WS-IT-005. Alcance = 3 sistemas.

    AN->>AD: Paso 4 Causa raiz: autenticaciones anomalas
    AD-->>AN: Cuenta svc_backup usada contra FS-01. Kerberoasting confirmado.

    Note over AN,VT: 09:40 - Assessment documentado - MTTI = 1 hora 25 minutos

    AN->>AN: Assessment: 3 hosts comprometidos. Exfiltracion potencial 47GB. Vector = phishing a Cobalt Strike a Kerberoasting a exfiltracion SMB.
```

**Resultado del MTTI**: 1 hora 25 minutos. El analista pudo determinar el alcance (3 sistemas), la causa raíz (Kerberoasting de cuenta de servicio) y el impacto potencial (47GB de datos posiblemente exfiltrados). Toda esa información es necesaria para que la fase de contención sea efectiva.

### Factores que inflan el MTTI

| Factor | Por que infla el MTTI | Solución |
|---|---|---|
| **Falta de contexto automatizado** | El analista abre 5 consolas diferentes (SIEM, EDR, firewall, DNS, AD) y correlaciona manualmente. | SOAR que pre-enriquece alertas con IOCs, reputación y contexto del usuario. |
| **Logging insuficiente** | Sin Sysmon configurado, no se ven ejecuciones de DLLs sospechosas. El analista busca información que no existe. | Sysmon con reglas de SwiftOnSecurity u Olaf Hartong en todos los endpoints. |
| **Herramientas fragmentadas** | Analista usa 7 herramientas con 7 logins diferentes. El context switching consume tiempo. | Plataformas XDR o SIEM con integraciones nativas a EDR, NDR y threat intel. |
| **Falta de experiencia** | Un analista junior puede tardar 4 horas en lo que un senior hace en 30 minutos. | Playbooks detallados paso a paso, mentoring, y ejercicios regulares con CyberDefenders o similares. |
| **Alertas sin contexto** | La alerta dice "suspicious activity" sin detalles. El analista parte de cero. | Reglas de detección que incluyan contexto: usuario, host, proceso padre, comando ejecutado. |

### Perspectiva Red Team a Blue Team

Desde el lado ofensivo, el MTTI es la ventana donde el atacante todavía puede operar mientras el analista está "mirando". Si el atacante sabe que fue detectado (por ejemplo, porque ve que su beacon dejó de recibir callbacks), puede intentar pivotar a otro implante, escalar privilegios rápidamente, o destruir evidencia antes de que la investigación termine.

Desde el lado defensivo, el MTTI es la fase donde la calidad del trabajo determina todo lo que sigue. Una investigación incompleta lleva a una contención incompleta — si no se identifican todos los hosts comprometidos, los que se pierden siguen en manos del atacante.

---

## Las tres métricas juntas: el pipeline de detección y triaje

Para ver el impacto combinado de MTTD + MTTA + MTTI, se presenta un solo incidente medido en cada fase:

```mermaid
graph TD
    T0["09:00 - COMPROMISO INICIAL<br/>Cobalt Strike beacon activo"]

    T0 --> MTTD["MTTD = 3 minutos"]
    MTTD --> T1["09:03 - ALERTA GENERADA<br/>EDR detecta process hollowing"]

    T1 --> MTTA["MTTA = 12 minutos"]
    MTTA --> T2["09:15 - ANALISTA RECONOCE<br/>Abre la alerta y comienza triage"]

    T2 --> MTTI["MTTI = 45 minutos"]
    MTTI --> T3["10:00 - INVESTIGACION COMPLETA<br/>1 host comprometido, C2 identificado,<br/>sin movimiento lateral detectado"]

    T3 --> NEXT["LISTO PARA CONTENER<br/>Total: 1 hora desde compromiso"]

    T0 -.->|"Breakout Time promedio eCrime: 29 min"| BT["BREAKOUT A LAS 09:29<br/>Si el atacante es promedio,<br/>ya hizo movimiento lateral"]

    style T0 fill:#1a0a0a,stroke:#ff3355,color:#ff3355
    style T1 fill:#0d1117,stroke:#00b4ff,color:#00b4ff
    style T2 fill:#0d1117,stroke:#00e5ff,color:#00e5ff
    style T3 fill:#0d1117,stroke:#aa44ff,color:#aa44ff
    style NEXT fill:#0a1a0a,stroke:#00ff88,color:#00ff88
    style MTTD fill:#0d1117,stroke:#00b4ff,color:#00b4ff
    style MTTA fill:#0d1117,stroke:#00e5ff,color:#00e5ff
    style MTTI fill:#0d1117,stroke:#aa44ff,color:#aa44ff
    style BT fill:#1a0a0a,stroke:#ff3355,color:#ff3355
```

En este ejemplo, el equipo completó su pipeline de detección y triaje en **1 hora**. Pero el Breakout Time promedio de eCrime fue de 29 minutos — lo que significa que a las 09:29, el atacante ya se movió lateralmente, y cuando la investigación termina a las 10:00, el alcance real puede ser mayor que el detectado inicialmente.

> **La implicación operativa es directa**: incluso con un MTTD excelente de 3 minutos, si el MTTA + MTTI suman más de 26 minutos, el atacante completa su breakout antes de que el SOC tenga la imagen completa. Esto es lo que hace que la automatización del triage (SOAR) y la contención automatizada (auto-isolation en EDR) sean cada vez más necesarias.
{: .prompt-danger }

---

## Errores comunes al medir estas métricas

Antes de cerrar, es importante señalar las trampas más frecuentes en la medición de MTTD, MTTA y MTTI:

```mermaid
graph TD
    ERR["ERRORES COMUNES DE MEDICION"]

    ERR --> E1["Confundir hora de alerta<br/>con hora de deteccion"]
    E1 --> E1d["Que se dispare una alerta no significa<br/>que alguien la haya visto. La alerta<br/>puede estar en cola durante horas.<br/>Eso es MTTA, no MTTD."]

    ERR --> E2["Timestamps desincronizados"]
    E2 --> E2d["Logs de diferentes sistemas con<br/>relojes fuera de sync pueden hacer<br/>que el MTTD aparezca como negativo<br/>o artificialmente bajo. NTP obligatorio."]

    ERR --> E3["Medir volumen de alertas<br/>en lugar de incidentes"]
    E3 --> E3d["Una sola intrusion puede generar<br/>50 alertas. Medir MTTD por alerta<br/>en lugar de por incidente distorsiona<br/>la metrica completamente."]

    ERR --> E4["Ignorar los incidentes<br/>no detectados"]
    E4 --> E4d["El MTTD solo incluye incidentes<br/>que se detectaron. Los que nunca<br/>se detectaron tienen MTTD infinito<br/>y no aparecen en la estadistica."]

    ERR --> E5["Usar promedio en lugar<br/>de mediana"]
    E5 --> E5d["Un solo incidente de espionaje con<br/>MTTD de 180 dias distorsiona el<br/>promedio de todo el trimestre.<br/>La mediana es mas resistente a outliers."]

    style ERR fill:#1a0a0a,stroke:#ff3355,color:#ff3355
    style E1 fill:#0d1117,stroke:#ff8800,color:#ff8800
    style E2 fill:#0d1117,stroke:#ff8800,color:#ff8800
    style E3 fill:#0d1117,stroke:#ff8800,color:#ff8800
    style E4 fill:#0d1117,stroke:#ff8800,color:#ff8800
    style E5 fill:#0d1117,stroke:#ff8800,color:#ff8800
    style E1d fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style E2d fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style E3d fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style E4d fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style E5d fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
```

---

## Resumen del grupo de detección y triaje

| Métrica | Que mide | Segmento | Benchmark objetivo |
|---|---|---|---|
| **MTTD** | Compromiso hasta primera alerta | T0 a T1 | Menos de 1 hora (critico), menos de 4 horas (general) |
| **MTTA** | Alerta hasta que un analista la acepta | T1 a T2 | Menos de 5 min (SEV-1), menos de 20 min (SEV-3) |
| **MTTI** | Acknowledge hasta investigación completa | T2 a T3 | 10 min a 1 hora (top), 1 a 4 horas (aceptable) |
| **Suma** | Compromiso hasta assessment completo | T0 a T3 | Debe ser menor al Breakout Time del adversario |

La próxima entrada de esta serie cubrirá el segundo grupo de métricas: **MTTC y MTTR** — las métricas de contención y recuperación que determinan cuánto tiempo toma detener al atacante y restaurar las operaciones.

---

*Fuentes: SANS 2023 IR Survey, SANS 2025 SOC Survey, IBM Cost of a Data Breach Report 2025, CrowdStrike 2026 Global Threat Report, AI SOC Market Landscape 2025, Osterman Research Report 2025, Prophet Security SOC Metrics 2026, MetricFire MTTR/MTTA/MTTD Guide 2026, Rapid7 MTTD Fundamentals.*
