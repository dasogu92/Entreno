# Modo social: competir con amigos

Permite ver quién ha entrenado hoy y quién lleva mejor racha. Hasta que ejecutes este SQL, la sección simplemente no aparece.

## Qué se comparte y qué no

Tus amigos ven **solo** tu nombre, tu racha, si has entrenado hoy y cuántos entrenos llevas esta semana.

No ven tu peso, ni tus medidas, ni tu composición corporal, ni lo que comes, ni tus excesos. Esos datos siguen en `user_data`, que nadie más puede leer.

Además, solo pueden verte quienes tú hayas añadido como amigos. No hay directorio público ni forma de buscar gente al azar.

## El SQL

En Supabase → **SQL Editor** → **New query**, pega esto y pulsa **Run**:

```sql
-- Datos públicos entre amigos (nada sensible)
create table if not exists public.profiles (
  user_id       uuid primary key references auth.users on delete cascade,
  nick          text,
  code          text unique,
  streak        int  default 0,
  today_done    boolean default false,
  today_date    date,
  week_workouts int  default 0,
  updated_at    timestamptz default now()
);

-- Quién sigue a quién
create table if not exists public.friends (
  user_id   uuid references auth.users on delete cascade,
  friend_id uuid references auth.users on delete cascade,
  primary key (user_id, friend_id)
);

alter table public.profiles enable row level security;
alter table public.friends  enable row level security;

drop policy if exists "escribir mi perfil"      on public.profiles;
drop policy if exists "actualizar mi perfil"    on public.profiles;
drop policy if exists "ver perfiles de amigos"  on public.profiles;
drop policy if exists "gestionar mis amistades" on public.friends;

-- Cada uno escribe solo su perfil
create policy "escribir mi perfil"   on public.profiles
  for insert with check (auth.uid() = user_id);
create policy "actualizar mi perfil" on public.profiles
  for update using (auth.uid() = user_id);

-- Y solo puede leer el suyo y el de sus amigos
create policy "ver perfiles de amigos" on public.profiles
  for select using (
    auth.uid() = user_id
    or exists (select 1 from public.friends f
               where f.user_id = auth.uid() and f.friend_id = profiles.user_id)
  );

-- Cada uno gestiona su propia lista de amigos
create policy "gestionar mis amistades" on public.friends
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);

-- Buscar a alguien por su código sin poder listar a todo el mundo.
-- Devuelve únicamente el identificador y el nombre, nunca el resto de datos.
create or replace function public.buscar_amigo(p_code text)
returns table(user_id uuid, nick text)
language sql
security definer
set search_path = public
as $$
  select p.user_id, p.nick
  from public.profiles p
  where upper(p.code) = upper(p_code)
  limit 1;
$$;

revoke all on function public.buscar_amigo(text) from public, anon;
grant execute on function public.buscar_amigo(text) to authenticated;
```

Si sale **Success. No rows returned**, listo.

## Por qué esa función de búsqueda

Para añadir a alguien necesitas encontrarlo por su código, pero todavía no es tu amigo, así que las políticas te impedirían leer su fila.

La función resuelve el problema sin abrir la mano: se ejecuta con permisos elevados pero **solo devuelve el identificador y el nombre** de quien tenga exactamente ese código. No permite listar usuarios ni ver nada más, y únicamente pueden llamarla quienes hayan iniciado sesión.

## Cómo se usa en la app

El código de amigo es el mismo que el de invitación, el de seis letras que aparece en tu perfil. Uno le pasa el suyo al otro y cada cual lo introduce en **Progreso → Tus amigos**.

La clasificación ordena por racha y, en caso de empate, por entrenos de la semana. El círculo verde con el visto indica que esa persona ya ha completado el entreno de hoy.

Para quitar a alguien, doble toque sobre su fila.

## Nombre visible

Por defecto se usa la parte del correo anterior a la arroba. Cada uno puede cambiarlo en su perfil, en **Nombre para tus amigos**.
