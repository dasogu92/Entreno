# Activar cuentas y sincronización

La app funciona sin esto. Si no configuras nada, los datos se guardan solo en el móvil, como hasta ahora, y la sección de cuenta ni siquiera aparece.

## 1. Crea el proyecto

Entra en [supabase.com](https://supabase.com), crea una cuenta y un proyecto nuevo. Elige la región **West EU (Ireland)** o **Central EU (Frankfurt)**, que son las más cercanas.

El plan gratuito incluye 50.000 usuarios activos al mes y 500 MB de base de datos. Para el tamaño de esta app, los datos de un usuario ocupan unos pocos KB: te sobra de largo.

## 2. Crea la tabla

En el panel de Supabase, ve a **SQL Editor** y ejecuta esto:

```sql
create table if not exists public.user_data (
  user_id    uuid primary key references auth.users on delete cascade,
  data       jsonb not null default '{}'::jsonb,
  updated_at timestamptz not null default now()
);

alter table public.user_data enable row level security;

create policy "leer lo propio"      on public.user_data
  for select using (auth.uid() = user_id);
create policy "insertar lo propio"  on public.user_data
  for insert with check (auth.uid() = user_id);
create policy "actualizar lo propio" on public.user_data
  for update using (auth.uid() = user_id);
```

Las tres políticas son importantes: garantizan que cada usuario solo puede leer y escribir **sus** datos, aunque la clave de la app sea pública.

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

## 5. Ajusta el correo de confirmación

Por defecto Supabase envía un correo para confirmar la cuenta. Dos opciones:

- **Dejarlo activado** (recomendado): más seguro, pero el usuario debe abrir su correo antes de entrar.
- **Desactivarlo** para que sea inmediato: **Authentication → Providers → Email** y desactiva *Confirm email*.

Ojo con el límite de correos del plan gratuito (unos pocos por hora). Si esto crece, conecta un servicio de envío propio en **Authentication → Emails → SMTP**.

## Cómo funciona la sincronización

- Los datos siguen guardándose **primero en el móvil**, así que la app funciona sin conexión igual que antes.
- Cada cambio se sube a la nube unos segundos después, agrupado para no saturar.
- Al entrar en otro dispositivo, se descarga lo que haya en la nube.
- Si hay conflicto, gana **la versión más reciente** según su marca de tiempo. Si el dispositivo está vacío, siempre se descarga la nube.
- Sin conexión no se pierde nada: se reintenta en el siguiente guardado o al recuperar la conexión.

## Si más adelante cobras

Con las cuentas ya montadas, añadir el pago consiste en guardar en la tabla el estado de la suscripción y comprobarlo al entrar. Antes de cobrar, revisa con tu gestor el alta y la facturación, y ten en cuenta que un *merchant of record* (Lemon Squeezy, Paddle, Stripe Managed Payments) te evita gestionar el IVA de cada país.
