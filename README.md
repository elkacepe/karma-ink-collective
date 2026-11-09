# Karma Ink Collective · Agenda

Webapp de agenda para un colectivo de cuatro tatuadores. Arma la sesión a partir de una cotización y lleva una **lista previa**: anticipo registrado y referencias revisadas. Así el artista no se entera el mismo día de que tiene que rediseñar la pieza.

Pensada primero para el celular: funciona desde 360 px de ancho.

**Previsualización:** <https://karma-ink-collective.vercel.app/>

Está conectada a este repositorio en Vercel: cada push a `main` se publica solo. Los datos viven en el navegador de cada visitante (localStorage), así que lo que reserva una persona no lo ve el panel desde otro dispositivo hasta conectar el backend.

---

## 1. Cómo arrancar el proyecto

### Requisitos

- Un navegador actual (Chrome, Safari, Firefox o Edge).
- **No hay dependencias que instalar.** El proyecto no tiene `package.json`, `node_modules` ni paso de compilación.
- **Node no es necesario.** Solo lo usás si elegís levantar el servidor local con `npx` (cualquier Node ≥ 18 sirve). Python 3 funciona igual de bien.
- Conexión a internet: React, Tailwind y las tipografías se cargan desde CDN.

### Levantar en local

Desde la raíz del repositorio, cualquiera de estas dos opciones:

```bash
# Opción A: Python 3
python3 -m http.server 8080

# Opción B: Node
npx serve -l 8080 .
```

Después abrí <http://localhost:8080>.

También funciona abriendo `index.html` directo con doble clic. Igual conviene usar un servidor, porque algunos navegadores limitan localStorage en `file://`.

### Stack

| Pieza | Versión | Cómo se carga |
|---|---|---|
| React + ReactDOM | 18.3.1 (UMD) | cdnjs |
| [htm](https://github.com/developit/htm) | 3.1.1 | jsDelivr. Permite escribir JSX en template strings sin compilar |
| Tailwind CSS | 3.4.17 (Play CDN) | cdn.tailwindcss.com |
| Syne 700 / Inter | — | Google Fonts |
| Persistencia | localStorage | — |

### Dónde está cada cosa

```
karma-ink-collective/
├── index.html   ← toda la app (HTML, estilos, config de Tailwind y código React)
└── README.md
```

No hay carpetas: la app entera vive en `index.html`, dividida en bloques marcados con comentarios `/* ---------- … ---------- */`.

| Líneas | Bloque | Contenido |
|---|---|---|
| 11–33 | `<style>` | Tokens de color y tipografía, gutter seguro para móvil |
| 34–45 | Scripts | Tailwind + `tailwind.config` (colores `paper`, `ink`, `violet`, `lilac`, `stone`, `jade`), React, ReactDOM, htm |
| 57–75 | **Catálogo** | `ARTISTS`, `SIZES` y `ART` (bloques de color de cada artista) |
| 77–81 | **Configuración del estudio** | `ADMIN_KEY` (clave del panel) y `STUDIO_WHATSAPP` (número opcional del estudio) |
| 83–160 | **Reglas de negocio** | Constantes de plazos, estados, `suggest` (anticipo), `isReady` (lista previa), `conflicts` (cupo), `daySlots` (horarios ofrecidos), `sweep` (vencimiento a 48 h) |
| 162–236 | **Datos de ejemplo** | `seed()` y su buscador de huecos `place()` |
| 238–248 | **Persistencia** | `loadDB`, `saveDB`, `loadMe`, `saveMe`, `loadAdmin`, `saveAdmin` |
| 250–313 | **Compartir y calendario** | `shareText` y `whatsappUrl` (WhatsApp), `buildIcs` y `downloadIcs` (archivo .ics) |
| 315–378 | Piezas de UI | `PrepTag` (etiqueta Lista / Le falta algo), `StatusPill`, `Header`, `Footer`, `Toast` |
| 380–439 | Pantalla | `Home`: inicio |
| 441–596 | Pantalla | `Cotizar`: artista → tamaño → día y horario → datos |
| 598–654 | Pantalla | `Listo`: código, compartir por WhatsApp, descargar .ics y pendientes |
| 656–798 | Pantalla | `Login` y `ClientSession`: Mi sesión |
| 800–1045 | Pantalla | `Admin`, `SessionCard` y `DetailSheet`: panel del estudio |
| 1047–1067 | Pantalla | `AdminGate`: pide la clave antes de abrir el panel |
| 1069–1127 | App | Router por hash, estado global, `update`, `onBook`, `resetDemo` |

Los números de línea son de este commit. Si cambian, buscá por nombre de función.

### Clave del panel

El panel `#admin` pide una clave antes de abrirse. La clave de ejemplo es:

```
karma-2026
```

**Cómo cambiarla:** abrí `index.html`, buscá `ADMIN_KEY` (línea 79) y reemplazá el texto entre comillas:

```js
const ADMIN_KEY = 'karma-2026';   // ← poné la clave nueva acá
```

Guardá, subí el cambio a `main` y Vercel publica la versión nueva solo. Quien tenga el panel abierto lo sigue viendo hasta cerrar la pestaña; la próxima vez le pide la clave nueva.

Una vez que entrás, el panel queda abierto hasta cerrar la pestaña (se guarda en sessionStorage, clave `karma-ink-panel`). El botón «Cerrar panel» de la cabecera lo bloquea antes.

> **Ojo:** es una clave simple, como pide el encargo. Está escrita en el código del navegador, así que cualquiera que mire el código fuente la puede leer. Sirve para que el panel no quede a la vista, no como seguridad real. Al conectar el backend, la clave tiene que validarse en el servidor.

### Rutas

| Ruta | Para quién | Qué hace |
|---|---|---|
| `#inicio` | Público | Presentación, artistas, cómo funciona y tamaños |
| `#cotizar` | Público | Arma la cotización y aparta el turno |
| `#listo` | Público | Muestra el código, compartir por WhatsApp, descarga .ics y lo que falta |
| `#mi-sesion` | Público | Volver con celular + código: lista previa, referencias, cancelar |
| `#admin` | Estudio | Pide la clave. Después: agenda por día y artista, «Por revisar», detalle de sesión. Equivale a `/admin` |

---

## 2. Qué está simulado y cómo se comporta

| Qué | Cómo se comporta hoy | Punto de conexión |
|---|---|---|
| **Base de datos** | Todo se guarda en localStorage, clave `karma-ink-agenda-v2`. Solo comparten datos quienes usan el mismo navegador. Dos pestañas abiertas se sincronizan por el evento `storage`. | `loadDB` / `saveDB` (l. 239–243) y el listener `storage` en `App` (l. 1088) |
| **Crear reserva** | Comprueba choques contra los datos locales y guarda. | `onBook` en `App` (l. 1096). El servidor debe repetir `conflicts` de forma atómica |
| **Cobro del anticipo** | **No hay pago ni tarjeta en ningún lado.** La app solo sugiere el monto. El estudio lo registra a mano en `#admin` con «Registrar», con el monto que quiera. | `registerDeposit` en `DetailSheet` (l. 832) |
| **Avisos a la persona** | **La app no envía ningún correo, WhatsApp ni SMS por su cuenta.** El código se muestra en pantalla con botón «Copiar». La pantalla `#listo` le dice a la persona que pague por fuera de la app y coordine con el estudio cómo hacerlo. | Textos en `Listo` (l. 616). Si se automatiza un aviso, se dispara desde `onBook` |
| **Compartir por WhatsApp** | El botón abre `https://wa.me/?text=…` en una pestaña nueva con el texto ya armado: artista, tamaño, día y hora, código, anticipo y enlace a «Mi sesión». **No se manda nada solo:** WhatsApp abre el chat y la persona elige a quién mandarlo y toca enviar. Con `STUDIO_WHATSAPP` vacío, WhatsApp le deja elegir el chat; si cargás el número del estudio, abre directo ese chat. | `shareText` y `whatsappUrl` (l. 251–266). Número del estudio en `STUDIO_WHATSAPP` (l. 81) |
| **Calendario (.ics)** | El botón «Agregar a mi calendario» genera el archivo `karma-ink-KI-XXXX.ics` en el navegador y lo descarga. Lo abren Google Calendar, Calendario de Apple y Outlook. Trae la hora de la sesión (sin la preparación), el código, el anticipo y un recordatorio 24 h antes. **No se manda ninguna invitación por correo.** | `buildIcs` y `downloadIcs` (l. 280–313) |
| **Vencimiento a 48 h** | Lo calcula `sweep` en el navegador al abrir la app y luego cada 60 s. No hay tarea programada. | `sweep` (l. 151) e intervalo en `App` (l. 1087). En producción, mover a un job del servidor |
| **Identidad de la persona** | Sin cuentas. Se valida celular + código contra los datos locales. La sesión abierta se recuerda en el dispositivo (clave `karma-ink-mi-sesion`) hasta tocar «Salir». | `Login` (l. 657) y `loadMe` / `saveMe` |
| **Acceso al panel** | Clave simple guardada en el código (`karma-2026`). Se compara en el navegador, así que no es seguridad real: ver «Clave del panel» arriba. | `ADMIN_KEY` (l. 79) y `AdminGate` (l. 1048). En producción, validar en el servidor |
| **Referencias** | Solo un enlace de texto. No se suben archivos ni se valida que el enlace exista. | `saveLink` en `ClientSession` (l. 703) |
| **Reloj** | Usa la hora real del dispositivo y su zona horaria. Los horarios de los artistas se interpretan en esa zona. | `Date.now()` en `daySlots`, `sweep` y pantallas |

---

## 3. Datos de ejemplo y cómo reiniciarlos

### Qué viene precargado

**15 sesiones de 15 clientes distintos: 12 en las próximas dos semanas y 3 pasadas**, repartidas entre los 4 artistas. Entre todas cubren los seis estados. Todos los nombres, celulares, montos y enlaces son inventados.

Las fechas se generan **relativas al momento en que se cargan los datos**, así que la agenda siempre se ve vigente. Cada sesión se ubica con un buscador de huecos (`place()` dentro de `seed()`). Respeta el día y horario de cada artista, el cupo y los 30 min de preparación, así que nunca hay superposiciones. Se probó simulando cargas en los siete días de la semana y a distintas horas.

**Próximas dos semanas (12)**

| Código | Cliente | Artista | Tamaño | Cuándo | Estado al cargar | Lista previa |
|---|---|---|---|---|---|---|
| KI-R4MS | Martina Salas | El primero que atienda en las próximas 72 h | Mediano | En menos de 72 h | Confirmada ($1.100) | **Lista** |
| KI-V7QC | Valentina Ochoa | Emiliano | Grande | ~3 días | Confirmada ($2.100) | **Lista** |
| KI-Q8RN | Renata Quiroga | Regina | Mediano | ~4 días | Confirmada ($1.100) | **Lista** |
| KI-J8PD | Joaquín Pardo | Santiago | Chico | ~5 días | Confirmada ($500) | Falta mandar referencias |
| KI-L5FR | Lucía Ferrer | Paula | Jornada | ~5 días | Pendiente de anticipo (quedan 18 h) | Falta anticipo y revisión |
| KI-T3RV | Tomás Rivero | Emiliano | Mediano | ~6 días | Confirmada ($1.100) | Falta revisar referencias |
| KI-B9AG | Bruno Aguirre | Paula | Chico | ~7 días | **Vencida** (reservó hace 3 días sin anticipo; el horario queda libre) | — |
| KI-S2MZ | Sofía Méndez | Regina | Chico | ~8 días | Pendiente de anticipo (quedan 38 h) | Falta anticipo y revisión |
| KI-C6BN | Camila Benítez | Santiago | Grande | ~9 días | Confirmada ($2.100), cotización ajustada a $8.200 → saldo $6.100 | **Lista** |
| KI-M3VB | Mateo Villalba | Paula | Grande | ~10 días | Confirmada con monto distinto del sugerido ($2.500 en vez de $2.100) | **Lista** |
| KI-A5RS | Agustina Ríos | Emiliano | Jornada | ~11 días | Confirmada ($3.300). Cambió el enlace hace 5 h | Falta revisar referencias (el panel avisa del cambio) |
| KI-F2PZ | Federico Paz | Santiago | Mediano | ~11 días | **Cancelada** por la persona con más de 72 h: $1.100 a devolver | — |

**Pasadas (3)**

| Código | Cliente | Artista | Tamaño | Cuándo | Estado |
|---|---|---|---|---|---|
| KI-D4LR | Diego Lara | Emiliano | Mediano | Hace ~3 días | Realizada |
| KI-P7HR | Paloma Herrera | Regina | Grande | Hace ~5 días | Realizada (cotización final $7.500) |
| KI-I4CT | Iván Castro | Santiago | Chico | Hace ~2 días | **Ausente**: anticipo de $500 perdido |

En resumen: **8 confirmadas** (5 listas, 3 con pendientes), **2 pendientes de anticipo**, **1 vencida**, **1 cancelada**, **2 realizadas** y **1 ausente**.

«~N días» es aproximado: si ese día el artista no atiende o no hay hueco, la sesión pasa al siguiente día libre, siempre dentro de las dos semanas. Como las fechas son relativas a la carga, si dejás pasar el tiempo sin reiniciar, Lucía y Sofía terminan vencidas.

### Accesos de prueba para «Mi sesión»

| Celular | Código | Caso |
|---|---|---|
| 55 3187 2290 | KI-R4MS | Faltan menos de 72 h: enlace bloqueado |
| 55 4410 7735 | KI-J8PD | Le faltan referencias: el enlace se puede editar |
| 55 6771 2053 | KI-L5FR | Pendiente de anticipo |

Los tres aparecen como atajos en la misma pantalla de «Mi sesión».

### Cómo volver al estado inicial

- **Desde la app:** en `#admin`, al pie, «Restablecer datos de ejemplo» → «Restablecer». Vuelve a generar las 15 sesiones con fechas relativas a ese momento.
- **A mano:** en las herramientas del navegador (Application → Local Storage), borrá `karma-ink-agenda-v2` y, si querés cerrar también la sesión de «Mi sesión», `karma-ink-mi-sesion`. Recargá la página. O desde la consola:

  ```js
  localStorage.removeItem('karma-ink-agenda-v2');
  localStorage.removeItem('karma-ink-mi-sesion');
  sessionStorage.removeItem('karma-ink-panel');   // vuelve a pedir la clave del panel
  location.reload();
  ```

La clave de los datos pasó de `v1` a `v2` al cargar los 15 datos nuevos. Así, quien ya había abierto la app ve los datos nuevos sin tener que borrar nada. La clave vieja `karma-ink-agenda-v1` queda sin uso y se puede borrar.

---

## 4. Reglas del encargo

- **Cupo:** un artista atiende a una persona por vez. Turnos cada 30 min. Cada sesión ocupa su duración + 30 min de preparación (no visibles al público). Solo se ofrece si la sesión entra entera antes del cierre.
- **Anticipo:** 30 % de la cotización inicial redondeado a la centena (`$1.800 → $500`, `$3.500 → $1.100`, `$7.000 → $2.100`, `$11.000 → $3.300`). El estudio puede anotar otro monto.
- **Saldo:** cotización final − anticipo. Se recalcula al escribir, sin recargar.
- **Lista previa:** «Lista» = anticipo registrado + referencias revisadas. Cambiar el enlace desmarca «Revisadas». A menos de 72 h el enlace queda bloqueado.
- **Estados:** Pendiente de anticipo, Vencida, Confirmada, Realizada, Ausente, Cancelada. Cancelar con ≥ 72 h marca anticipo a devolver; con menos, o ausencia, se pierde.

---

## 5. Decisiones que no estaban en el encargo

- **Sin JSX compilado:** se usa `htm` en lugar de Babel para no necesitar build. Se lee igual que JSX y cada componente puede pasarse a `.jsx` sin cambiar la lógica.
- **Un solo archivo:** la app vive en `index.html` para que se abra sin instalar nada. Al sumar backend conviene separarla en módulos (catálogo, reglas, pantallas).
- **Turnos desde 72 h adelante:** el encargo no fijaba antelación mínima. Se abre desde 72 h para que entren el plazo de anticipo (48 h) y la revisión de referencias antes del bloqueo.
- **El plazo de 48 h corre desde que se aparta el turno,** no desde otra fecha.
- **El vencimiento se revisa también cada 60 s** con la app abierta, además de al abrirla.
- **La preparación puede pasar del cierre:** solo la sesión tiene que entrar antes del cierre. Una sesión que termina justo al cierre deja sus 30 min de preparación después.
- **Realizada y Ausente siguen ocupando su tramo** (son historial). Solo Vencida y Cancelada lo liberan.
- **El enlace de referencias es opcional al apartar.** Se puede sumar después desde «Mi sesión», y también quitar.
- **«Referencias revisadas» se desactiva sin enlace:** no se puede marcar si la persona no mandó nada.
- **Cancelan las dos partes:** la persona desde «Mi sesión» y el estudio desde el panel. Ambas muestran antes si el anticipo se devuelve o se pierde.
- **Confirmaciones dentro de la página,** sin diálogos del navegador.
- **Anticipo corregible:** el estudio puede corregir el monto registrado. No se puede anular el registro.
- **Una sesión vencida no se reactiva:** su horario ya se liberó. La persona tiene que cotizar de nuevo.
- **«Realizada» se puede marcar en cualquier momento,** sin comprobar que la hora ya haya pasado.
- **Saldo negativo:** si el anticipo supera la cotización final, se avisa que queda saldo a favor de la persona.
- **Código de sesión:** formato `KI-XXXX`, sin caracteres ambiguos (sin O, 0, I, 1, L). Al buscar, el prefijo `KI-` es opcional y el celular se compara por sus últimos 10 dígitos.
- **Formato de montos:** pesos mexicanos con separador de miles con punto (`$3.500`), por coherencia con el español rioplatense.
- **Solo tema claro:** la paleta de marca está pensada sobre fondo `#FAFAF7`, así que no se armó modo oscuro.
- **Panel del estudio:** la agenda muestra desde 14 días atrás hasta 30 adelante y abre en el día de la próxima sesión. Suma una pestaña «Por revisar» con las próximas sesiones que no están listas, ordenadas por fecha, y marca las que están a menos de 72 h.
- **Bloques de color por artista:** reemplazan imágenes de tatuajes, como pedía el encargo. Las composiciones (manchas, negro sólido, escala de grises, puntillado) son una interpretación de cada estilo.
- **Clave del panel en sessionStorage:** el encargo no decía cuánto dura el acceso. Queda abierto hasta cerrar la pestaña o tocar «Cerrar panel».
- **WhatsApp sin número fijo:** el encargo no daba un número del estudio. Por defecto la persona elige a quién mandarlo; se puede fijar el del estudio en `STUDIO_WHATSAPP`.
- **Texto de WhatsApp en primera persona:** sirve tanto para mandárselo al estudio como a alguien de confianza. Incluye el código, pero no el celular, así que con el texto solo no se entra a «Mi sesión».
- **El .ics trae solo la sesión:** la hora de fin no incluye los 30 min de preparación, que no se muestran al público. Suma un recordatorio 24 h antes.
- **El .ics no tiene dirección:** el encargo no daba la dirección del estudio, así que el lugar dice solo «Karma Ink Collective».
- **«Listo» se puede recargar:** si se recarga la página, muestra la sesión abierta en «Mi sesión» en lugar de un error.
- **Las 12 próximas incluyen una vencida y una cancelada:** así los datos de ejemplo cubren los seis estados. En la agenda activa quedan 10.
