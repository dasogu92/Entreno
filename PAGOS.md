# Activar la suscripción

Requisito previo: tener las cuentas funcionando (ver `CUENTAS.md`). Sin cuentas no hay forma de saber quién ha pagado.

Mientras `CHECKOUT_URL` esté vacío, la app no muestra nada de pago y todo sigue abierto. Puedes subirla sin miedo.

## 1. Prepara la base de datos

En Supabase → **SQL Editor**:

```sql
-- columna donde se guarda hasta cuándo tiene Pro cada usuario
alter table public.user_data
  add column if not exists premium_until timestamptz;

-- El usuario puede LEER su estado pero NO escribirlo.
-- Sin esto, cualquiera se regalaría la suscripción desde el navegador.
revoke update on public.user_data from authenticated;
grant  update (data, updated_at) on public.user_data to authenticated;
```

Esa última parte es la más importante de todo el documento. La clave `anon` es pública, así que si dejas que el usuario escriba `premium_until`, el candado no sirve de nada.

## 2. Dónde va tu cuenta bancaria

Tu IBAN **nunca va en la app**. Se configura en el panel de la plataforma de pago, y es ella quien te ingresa el dinero. En el código solo pones un enlace público de pago.

### Opción recomendada: Stripe

Te paga en euros, directo a tu cuenta española, sin importe mínimo.

1. Crea una cuenta en [stripe.com](https://stripe.com) y activa el modo real.
2. Te pedirán: DNI o NIE, tu NIF, dirección, actividad económica y el **IBAN** donde quieres cobrar.
3. **Productos → Añadir producto** → precio recurrente de **6,99 € al mes**.
4. En ese producto, **Crear enlace de pago**. Ese enlace es el que va en `CHECKOUT_URL`.
5. **Configuración → Pagos**: eliges cada cuánto te transfieren (por defecto automático cada pocos días).

Comisión: 1,5 % + 0,25 € para tarjetas del Espacio Económico Europeo, más 0,5 % por ser suscripción. De 6,99 € te quedan unos **6,60 €**; con el descuento aplicado, de 4,89 € te quedan unos **4,54 €**.

Activa también el **portal de cliente** (Configuración → Billing → Portal) para que cada usuario pueda cancelar solo, sin escribirte.

### Alternativa: Lemon Squeezy

Interesa si vendes fuera de la UE y no quieres saber nada de impuestos, porque actúa como vendedor oficial y gestiona el IVA de cada país.

Pero ojo a dos cosas: tiene un **mínimo de 50 $ para pagarte** (por debajo, se acumula al siguiente ciclo) y paga en dólares convertidos a euros. Su comisión es 5 % + 0,50 € más un 1,5 % por transacción internacional: de 6,99 € te quedan unos 6,04 €.

Se configura en **Settings → Payout**, con transferencia bancaria o PayPal, dos veces al mes.

## 3. Crea el código de descuento

En Stripe → **Productos → Cupones → Crear cupón**:

- Tipo: **Porcentaje**, valor **30**.
- **Duración: Repeating → 12 months.** Esto es lo importante. Si lo dejas en *Once*, el descuento solo valdría el primer mes en lugar del primer año.
- Marca **Usar códigos visibles para el cliente** y escribe `FUNDADORES`.
- Si quieres que sea limitado, pon un tope de canjes o una fecha de caducidad. Quien ya lo haya usado conserva su descuento los 12 meses aunque luego lo cierres.

Después, en el **enlace de pago** que creaste, activa la casilla **Permitir códigos promocionales**. Sin eso el campo del código no aparece en la pantalla de pago.

La app ya lleva el código en el enlace, así que a tus usuarios les saldrá aplicado solo. Además se muestra en el muro por si tienen que escribirlo a mano.

## 3b. Pega el enlace en la app

En `index.html`, cerca del principio del `<script>`:

```js
const PAY = {
  CHECKOUT_URL: 'https://buy.stripe.com/XXXXXXXX',
  PRICE: '6,99 €/mes',
  GATE: true,      // false = no bloquea nada, solo invita a apoyar
  TRIAL_DAYS: 7,
  PROMO: { CODE: 'FUNDADORES', OFF: '30 %', PRICE: '4,89 €/mes', PLAZO: 'el primer año' }
};
```

La app añade sola al enlace el identificador y el correo del usuario, así que el pago queda ligado a su cuenta. Y si alguien pulsa suscribirse sin haberse registrado, primero le lleva a crear la cuenta: así no hay pagos huérfanos.

## 4. El webhook que activa el Pro

Conecta el pago con la app. En Supabase → **Edge Functions**, crea una función `pago`:

```ts
import Stripe from 'https://esm.sh/stripe@16?target=deno';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

const stripe = new Stripe(Deno.env.get('STRIPE_SECRET_KEY')!);
const admin  = createClient(
  Deno.env.get('SUPABASE_URL')!,
  Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!   // clave privada: solo vive en el servidor
);

Deno.serve(async (req) => {
  const body = await req.text();
  let evento;
  try {
    // comprobar la firma: sin esto cualquiera podría regalarse el Pro
    evento = await stripe.webhooks.constructEventAsync(
      body,
      req.headers.get('stripe-signature')!,
      Deno.env.get('STRIPE_WEBHOOK_SECRET')!
    );
  } catch {
    return new Response('firma no válida', { status: 400 });
  }

  // 1) Al completarse la compra guardamos la relación cliente de Stripe ↔ usuario
  if (evento.type === 'checkout.session.completed') {
    const s = evento.data.object as any;
    if (s.client_reference_id && s.customer) {
      await admin.from('billing').upsert({
        user_id: s.client_reference_id,
        stripe_customer: s.customer
      }, { onConflict: 'user_id' });
    }
  }

  // 2) Cualquier cambio de suscripción actualiza la fecha de acceso
  if (evento.type.startsWith('customer.subscription.')) {
    const sub = evento.data.object as any;
    const { data: fila } = await admin.from('billing')
      .select('user_id').eq('stripe_customer', sub.customer).maybeSingle();

    let userId = fila?.user_id;
    if (!userId) {                        // respaldo: emparejar por correo
      const cli: any = await stripe.customers.retrieve(sub.customer);
      const { data: lista } = await admin.auth.admin.listUsers();
      userId = lista.users.find(u => u.email?.toLowerCase() === cli.email?.toLowerCase())?.id;
    }
    if (!userId) return new Response('usuario no encontrado', { status: 404 });

    // activo hasta el fin del periodo pagado; si cancela, conserva lo que pagó
    const hasta = new Date(sub.current_period_end * 1000).toISOString();
    await admin.from('user_data')
      .upsert({ user_id: userId, premium_until: hasta }, { onConflict: 'user_id' });
  }

  return new Response('ok');
});
```

Necesitas también esta tabla auxiliar (SQL Editor):

```sql
create table if not exists public.billing (
  user_id         uuid primary key references auth.users on delete cascade,
  stripe_customer text unique not null
);
alter table public.billing enable row level security;   -- solo el servidor escribe aquí
```

Guarda las tres variables de entorno en Supabase (**Edge Functions → Secrets**): `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET` y la de servicio, que ya viene puesta.

Después, en Stripe → **Desarrolladores → Webhooks**, añade la URL de la función y marca los eventos `customer.subscription.created`, `.updated` y `.deleted`. Stripe te dará el `STRIPE_WEBHOOK_SECRET` al crearlo.

## 5. Compruébalo

1. Haz una compra en modo test.
2. En la app, perfil → **Ya soy Pro — actualizar**.
3. La insignia debe pasar a **PRO** con la fecha de renovación.

## Qué es gratis y qué es Pro

Gratis, para siempre: el entrenamiento guiado del día, el menú con ingredientes y la lista de la compra, y la compensación de excesos.

Pro: la pestaña de Progreso completa, el registro de cargas, las medidas corporales, cambiar de entreno o de comida, y la sincronización entre dispositivos.

La idea es que cualquiera pueda usarla y engancharse, y que pague quien va en serio y quiere no perder su historial. Puedes cambiar el reparto tocando las llamadas a `requirePro()` en el código.

## Antes de cobrar el primer euro

Habla con tu gestor. Cobrar de forma recurrente es un ingreso que hay que declarar, y según el volumen toca darse de alta como autónomo. Llamarlo donativo no cambia nada si lo das a cambio de acceso.

Necesitarás también una página de términos y otra de privacidad enlazadas desde la app, y tener claro el reembolso: en la UE hay 14 días de desistimiento para servicios digitales, salvo renuncia expresa del cliente al contratar.

## Sobre la publicidad

La descarté por números: con unos 30 usuarios activos saldrían unos 3 € al mes, frente a 25 € con cinco suscriptores. Además AdSense exige aprobación, banner de consentimiento de cookies por el RGPD, y ensucia una interfaz que ahora está limpia. Si algún día tienes miles de usuarios la conversación cambia, pero a esta escala no compensa.
