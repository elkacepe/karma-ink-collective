# Karma Ink Collective · Agenda

Webapp de agenda para un colectivo de cuatro tatuadores. Arma la sesión a partir de una cotización y lleva una **lista previa**: anticipo registrado y referencias revisadas. Así el artista no se entera el mismo día de que tiene que rediseñar la pieza.

Pensada primero para el celular: funciona desde 360 px de ancho.

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
| 11–31 | `<style>` | Tokens de color y tipografía, gutter seguro para móvil |
| 34–45 | Scripts | Tailwind + `tailwind.config` (colores `paper`, `ink`, `violet`, `lilac`, `stone`, `jade`), React, ReactDOM, htm |
| 57–75 | **Catálogo** | `ARTISTS`, `SIZES` y `ART` (bloques de color de cada artista) |
| 77–154 | **Reglas de negocio** | Constantes de plazos, estados, `suggest` (anticipo), `isReady` (lista previa), `conflicts` (cupo), `daySlots` (horarios ofrecidos), `sweep` (vencimiento a 48 h) |
| 156–188 | **Datos de ejemplo** | `seed()` |
| 190–197 | **Persistencia** | `loadDB`, `saveDB`, `loadMe`, `saveMe` |
| 199–260 | Piezas de UI | `PrepTag` (etiqueta Lista / Le falta algo), `StatusPill`, `Header`, `Footer`, `Toast` |
| 262–321 | Pantalla | `Home`: inicio |
| 323–478 | Pantalla | `Cotizar`: artista → tamaño → día y horario → datos |
| 480–524 | Pantalla | `Listo`: código y pendientes tras apartar |
| 526–668 | Pantalla | `Login` y `ClientSession`: Mi sesión |
| 670–916 | Pantalla | `Admin`, `SessionCard` y `DetailSheet`: panel del estudio |
| 917–972 | App | Router por hash, estado global, `update`, `onBook`, `resetDemo` |

Los números de línea son de este commit. Si cambian, buscá por nombre de función.

### Rutas

| Ruta | Para quién | Qué hace |
|---|---|---|
| `#inicio` | Público | Presentación, artistas, cómo funciona y tamaños |
| `#cotizar` | Público | Arma la cotización y aparta el turno |
| `#listo` | Público | Muestra el código y lo que falta |
| `#mi-sesion` | Público | Volver con celular + código: lista previa, referencias, cancelar |
| `#admin` | Estudio | Agenda por día y artista, «Por revisar», detalle de sesión. Equivale a `/admin` |

---

## 2. Qué está simulado y cómo se comporta

| Qué | Cómo se comporta hoy | Punto de conexión |
|---|---|---|
| **Base de datos** | Todo se guarda en localStorage, clave `karma-ink-agenda-v1`. Solo comparten datos quienes usan el mismo navegador. Dos pestañas abiertas se sincronizan por el evento `storage`. | `loadDB` / `saveDB` (l. 191–195) y el listener `storage` en `App` (l. 935) |
| **Crear reserva** | Comprueba choques contra los datos locales y guarda. | `onBook` en `App` (l. 943). El servidor debe repetir `conflicts` de forma atómica |
| **Cobro del anticipo** | **No hay pago ni tarjeta en ningún lado.** La app solo sugiere el monto. El estudio lo registra a mano en `#admin` con «Registrar», con el monto que quiera. | `registerDeposit` en `DetailSheet` (l. 702) |
| **Avisos a la persona** | **No se envía ningún correo, WhatsApp ni SMS.** El código se muestra en pantalla con botón «Copiar». La pantalla `#listo` le dice a la persona que pague por fuera de la app y coordine con el estudio cómo hacerlo; la app no le escribe. | Textos en `Listo` (l. 515). Si se automatiza un aviso, se dispara desde `onBook` |
| **Vencimiento a 48 h** | Lo calcula `sweep` en el navegador al abrir la app y luego cada 60 s. No hay tarea programada. | `sweep` (l. 145) e intervalo en `App` (l. 934). En producción, mover a un job del servidor |
| **Identidad de la persona** | Sin cuentas. Se valida celular + código contra los datos locales. La sesión abierta se recuerda en el dispositivo (clave `karma-ink-mi-sesion`) hasta tocar «Salir». | `Login` (l. 527) y `loadMe` / `saveMe` |
| **Acceso al panel** | **`#admin` no tiene contraseña.** Cualquiera con la URL entra. | Ruta `admin` en `App` |
| **Referencias** | Solo un enlace de texto. No se suben archivos ni se valida que el enlace exista. | `saveLink` en `ClientSession` (l. 573) |
| **Reloj** | Usa la hora real del dispositivo y su zona horaria. Los horarios de los artistas se interpretan en esa zona. | `Date.now()` en `daySlots`, `sweep` y pantallas |

---

## 3. Datos de ejemplo y cómo reiniciarlos

### Qué viene precargado

**9 sesiones de 9 clientes distintos**, repartidas entre los 4 artistas. Todos los nombres, celulares, montos y enlaces son inventados. Las fechas se generan **relativas al momento en que se cargan los datos**, así que la agenda siempre se ve vigente.

| Cliente | Artista | Tamaño | Cuándo | Estado al cargar | Lista previa |
|---|---|---|---|---|---|
| Martina Salas | El primero que atienda en 1–2 días | Mediano | En menos de 72 h | Confirmada ($1.100) | **Lista** |
| Valentina Ochoa | Emiliano | Grande | ~4 días | Confirmada ($2.100) | **Lista** |
| Camila Benítez | Santiago | Grande | ~11 días | Confirmada ($2.100), cotización ajustada a $8.200 → saldo $6.100 | **Lista** |
| Tomás Rivero | Emiliano | Mediano | ~9 días | Confirmada ($1.100) | Falta revisar referencias |
| Joaquín Pardo | Santiago | Chico | ~4 días | Confirmada ($500) | Falta mandar referencias |
| Lucía Ferrer | Paula | Jornada | ~5 días | Pendiente de anticipo (quedan 18 h) | Falta anticipo y revisión |
| Sofía Méndez | Regina | Chico | ~10 días | Pendiente de anticipo (quedan 38 h) | Falta anticipo y revisión |
| Bruno Aguirre | Paula | Chico | ~8 días | **Vencida** (se reservó hace 3 días sin anticipo; el horario queda libre) | — |
| Diego Lara | Emiliano | Mediano | Hace ~3 días | Realizada | — |

En resumen: **5 confirmadas** (3 listas, 2 con pendientes), **2 pendientes de anticipo**, **1 vencida**, **1 realizada** y **0 canceladas o ausentes**. Para ver cancelaciones y ausencias, generalas desde `#admin` o desde «Mi sesión».

Como las fechas son relativas a la carga, si dejás pasar el tiempo sin reiniciar, Lucía y Sofía terminan vencidas.

### Accesos de prueba para «Mi sesión»

| Celular | Código | Caso |
|---|---|---|
| 55 3187 2290 | KI-R4MS | Faltan menos de 72 h: enlace bloqueado |
| 55 4410 7735 | KI-J8PD | Le faltan referencias: el enlace se puede editar |
| 55 6771 2053 | KI-L5FR | Pendiente de anticipo |

Los tres aparecen como atajos en la misma pantalla de «Mi sesión».

### Cómo volver al estado inicial

- **Desde la app:** en `#admin`, al pie, «Restablecer datos de ejemplo» → «Restablecer». Vuelve a generar las 9 sesiones con fechas relativas a ese momento.
- **A mano:** en las herramientas del navegador (Application → Local Storage), borrá `karma-ink-agenda-v1` y, si querés cerrar también la sesión de «Mi sesión», `karma-ink-mi-sesion`. Recargá la página. O desde la consola:

  ```js
  localStorage.removeItem('karma-ink-agenda-v1');
  localStorage.removeItem('karma-ink-mi-sesion');
  location.reload();
  ```

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
- **Panel del estudio:** la agenda muestra desde 3 días atrás hasta 30 adelante y abre en el día de la próxima sesión. Suma una pestaña «Por revisar» con las próximas sesiones que no están listas, ordenadas por fecha, y marca las que están a menos de 72 h.
- **Bloques de color por artista:** reemplazan imágenes de tatuajes, como pedía el encargo. Las composiciones (manchas, negro sólido, escala de grises, puntillado) son una interpretación de cada estilo.
