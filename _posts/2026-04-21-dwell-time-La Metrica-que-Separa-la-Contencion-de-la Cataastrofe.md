---
layout: post
title: "Dwell Time: La Métrica que Separa la Contención de la Catástrofe."
date: 2026-04-21
categories: [blueteam, incident-response]
tags: [dwell-time, metricas, ir, mandiant, mtrends, dfir]
mermaid: true
---

## Lo que pasa antes de que alguien se entere

Toda brecha tiene un secreto incómodo: siempre existe un hueco temporal entre el momento en que un atacante entra y el momento en que alguien del lado defensivo se da cuenta. Ese hueco tiene nombre — **Dwell Time** (Tiempo de Permanencia) — y es la métrica estratégica más importante en Incident Response.

El Dwell Time es el período total que un adversario permanece dentro de un entorno comprometido **antes de ser detectado**. No antes de ser contenido. No antes de ser erradicado. Antes de que alguien siquiera *sepa* que está ahí.

```mermaid
timeline
    title Dwell Time — Lo que pasa en las sombras
    section Actividad del atacante - Invisible
        Dia 0  : Acceso inicial via VPN explotada sin parche
        Dia 1  : Reconocimiento interno con whoami y net group
        Dia 3  : Kerberoasting y cuenta de servicio crackeada
        Dia 5  : Movimiento lateral al file server
        Dia 7  : Staging de datos y 47GB comprimidos
        Dia 9  : Exfiltracion via HTTPS a cloud storage
    section Deteccion - Visible
        Dia 11 : Alerta EDR por trafico SMB anomalo
        Dia 11 : SOC reconoce e investiga
        Dia 11 : Contencion y hosts aislados
```

Todo lo que está arriba de la línea de detección ocurrió en **completo silencio**. Sin alertas. Sin tickets. Sin conciencia de lo que pasaba. Eso es Dwell Time.

---

## La fórmula

El cálculo en sí es simple. Lo difícil es determinar `T0`:

```
Dwell Time = Fecha_de_Detección − Fecha_de_Compromiso_Inicial
```

El problema práctico es que la `Fecha de Compromiso Inicial` casi siempre se determina **retroactivamente** durante el análisis forense. El analista tiene que reconstruir la línea temporal buscando la evidencia más temprana de actividad maliciosa en logs, artefactos de disco y capturas de red. Si el logging era insuficiente, `T0` quizás nunca se determine con precisión — lo que significa que el Dwell Time real probablemente fue **aún mayor** que el reportado.

> **Nota importante**: el Dwell Time se reporta como **mediana**, no como promedio. Esto es porque la distribución es extremadamente asimétrica — unas pocas operaciones de espionaje con Dwell Times de meses distorsionan cualquier promedio aritmético.
{: .prompt-info }

---

## Por qué importa — La curva de daño

El Dwell Time no es solo un número para dashboards ejecutivos. Correlaciona directamente con **cuánto daño puede causar el atacante**. A mayor tiempo sin ser detectado, más avanza en la cadena de ataque:

```mermaid
graph TD
    A1["Acceso Inicial"] --> A2["Reconocimiento Local"]
    A2 --> B1["Robo de Credenciales"]
    B1 --> B2["Escalacion de Privilegios"]
    B2 --> C1["Movimiento Lateral"]
    C1 --> C2["Descubrimiento de Datos"]
    C2 --> D1["Exfiltracion"]
    D1 --> D2["Ransomware o Destruccion"]

    A1 -.- T1["Primeras horas"]
    B1 -.- T2["Dias 1 a 3"]
    C1 -.- T3["Dias 3 a 7"]
    D1 -.- T4["Dia 7 en adelante"]

    style A1 fill:#1a1a2e,stroke:#00b4ff,color:#00e5ff
    style A2 fill:#1a1a2e,stroke:#00b4ff,color:#00e5ff
    style B1 fill:#1a1a2e,stroke:#ff8800,color:#ffcc00
    style B2 fill:#1a1a2e,stroke:#ff8800,color:#ffcc00
    style C1 fill:#1a1a2e,stroke:#ff5500,color:#ff8800
    style C2 fill:#1a1a2e,stroke:#ff5500,color:#ff8800
    style D1 fill:#1a1a2e,stroke:#ff3355,color:#ff3355
    style D2 fill:#1a1a2e,stroke:#ff3355,color:#ff3355
    style T1 fill:none,stroke:none,color:#556677
    style T2 fill:none,stroke:none,color:#556677
    style T3 fill:none,stroke:none,color:#556677
    style T4 fill:none,stroke:none,color:#556677
```

Un Dwell Time de **3 horas** significa que el atacante probablemente tiene foothold en una máquina y algo de reconocimiento local. Contener en esta fase es relativamente limpio: aislar el host, resetear la credencial comprometida, parchar la vulnerabilidad. Listo.

Un Dwell Time de **11 días** (la mediana global de 2024) significa que el atacante probablemente completó toda la kill chain: robo de credenciales, movimiento lateral a múltiples sistemas, descubrimiento de datos y potencialmente exfiltración completa. Contener en esta fase es un engagement de IR completo — imágenes forenses de docenas de máquinas, rotación total de credenciales, threat hunting en toda la red y semanas de recuperación.

> **La regla es simple**: cada día adicional de Dwell Time multiplica exponencialmente el esfuerzo de respuesta y el impacto al negocio.
{: .prompt-warning }

---

## Los números: datos de M-Trends (2014–2025)

La fuente más confiable para datos de Dwell Time es el reporte **M-Trends de Mandiant** (Google), publicado anualmente basándose en cientos de investigaciones de IR reales.

La tendencia histórica:

```mermaid
xychart-beta
    title "Mediana Global de Dwell Time en dias - Mandiant M-Trends"
    x-axis [2014, 2016, 2018, 2019, 2020, 2021, 2022, 2023, 2024, 2025]
    y-axis "Dias" 0 --> 220
    bar [205, 146, 78, 56, 24, 21, 16, 10, 11, 14]
```

La tendencia a largo plazo es indiscutiblemente positiva — una **reducción del 96%**, de 205 días en 2014 a 11 días en 2024. Pero el ligero repunte en 2025 (a 14 días) cuenta una historia importante que se analiza más abajo.

---

## Las tres fuentes de detección

No todos los Dwell Times son iguales. La forma en que se descubre la intrusión cambia radicalmente el número y lo que implica:

```mermaid
graph TD
    I["Intrusion Detectada"]
    I --> INT
    I --> EXT
    I --> ADV

    INT["DETECCION INTERNA<br/>Mediana: 10 dias"]
    INT --> INT_D["El SOC o equipo interno<br/>descubrio la actividad.<br/>Indica madurez defensiva."]

    EXT["NOTIFICACION EXTERNA<br/>Mediana: 26 dias"]
    EXT --> EXT_D["Un tercero aviso:<br/>agencia de ley, vendor o CERT.<br/>No habia visibilidad propia."]

    ADV["NOTIFICACION DEL ADVERSARIO<br/>Mediana: 5 dias"]
    ADV --> ADV_D["El atacante aviso:<br/>nota de ransomware desplegada.<br/>Ya cumplio su objetivo."]

    style I fill:#0d1117,stroke:#00b4ff,color:#00b4ff
    style INT fill:#0d1117,stroke:#00ff88,color:#00ff88
    style EXT fill:#0d1117,stroke:#ff8800,color:#ff8800
    style ADV fill:#0d1117,stroke:#ff3355,color:#ff3355
    style INT_D fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style EXT_D fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style ADV_D fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
```

### Desglose por fuente (datos M-Trends 2025, investigaciones de 2024)

| Fuente de detección | Mediana | Lo que implica |
|---|---|---|
| **Detección interna** | 10 días | La organización tiene visibilidad. El SOC funciona. |
| **Notificación externa** | 26 días | Un tercero (LE, vendor, CERT) avisó. Los datos aparecieron en un mercado clandestino o la inteligencia de terceros detectó la actividad. |
| **Notificación del adversario** | 5 días | El atacante avisó (ransom note). Es el más rápido, pero por la peor razón: ya completó su objetivo. |
| **Ransomware específico** | 6 días | El modelo de extorsión requiere velocidad. El atacante no quiere permanecer — quiere cobrar. |
| **Espionaje / NK IT workers** | 122 días | Operaciones diseñadas para persistencia a largo plazo. Algunos casos superaron el año sin detección. |

> **Dato clave del M-Trends 2026**: el repunte a 14 días en 2025 fue causado por el alto volumen de investigaciones de espionaje y operaciones de trabajadores IT de Corea del Norte (mediana de 122 días), que arrastraron la mediana global hacia arriba.
{: .prompt-tip }

---

## Por que aumentó en 2025

Este punto merece atención especial porque parece contradictorio — ¿cómo puede subir el Dwell Time si las herramientas de detección son cada vez mejores?

```mermaid
graph TD
    subgraph RANSOM["Ransomware - Dwell Time baja"]
        R1["Breakout en 29 min"]
        R1 --> R2["Cifrado en menos de 7 dias"]
        R2 --> R3["Mediana: 6 dias"]
    end

    subgraph ESPIA["Espionaje - Dwell Time sube"]
        E1["Infiltracion silenciosa"]
        E1 --> E2["IT workers con identidad falsa"]
        E2 --> E3["Persistencia mayor a 1 anio"]
        E3 --> E4["Mediana: 122 dias"]
    end

    R3 --> M["MEDIANA GLOBAL<br/>14 dias"]
    E4 --> M

    style R3 fill:#0d1117,stroke:#00ff88,color:#00ff88
    style E4 fill:#0d1117,stroke:#ff3355,color:#ff3355
    style M fill:#0d1117,stroke:#ff8800,color:#ff8800
    style R1 fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style R2 fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style E1 fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style E2 fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style E3 fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
```

La respuesta está en la **composición del caseload**. En 2025, Mandiant investigó un volumen alto de operaciones de espionaje y amenazas internas (trabajadores IT norcoreanos usando identidades fabricadas para conseguir empleo en empresas occidentales). Estas operaciones están diseñadas específicamente para **no ser detectadas** — no despliegan ransomware, no generan ruido, no dejan artefactos obvios. Operan dentro del alcance de sus responsabilidades laborales legítimas, lo que las hace extremadamente difíciles de distinguir de la actividad normal.

Mientras tanto, los incidentes de ransomware (que tienen Dwell Times mucho más cortos) siguieron bajando. Pero la mezcla de muchos casos de espionaje con Dwell Times de 100+ días elevó la mediana global de 11 a 14.

---

## Dwell Time no es un solo número

Esta es una de las trampas más comunes al usar esta métrica. El Dwell Time no debe tratarse como un valor único y uniforme. Hay que segmentarlo:

```mermaid
graph TD
    DT["DWELL TIME<br/>Mediana global: 14 dias"]
    DT --> S1["Por tipo de amenaza<br/>Ransomware vs Espionaje vs Insider"]
    DT --> S2["Por fuente de deteccion<br/>Interna vs Externa vs Adversario"]
    DT --> S3["Por industria<br/>Finanzas vs Salud vs Gobierno"]
    DT --> S4["Por region geografica<br/>JAPAC: 9 dias - EMEA: 22 dias"]

    style DT fill:#0d1117,stroke:#00b4ff,color:#00b4ff
    style S1 fill:#0d1117,stroke:#00e5ff,color:#00e5ff
    style S2 fill:#0d1117,stroke:#00e5ff,color:#00e5ff
    style S3 fill:#0d1117,stroke:#00e5ff,color:#00e5ff
    style S4 fill:#0d1117,stroke:#00e5ff,color:#00e5ff
```

Por ejemplo, según M-Trends 2025:
- **JAPAC** (Asia-Pacífico y Japón) tuvo la detección más rápida con una mediana de **9 días**.
- **EMEA** (Europa, Medio Oriente y África) tuvo la más lenta con **22 días**, en parte debido a un incremento en actividad de ransomware en la región.

Si una organización reporta "nuestro Dwell Time promedio es 15 días" sin segmentar, ese número puede esconder un incidente de ransomware detectado en 2 días y un compromiso de espionaje que llevaba 6 meses sin detectarse.

---

## Perspectiva Red Team a Blue Team

Este es el puente que conecta la mentalidad ofensiva con la defensiva y que ayuda a entender por qué el Dwell Time importa desde ambos lados.

### Desde el Red Team

Para el atacante, el Dwell Time es su **recurso más valioso**. Todo lo que necesita hacer después del acceso inicial requiere tiempo:

```mermaid
sequenceDiagram
    participant A as Atacante
    participant AD as Active Directory
    participant FS as File Server
    participant C2 as C2 Server

    Note over A,C2: Dwell Time = Todo este periodo sin deteccion

    A->>AD: Dia 1 - Reconocimiento con BloodHound
    AD-->>A: Mapa completo de relaciones y privilegios
    A->>AD: Dia 3 - Kerberoasting de TGS tickets
    AD-->>A: Hash de svc_backup crackeado offline
    A->>AD: Dia 4 - DCSync con credenciales de servicio
    AD-->>A: NTLM hashes de todo el dominio
    A->>FS: Dia 5-7 - Acceso a file server con creds elevadas
    FS-->>A: Datos financieros y contratos y PII
    A->>C2: Dia 8-10 - Exfiltracion gradual via HTTPS
    C2-->>A: 47GB recibidos
    A->>FS: Dia 11 - Ransomware desplegado

    Note over A,C2: El SOC se entera AHORA
```

Un operador de Red Team competente puede hacer **todo lo anterior** en un Dwell Time de 11 días. BloodHound, Kerberoasting, DCSync, movimiento lateral con PsExec o WMI — cada técnica del playbook de post-explotación consume tiempo, y 11 días es una cantidad *enorme* para alguien que sabe lo que hace.

### Desde el Blue Team

Cada día de Dwell Time tiene un costo acumulativo para el equipo defensivo:

| Día de Dwell Time | Lo que significa para el Blue Team |
|---|---|
| **Día 1** | 1 host comprometido. 1 credencial que rotar. Contención simple. |
| **Día 3** | Posible escalación de privilegios. Hay que verificar integridad de AD. |
| **Día 5** | Movimiento lateral probable. Alcance desconocido. Hay que hacer threat hunting en toda la red. |
| **Día 7** | Posible exfiltración. Hay que involucrar a legal, compliance y potencialmente a reguladores. |
| **Día 11** | Compromiso total probable. Rotación completa de credenciales, rebuild de sistemas, semanas de forensics. Escenario de crisis. |

> **La conclusión es directa**: reducir el Dwell Time no es un objetivo de mejora continua abstracto. Es la diferencia literal entre un incidente contenido en una máquina y una catástrofe organizacional.
{: .prompt-danger }

---

## Que factores afectan el Dwell Time

El Dwell Time no es una métrica que se reduzca con un solo cambio. Es el resultado de múltiples capacidades defensivas (o la falta de ellas):

```mermaid
mindmap
  root(("DWELL TIME"))
    Deteccion
      Cobertura de EDR en endpoints
      Reglas de deteccion en SIEM
      Monitoreo de red con NDR
      Deteccion de comportamiento con UEBA
      Threat Hunting proactivo
    Visibilidad
      Logging centralizado
      Cobertura de Active Directory
      Visibilidad red este-oeste
      Monitoreo cloud y SaaS
    Proceso
      Turnos del SOC 8x5 vs 24x7
      Playbooks de investigacion
      Escalamiento definido
      Integracion de threat intel
    Personas
      Experiencia de analistas
      Rotacion y burnout
      Training continuo
      Ratio alertas por analista
```

La realidad es que una organización con un EDR de última generación pero sin logging centralizado, o con un SIEM perfecto pero un SOC que solo opera 8x5, seguirá teniendo Dwell Times altos. Cada componente es necesario pero no suficiente por sí solo.

---

## Ejemplo práctico: mismo ataque, tres organizaciones

Para ilustrar cómo el Dwell Time varía según la madurez defensiva, se presenta el mismo escenario de ataque contra tres organizaciones con niveles de madurez diferentes.

**Escenario**: un atacante explota una vulnerabilidad en el VPN (CVE-2024-21887 en Ivanti Connect Secure), instala un webshell y comienza reconocimiento interno.

```mermaid
graph TD
    ATK["MISMO ATAQUE<br/>Exploit VPN Ivanti - Webshell - Recon"]

    ATK --> ORG1_H
    ATK --> ORG2_H
    ATK --> ORG3_H

    ORG1_H["ORG A: MADUREZ BAJA"]
    ORG1_H --> O1A["Sin EDR"]
    O1A --> O1B["Sin regla para webshells"]
    O1B --> O1C["SOC 8x5"]
    O1C --> O1D["Dwell Time: 45 dias<br/>Detectado por notificacion externa"]

    ORG2_H["ORG B: MADUREZ MEDIA"]
    ORG2_H --> O2A["EDR basico"]
    O2A --> O2B["SIEM con reglas genericas"]
    O2B --> O2C["SOC 12x5"]
    O2C --> O2D["Dwell Time: 8 dias<br/>SIEM detecto beaconing C2"]

    ORG3_H["ORG C: MADUREZ ALTA"]
    ORG3_H --> O3A["EDR avanzado y NDR"]
    O3A --> O3B["Regla Sigma para webshells Ivanti"]
    O3B --> O3C["SOC 24x7 y Threat Hunting"]
    O3C --> O3D["Dwell Time: 4 horas<br/>EDR detecto ejecucion sospechosa"]

    style ATK fill:#0d1117,stroke:#00b4ff,color:#00b4ff
    style ORG1_H fill:#1a0a0a,stroke:#ff3355,color:#ff3355
    style ORG2_H fill:#1a1a0a,stroke:#ff8800,color:#ff8800
    style ORG3_H fill:#0a1a0a,stroke:#00ff88,color:#00ff88
    style O1D fill:#1a0a0a,stroke:#ff3355,color:#ff3355
    style O2D fill:#1a1a0a,stroke:#ff8800,color:#ff8800
    style O3D fill:#0a1a0a,stroke:#00ff88,color:#00ff88
    style O1A fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style O1B fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style O1C fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style O2A fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style O2B fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style O2C fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style O3A fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style O3B fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
    style O3C fill:#0d1117,stroke:#1a2a3a,color:#c0d0e0
```

La diferencia entre 45 días y 4 horas no es suerte — es inversión en capacidades de detección, reglas específicas para amenazas relevantes y cobertura operativa del SOC.

---

## Relación con otras métricas TTx

El Dwell Time no existe en aislamiento. Es el resultado de otra métrica clave — el **MTTD** (Mean Time To Detect) — y alimenta todo lo que viene después:

```mermaid
graph TD
    DT["DWELL TIME<br/>Compromiso hasta Deteccion"]
    DT -->|"es igual o mayor que"| MTTD["MTTD<br/>Mean Time To Detect"]
    MTTD -->|"dispara"| MTTA["MTTA<br/>Mean Time To Acknowledge"]
    MTTA -->|"inicia"| MTTI["MTTI<br/>Mean Time To Investigate"]
    MTTI -->|"habilita"| MTTC["MTTC<br/>Mean Time To Contain"]
    MTTC -->|"permite"| MTTR["MTTR<br/>Mean Time To Recover"]

    style DT fill:#0d1117,stroke:#ff3355,color:#ff3355
    style MTTD fill:#0d1117,stroke:#00b4ff,color:#00b4ff
    style MTTA fill:#0d1117,stroke:#00e5ff,color:#00e5ff
    style MTTI fill:#0d1117,stroke:#aa44ff,color:#aa44ff
    style MTTC fill:#0d1117,stroke:#ff8800,color:#ff8800
    style MTTR fill:#0d1117,stroke:#00ff88,color:#00ff88
```

La relación matemática clave:

```
Dwell Time >= MTTD (siempre)
```

Si se reduce el MTTD a minutos, el Dwell Time se comprime automáticamente. Esto reduce drásticamente el impacto del incidente porque se interrumpe al atacante en las fases tempranas de la kill chain — antes de que haga credential theft, antes del movimiento lateral, antes de la exfiltración.

---

## Como reducirlo: acciones concretas

No se trata de comprar una herramienta mágica. La reducción del Dwell Time requiere mejoras en múltiples frentes simultáneamente:

| Área | Acción concreta | Impacto en Dwell Time |
|---|---|---|
| **Logging** | Habilitar Sysmon con config de SwiftOnSecurity en todos los endpoints. Centralizar eventos de AD (4624, 4625, 4672, 4768, 4769). | Elimina puntos ciegos donde el atacante opera sin dejar trazas. |
| **Detección** | Implementar reglas Sigma mapeadas a MITRE ATT&CK para las TTPs que los threat actors de la industria realmente usan. | Pasa de buscar IOCs conocidos a detectar comportamientos maliciosos — los IOCs cambian, los comportamientos no. |
| **Tuning** | Reducir falsos positivos agresivamente. Si el 90% de las alertas son FP, la alerta real se pierde en el ruido. | Reduce MTTA indirectamente: menos ruido significa que las alertas reales son más visibles. |
| **Cobertura** | Extender el SOC de 8x5 a 16x5 o 24x7 (incluso con MDR externo si no hay presupuesto para equipo propio). | Elimina la ventana de 16 horas nocturnas donde nadie mira las alertas. |
| **Threat Hunting** | Ejecutar hunts periódicos buscando TTPs específicas sin depender de alertas. | Encuentra lo que las reglas automatizadas no detectan — el delta entre lo que se puede detectar y lo que realmente hay. |

---

## Conclusión

El Dwell Time es la métrica que convierte la seguridad en algo medible y accionable. No es un número para presentaciones ejecutivas — es un indicador directo de cuánto daño puede causar un atacante antes de que el equipo defensivo siquiera sepa que hay un problema.

Los datos de Mandiant M-Trends muestran una mejora sostenida a lo largo de la última década (de 205 a 11 días), pero también revelan que el número global esconde realidades muy diferentes según el tipo de amenaza, la región y la madurez de la organización.

La próxima entrada de esta serie cubrirá las métricas **MTTD, MTTA y MTTI** — los segmentos temporales que componen el Dwell Time y que el SOC puede medir y optimizar directamente.

---

*Fuentes: Mandiant M-Trends 2025 Report (abril 2025), Mandiant M-Trends 2026 Report (abril 2026), CrowdStrike 2026 Global Threat Report (febrero 2026).*
