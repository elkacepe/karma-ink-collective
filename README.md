# Karma Ink Collective · Agenda

Webapp de agenda para un colectivo de cuatro tatuadores. Arma la sesión a partir de una cotización y lleva una **lista previa**: anticipo registrado y referencias revisadas. Así el artista no se entera el mismo día de que tiene que rediseñar la pieza.

Pensada primero para el celular (funciona desde 360 px de ancho).

## Stack

- **React 18.3.1** (UMD) con [htm](https://github.com/developit/htm) para escribir JSX sin paso de compilación
- **Tailwind CSS 3.4.17** (Play CDN)
- **localStorage** como persistencia (clave `karma-ink-agenda-v1`)
- Tipografías: **Syne** 700 para titulares e **Inter** para el texto

Todo vive en `index.html`. No hace falta instalar nada.

## Cómo correrla

```bash
npx serve .
# o
python3 -m http.server 8080
```

Abrí `http://localhost:8080`.

## Vistas

| Ruta | Para quién | Qué hace |
|---|---|---|
| `#inicio` | Público | Presentación, artistas y tamaños |
| `#cotizar` | Público | Artista → tamaño → día y horario → datos. Muestra cotización y anticipo sugerido |
| `#listo` | Público | Código de la sesión y lo que falta para que quede lista |
| `#mi-sesion` | Público | Vuelve con celular + código. Lista previa, enlace de referencias, cancelación |
| `#admin` | Estudio | Agenda por día y artista, pestaña «Por revisar», detalle de cada sesión |

## Reglas de negocio

**Cupo.** Cada artista atiende a una persona por vez. Los turnos arrancan cada 30 min dentro del horario y ocupan la duración completa más 30 min de preparación (no visibles al público). Un horario se ofrece solo si la sesión entra entera antes del cierre. Se abren turnos desde 72 h adelante.

**Anticipo.** Se sugiere el 30 % de la cotización inicial, redondeado a la centena (`$3.500 → $1.100`). La app no cobra: el estudio lo recibe por fuera y lo registra a mano, con el monto que corresponda. Sin anticipo a las 48 h, la sesión pasa a **Vencida** al abrir la app y libera el horario.

**Saldo.** Cotización final − anticipo. El artista puede ajustar la cotización final; el anticipo registrado no cambia.

**Lista previa.** Una sesión está **Lista** cuando tiene anticipo registrado y referencias revisadas. Si la persona cambia el enlace de referencias, se desmarca «Revisadas». A menos de 72 h el enlace queda bloqueado.

**Estados.** Pendiente de anticipo, Vencida, Confirmada, Realizada, Ausente y Cancelada. Cancelar con 72 h o más indica anticipo a devolver; con menos, o si la persona no se presenta, el anticipo se pierde.

## Catálogo de ejemplo

| Artista | Estilo | Horario |
|---|---|---|
| Regina | Abstracto y acuarela | Mar a vie, 11–19 |
| Paula | Blackwork | Mié a dom, 10–18 |
| Emiliano | Realismo black and grey y tradición mexicana | Mar a sáb, 12–20 |
| Santiago | Microrrealismo y color | Jue a dom, 9–17 |

| Tamaño | Duración | Cotización (MXN) |
|---|---|---|
| Chico, hasta 7 cm | 1 h | $1.800 |
| Mediano, hasta 15 cm | 2 h | $3.500 |
| Grande, hasta 25 cm | 4 h | $7.000 |
| Sesión de jornada | 7 h | $11.000 |

Todos los nombres, montos y registros son inventados. Los datos de ejemplo se generan relativos a la fecha actual y se pueden restablecer desde el panel.

## Accesos de ejemplo para «Mi sesión»

| Celular | Código | Caso |
|---|---|---|
| 55 3187 2290 | KI-R4MS | Faltan menos de 72 h: enlace bloqueado |
| 55 4410 7735 | KI-J8PD | Le faltan referencias |
| 55 6771 2053 | KI-L5FR | Pendiente de anticipo |

## Modelo de datos (para el backend)

Cada sesión:

```js
{
  id, code,            // código público, p. ej. "KI-R4MS"
  name, phone,         // phone: solo dígitos
  artist, size,        // ids del catálogo
  start,               // timestamp ms
  quoteInitial,        // cotización por tamaño (base del anticipo sugerido)
  quoteFinal,          // ajustable por el artista
  depositPaid,         // número o null
  depositAt,
  refLink,             // enlace de texto, sin archivos
  refsReviewed,        // boolean
  refNote,             // nota del artista
  refChangedAt,
  status,              // 'pendiente' | 'confirmada' | 'vencida' | 'realizada' | 'ausente' | 'cancelada'
  createdAt, expiredAt, cancelledAt, cancelledBy, refundDue, depositLost
}
```

Puntos a reemplazar al conectar un backend:

- `loadDB` / `saveDB`: hoy leen y escriben localStorage.
- `onBook`: crea la sesión. El servidor debe repetir la comprobación de choque (`conflicts`) para evitar dos reservas en el mismo tramo.
- `sweep`: vencimiento a las 48 h. Conviene correrlo también del lado del servidor.
