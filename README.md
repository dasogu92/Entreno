# Fitness de semana

App personal de recomposición física: entrenamiento guiado, menú diario y compensación de excesos.

Funciona como PWA instalable en el móvil y sin conexión.

## Qué hace

- **Entrenar** — entrenamiento del día con sesión guiada por tiempo: cuenta atrás por serie, descansos automáticos, pitido y vibración en los cambios, pausa y saltar. Cada día tiene 3 variantes (A/B/C) que rotan por semana, con calentamiento, 7 ejercicios principales y bloque final: **40-45 min** en intensidad Normal. Selector **Hoy / Mañana** para prepararte con antelación.
- **Comer** — menú del día con la lista de ingredientes de cada comida y objetivos de kcal y proteína. En la vista **Mañana** genera además la lista de la compra sin repetidos.
- **Composición corporal** — IMC con rango saludable, metabolismo basal, gasto diario, grasa estimada y masa magra a partir de tu peso, altura, edad y sexo.
- **Intensidad ajustable** — Suave, Normal o Intenso: cambia series, repeticiones, duración y descansos de todos los entrenamientos.
- **Compensar** — registra cervezas, copas o comida extra (de hoy o de ayer) y te dice qué hacer para saldarlo. Descuenta automáticamente las calorías del entreno que ya has hecho.
- **Progreso** — peso, racha, curva de evolución e historial.

Los datos se guardan en el navegador del dispositivo (`localStorage`).

## Archivos

| Archivo | Para qué |
|---|---|
| `index.html` | La app completa |
| `manifest.webmanifest` | Metadatos de la PWA |
| `sw.js` | Service worker (funcionamiento sin conexión) |
| `icon-192.png` | Icono |
| `icon-512.png` | Icono grande |
| `icon-180.png` | Icono de iOS |
| `icon-maskable-512.png` | Icono adaptativo de Android |
| `favicon-32.png` | Favicon del navegador |

Todos deben ir en la **misma carpeta** (las rutas son relativas).

## Publicar en GitHub Pages

1. Sube todos los archivos a la raíz del repositorio.
2. **Settings → Pages**.
3. En *Source* deja **Deploy from a branch**.
4. En *Branch* elige **main** y **/ (root)** → **Save**.
5. Al cabo de un minuto la app está en `https://USUARIO.github.io/REPO/`.

## Instalar en el móvil

- **iPhone (Safari):** Compartir → *Añadir a pantalla de inicio*.
- **Android (Chrome):** menú ⋮ → *Instalar app*.

Ábrela una vez con conexión para que se guarde en caché; después funciona sin datos.

## Actualizar la app

Al subir una versión nueva de `index.html`, cambia el número de versión en `sw.js`:

```js
const CACHE = 'fds-v1';   // → 'fds-v2'
```

Así el móvil descarta la caché antigua y coge los cambios. Los datos guardados no se pierden.

## Personalizar

Todo está en `index.html`, en constantes al principio del `<script>`:

- `WORKOUTS` — días, variantes y ejercicios (series, repeticiones, duración, descansos).
- `MEALS` — comidas e ingredientes de cada día.
- `DRINKS` y `FOODS` — bebidas y excesos con sus calorías.
- `BURN` — equivalencias de ejercicio para compensar (kcal por repetición o minuto).
- `kcalPerMin()` — calorías estimadas por minuto de entreno según intensidad.
- `INTENS` — cuánto escala cada nivel de intensidad (series, volumen, descansos).

## Copia de seguridad

En el perfil tienes **Exportar copia** (descarga un `.json` con todo) e **Importar copia**. Úsalo antes de cambiar de móvil o si vas a borrar datos del navegador.

## Aviso

Las calorías y estimaciones son orientativas y sirven como guía de hábitos, no como cálculo médico. Para la recomposición manda el balance de la semana, no el de un día suelto.
