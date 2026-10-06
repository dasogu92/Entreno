# Activar cuentas y sincronización

La app funciona sin esto. Si no configuras nada, los datos se guardan solo en el móvil, como hasta ahora, y la sección de cuenta ni siquiera aparece.

## 1. Crea el proyecto

Entra en [supabase.com](https://supabase.com), crea una cuenta y un proyecto nuevo. Elige la región **West EU (Ireland)** o **Central EU (Frankfurt)**, que son las más cercanas.

El plan gratuito incluye 50.000 usuarios activos al mes y 500 MB de base de datos. Para el tamaño de esta app, los datos de un usuario ocupan unos pocos KB: te sobra de largo.

## 2. Crea las tablas

En el panel de Supabase, ve a **SQL Editor** → **New query**, pega esto entero y pulsa **Run**.
Lo deja todo listo de una vez, incluido lo que hará falta el día que cobres.

```sql
-- Datos de cada usuario (su historial completo en un JSON)
create table if not exists public.user_data (
  user_id       uuid primary key references auth.users on delete cascade,
  data          jsonb not null default '{}'::jsonb,
  premium_until timestamptz,
  updated_at    timestamptz not null default now()
);

-- Relación entre el cliente de Stripe y el usuario (para cuando actives el cobro)
create table if not exists public.billing (
  user_id         uuid primary key references auth.users on delete cascade,
  stripe_customer text unique not null
);

-- Seguridad: cada uno solo ve y toca lo suyo
alter table public.user_data enable row level security;
alter table public.billing   enable row level security;

drop policy if exists "leer lo propio"       on public.user_data;
drop policy if exists "insertar lo propio"   on public.user_data;
drop policy if exists "actualizar lo propio" on public.user_data;

create policy "leer lo propio"       on public.user_data for select using (auth.uid() = user_id);
create policy "insertar lo propio"   on public.user_data for insert with check (auth.uid() = user_id);
create policy "actualizar lo propio" on public.user_data for update using (auth.uid() = user_id);

-- El usuario puede leer si tiene Pro, pero NO puede escribírselo él mismo.
-- Sin esto, cualquiera se regalaría la suscripción desde el navegador.
revoke update on public.user_data from authenticated;
grant  update (data, updated_at) on public.user_data to authenticated;

-- El WITH CHECK impide además que nadie reasigne una fila a otra cuenta
drop policy if exists "actualizar lo propio" on public.user_data;
create policy "actualizar lo propio" on public.user_data
  for update using (auth.uid() = user_id) with check (auth.uid() = user_id);

-- La tabla billing no lleva políticas a propósito: solo la escribe el servidor.
```

Si sale **Success. No rows returned**, ha ido bien.

## 3. Copia tus credenciales

En **Project Settings → API** encontrarás:

- **Project URL** — algo como `https://abcdefgh.supabase.co`
- **anon public** — una clave larga

Esa clave `anon` está pensada para ir en el código del navegador, no es un secreto. Lo que protege los datos son las políticas del paso 2. **Nunca uses la clave `service_role`** en la app.

## 4. Pégalas en la app

Abre `index.html`, busca cerca del principio del `<script>`:

```js
const SUPA_URL='';
const SUPA_KEY='';
```

Y rellénalas:

```js
const SUPA_URL='https://abcdefgh.supabase.co';
const SUPA_KEY='eyJhbGci...';
```

Sube el archivo y listo: aparecerá la sección **Tu cuenta** dentro del perfil.

## 5. Configura la dirección de retorno (IMPORTANTE)

Sin este paso, el enlace del correo de confirmación lleva a `http://localhost:3000` y el usuario ve **«Safari no puede conectar con el servidor»**.

En Supabase → **Authentication** → **URL Configuration**:

- **Site URL**: `https://dasogu92.github.io/Entreno/`
- **Redirect URLs**: añade también `https://dasogu92.github.io/Entreno/**`

Guarda. A partir de ahí, quien confirme su correo aterriza en la app.

## 6. Decide si quieres confirmación por correo

En **Authentication → Providers → Email**, la opción *Confirm email*:

- **Activada** (por defecto): más segura, pero el usuario tiene que salir de la app, abrir su correo y volver. Se pierde gente por el camino.
- **Desactivada**: entra al instante. Para una app entre amigos es perfectamente razonable, y es lo que yo haría al principio.

Ten en cuenta que el plan gratuito limita bastante los correos por hora. Si más adelante crece, conecta tu propio servicio de envío en **Authentication → Emails → SMTP**.

## Invitaciones

Cada usuario tiene un código de 6 letras y un enlace del tipo `.../Entreno/?ref=ABC123`. Quien entre por ahí se lleva el doble de días de prueba, automáticamente.

Para ver quién ha invitado a quién, entra en **Table Editor → user_data** y mira el campo `_ref` dentro de la columna `data`. Ahí aparece el código de quien le invitó.

La recompensa del que invita (un mes gratis por amigo suscrito) **es manual de momento**: cuando veas en Stripe que alguien se suscribe y en Supabase que venía invitado, alarga a mano la fecha de `premium_until` del que le invitó. Automatizarlo requiere ampliar la función del webhook, y no merece la pena hasta que haya volumen.

## Cómo funciona la sincronización

- Los datos siguen guardándose **primero en el móvil**, así que la app funciona sin conexión igual que antes.
- Cada cambio se sube a la nube unos segundos después, agrupado para no saturar.
- Al entrar en otro dispositivo, se descarga lo que haya en la nube.
- Si hay conflicto, gana **la versión más reciente** según su marca de tiempo. Si el dispositivo está vacío, siempre se descarga la nube.
- Sin conexión no se pierde nada: se reintenta en el siguiente guardado o al recuperar la conexión.

## Si más adelante cobras

Con las cuentas ya montadas, añadir el pago consiste en guardar en la tabla el estado de la suscripción y comprobarlo al entrar. Antes de cobrar, revisa con tu gestor el alta y la facturación, y ten en cuenta que un *merchant of record* (Lemon Squeezy, Paddle, Stripe Managed Payments) te evita gestionar el IVA de cada país.
