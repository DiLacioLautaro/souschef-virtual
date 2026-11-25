# Cómo está hecho SousChef

El código de SousChef es privado. Esta página muestra **fragmentos reales** de ese código, tal
como están en la rama principal al 25/11/2026, para que se vea cómo está resuelto lo que el
[README](../README.md) afirma. Donde un fragmento está recortado se marca con `// …`. Los
nombres, comentarios y decisiones son los originales.

**En esta página:** [El recorrido de un mensaje](#el-recorrido-de-un-mensaje) ·
[Webhooks públicos](#1-webhooks-públicos-firma-sobre-el-cuerpo-crudo) ·
[El bot no dice precios](#2-el-bot-no-dice-precios-y-no-depende-del-modelo-para-cumplirlo) ·
[Aislamiento entre negocios](#3-aislamiento-entre-negocios-en-la-base) ·
[Precio en dólares](#4-el-precio-del-plan-pro-anclado-al-dólar) ·
[Tests de arquitectura](#5-un-test-que-lee-el-código-fuente) ·
[Cómo se prueba](#cómo-se-prueba)

---

## El recorrido de un mensaje

Una clienta escribe por WhatsApp, Instagram o Messenger. Los tres webhooks son adaptadores
finos sobre un único módulo compartido (`_shared/bot_core.ts`), así que el recorrido es el mismo:

| Paso | Qué pasa | Dónde |
|---|---|---|
| 1 | Se descarta el cuerpo si pesa más de 1 MB y se **verifica la firma** de Meta sobre el cuerpo crudo | `whatsapp-webhook/index.ts` |
| 2 | Se ubica el negocio dueño del número y se **persiste** el mensaje | `runWebhook`, `persistInbound` |
| 3 | Se **espera y se junta la ráfaga**: si la clienta manda cinco mensajitos seguidos, se responde una vez | `juntarRafaga` |
| 4 | **Freno de gasto de IA**, por negocio y global | `aiFlood` |
| 5 | **Triage**: el modelo clasifica el mensaje en un carril (`responde`, `prepara` o `escala`) y extrae los datos del pedido como JSON | `runTriage` |
| 6 | **Guardrail de salida**: si la respuesta trae un monto o datos de pago, se reemplaza | `scrubReply` |
| 7 | Se responde por el canal, se arma o actualiza el pedido y se le avisa a la dueña | `processTurn` |

El triage corre en cascada entre proveedores: Claude se usa solo si el negocio tiene plan Pro, y
si un proveedor falla o no está configurado, sigue el siguiente (Groq, Gemini). Cada respuesta
registra qué proveedor contestó y en qué nivel de la cascada.

---

## 1. Webhooks públicos: firma sobre el cuerpo crudo

Los webhooks de Meta son endpoints públicos: cualquiera puede mandarles un POST. Antes de
procesar nada se verifica el HMAC-SHA256 que Meta calcula con el secreto de la app.

```ts
// supabase/functions/_shared/bot_core.ts
export async function verifyMetaSignature(
  rawBody: string, header: string | null, appSecret: string,
): Promise<boolean> {
  if (!appSecret || !header || !header.startsWith('sha256=')) return false;
  const expected = header.slice('sha256='.length);
  const key = await crypto.subtle.importKey(
    'raw', new TextEncoder().encode(appSecret),
    { name: 'HMAC', hash: 'SHA-256' }, false, ['sign'],
  );
  const mac = await crypto.subtle.sign('HMAC', key, new TextEncoder().encode(rawBody));
  const hex = Array.from(new Uint8Array(mac)).map((b) => b.toString(16).padStart(2, '0')).join('');
  if (hex.length !== expected.length) return false;
  let diff = 0;
  for (let i = 0; i < hex.length; i++) diff |= hex.charCodeAt(i) ^ expected.charCodeAt(i);
  return diff === 0;
}
```

```ts
// supabase/functions/whatsapp-webhook/index.ts
  // TOPE DE TAMAÑO: este endpoint es PUBLICO (verify_jwt=false), asi que
  // cualquiera puede postearle. Sin tope, req.text() materializa lo que venga
  // en memoria del isolate. Meta manda notificaciones de unos pocos KB.
  const declarado = Number(req.headers.get('content-length') ?? '0');
  if (declarado > 1_048_576) {
    console.warn('[webhook] body demasiado grande, descartado:', declarado);
    return new Response('too large', { status: 413 });
  }
  const raw = await req.text();
  if (!(await verifyMetaSignature(raw, req.headers.get('x-hub-signature-256'), Deno.env.get('META_APP_SECRET') ?? ''))) {
    return new Response('invalid signature', { status: 401 });
  }
```

Dos detalles: la firma se calcula sobre el texto **antes** de parsear el JSON (reserializar
cambia los bytes y la firma deja de coincidir), y la comparación recorre siempre todo el largo
para no filtrar, por el tiempo de respuesta, cuántos caracteres acertó quien prueba.

---

## 2. El bot no dice precios, y no depende del modelo para cumplirlo

La regla del producto es que el bot nunca le pasa un precio a una clienta. Pedírselo al modelo
en el prompt no alcanza: un modelo puede desobedecer. Por eso hay una capa determinística
después del modelo, que mira el texto que está por salir.

```ts
// supabase/functions/_shared/bot_core.ts
// ── Guardrail de salida: el bot NUNCA INVENTA un precio ───────────────────────
const PRICE_RE = /\$\s*\d|\b\d{1,3}[.,]\d{3}\b|\b(cuesta|sale|precio|vale|pesos?)\b[^.\n]{0,12}\d|\d[^.\n]{0,4}\bpesos?\b|\b\d{4,}\b/i;

// ── Guardrail de pago POR TENANT ──────────────────────────────────────────────
// (…) con varias reposteras el alias/CVU de CADA negocio hay que sacarlo de SU
// config: del bloque de cobro que la dueña pegó (biz.paymentInfo) se extraen los
// tokens "tipo alias" — secuencias con puntos (tortas.de.pau) o palabras largas
// (>8 letras: alias de una palabra, apellido del titular) — escapados para regex.
// Si la reply contiene uno, es fuga de datos de pago de ESTE negocio → scrub.
function tenantPaymentLeakRe(paymentInfo: string): RegExp | null {
  const toks = (paymentInfo.match(/[\p{L}\d][\p{L}\d._-]{3,}/gu) ?? [])
    .filter((t) => t.includes('.') || (t.match(/\p{L}/gu) ?? []).length > 8)
    .filter((t) => !LEAK_COMUNES.has(normTxt(t)))
    .map((t) => t.replace(RE_ESCAPE, '\\$&'));
  if (toks.length === 0) return null;
  return new RegExp(toks.join('|'), 'i');
}

function scrubReply(text: string, ownerName: string, ownData = '', paymentInfo = ''): string {
  // 1) datos o confirmación de pago → SIEMPRE derivar a la dueña (hard-gate).
  const tenantRe = tenantPaymentLeakRe(paymentInfo);
  if (PAYMENT_LEAK_RE.test(text) || (tenantRe !== null && tenantRe.test(text))) {
    return `Los datos para transferir y la confirmación del pago los maneja ${ownerName} directamente 💕 Apenas los tenga te confirma.`;
  }
  // 2) precio inventado (guardrail existente).
  if (!PRICE_RE.test(text)) return text;
  // …
  return `El presupuesto exacto te lo confirma ${ownerName} 💕 Contame los detalles ` +
    '(fecha, porciones y temática) y se lo paso para que te pase el precio.';
}
```

Lo mismo vale para los datos de cobro: cuando una clienta pide el alias, el bloque que cargó la
dueña se manda tal cual desde la configuración, sin pasar por el modelo. El modelo nunca
escribe un alias ni confirma que un pago llegó; si lo intenta, esta capa lo reemplaza.

---

## 3. Aislamiento entre negocios, en la base

Cada repostera es un negocio y sus datos no se cruzan con los de otra. Eso se resuelve con
Row-Level Security de Postgres. La política básica (`user_id = auth.uid()`) valida que **la
fila** sea tuya, pero no a quién apuntan sus claves foráneas. Por eso las políticas de escritura
exigen además que cada FK apunte a algo de la misma cuenta:

```sql
-- supabase/migrations/20260893_fk_ajena_no_presta_nada.sql
ALTER POLICY owner_only ON public.order_items
  WITH CHECK (
    user_id = auth.uid()
    AND EXISTS (SELECT 1 FROM public.orders o
                 WHERE o.id = order_items.order_id AND o.user_id = auth.uid())
    AND (order_items.product_id IS NULL
         OR EXISTS (SELECT 1 FROM public.products p
                     WHERE p.id = order_items.product_id AND p.user_id = auth.uid()))
  );

ALTER POLICY conversations_mutate ON public.conversations
  WITH CHECK (
    user_id = auth.uid()
    AND (conversations.business_id IS NULL
         OR EXISTS (SELECT 1 FROM public.businesses b
                     WHERE b.id = conversations.business_id AND b.owner_user_id = auth.uid()))
    AND (conversations.client_id IS NULL
         OR EXISTS (SELECT 1 FROM public.clients c
                     WHERE c.id = conversations.client_id AND c.user_id = auth.uid()))
  );
```

Dos reglas de trabajo acompañan a las políticas:

- **RLS no corre dentro de una función `SECURITY DEFINER`.** Toda función de ese tipo lleva su
  propio predicado de dueño en cada join.
- Un archivo de consultas de solo lectura (`supabase/queries_aislamiento.sql`) recorre el
  catálogo de Postgres y lista las FK sin chequeo de dueño, los índices únicos que no incluyen
  al dueño y las tablas sin RLS. Se corre antes de cada migración que toque tablas o políticas.

Además, las tablas nuevas del esquema público nacen con RLS activada por un disparador de
esquema (`rls_auto_enable`), así que una tabla sin política queda
cerrada en vez de abierta.

---

## 4. El precio del plan Pro, anclado al dólar

El plan se cobra en pesos pero su valor está fijado en dólares. La cotización sale de una API
externa, y una API externa puede devolver cualquier cosa.

```ts
// supabase/functions/_shared/fx.ts
// - BANDA DE SANIDAD: solo se acepta una tasa entre 500 y 20.000 (una API de FX
//   glitcheada podría devolver 0, 19,5 por corrimiento de decimales, o un número
//   gigante → cobrarías $100/mes de por vida, $0, o una fortuna). Fuera de banda
//   → se ignora y se usa la caché / el default.
// - TTL: si la caché tiene < 20 min, se usa sin pegarle a dolarapi (evita
//   martillar la API en cada apertura de la pantalla de planes).
const RATE_MIN = 500;
const RATE_MAX = 20000;
const TTL_MS = 20 * 60_000; // 20 min

// …

/// Redondea al centenar de ARRIBA, no al mas cercano.
///
/// POR QUE ARRIBA. La landing es HTML estatico: no puede consultar el precio del
/// dia, asi que anuncia "desde $15.100" y ese numero tiene que ser el piso real.
/// Con Math.round no lo era — el ancla de USD 7,64 por un dolar de 1.969,50 da
/// 15.047, que redondeaba a 15.000 y dejaba a la app cobrando $100 MENOS que el
/// piso anunciado, o sea con la landing y la pantalla del plan diciendo cosas
/// distintas.
function round100(n: number): number {
  return Math.ceil(n / 100) * 100;
}

// …

  // 2) Traer la cotización EN VIVO, con banda de sanidad.
  let rate = 0;
  let source = 'cache';
  try {
    const r = await fetch('https://dolarapi.com/v1/dolares/tarjeta', { signal: AbortSignal.timeout(8000) });
    if (r.ok) {
      const d = await r.json();
      const v = Number(d?.venta);
      if (v >= RATE_MIN && v <= RATE_MAX) { rate = v; source = 'tarjeta'; }
    }
  } catch (_) { /* sin internet / basura → caemos a la caché */ }
```

El precio anual sale del mismo ancla y de la misma cotización que el mensual, para que el
ahorro que muestra la pantalla sea siempre el real.

---

## 5. Un test que lee el código fuente

Toda función que le manda a un modelo de IA datos que la dueña carga desde la app tiene que
comprobar antes que ella dio su permiso, y en un orden preciso: autenticar, permiso, freno de
gasto, IA. El compilador no puede verificar eso, así que lo verifica un test que lee los
archivos:

```ts
// supabase/functions/_shared/puerta_ia_test.ts
Deno.test('toda función con IA está decidida, y la que lleva la puerta la tiene en su lugar', async () => {
  const raiz = new URL('../', import.meta.url);
  const conIA: string[] = [];
  for await (const d of Deno.readDir(raiz)) {
    if (!d.isDirectory || d.name.startsWith('_')) continue;
    // …
    if (!USA_IA.test(fuente)) continue;
    conIA.push(d.name);
    if (SIN_PUERTA.includes(d.name)) continue;

    const autentica = fuente.indexOf('getUser(');
    const puerta = fuente.indexOf('tieneConsentimientoIA(');
    const freno = fuente.indexOf('hayFlood(');
    assert(autentica >= 0, `${d.name}: no autentica a la dueña (getUser)`);
    assert(puerta > autentica, `${d.name}: falta la puerta o va antes de autenticar`);
    assert(freno < 0 || puerta < freno, `${d.name}: la puerta va después del freno`);
  }
  assertEquals(conIA.sort(), [...CON_PUERTA, ...SIN_PUERTA].sort());
});
```

El último `assert` es el que importa: una función nueva que use IA hace fallar el test hasta
que alguien decide, a propósito, en cuál de las dos listas va.

---

## Cómo se prueba

| Qué | Con qué | Estado al 25/11/2026 |
|---|---|---|
| Lógica compartida del backend | 14 archivos de test en Deno | **127 tests, todos pasan** (corrida local) |
| El bot en el tiempo | Simulador de charlas | **17 tests, todos pasan**; incluye 120 charlas al azar |
| Comportamiento del modelo | Evals con promptfoo | 168 casos escritos en seis archivos; necesitan las claves de los modelos, no se corrieron para esta página |
| La app | `flutter test` | 74 archivos de test; corren en el CI (fila de abajo) |
| CI | GitHub Actions | Analiza y testea la app en Linux y, si pasa, compila iOS sin firma en macOS. **Última corrida: n.º 10, el 14/11/2026, en verde**; las cinco más recientes pasaron |

**El simulador de charlas** corre el bot real (`runWebhook` y `processTurn` enteros) con una
base y un reloj de mentira, y un modelo de juguete que devuelve a propósito las respuestas
malas que se vieron en uso real. Sirve para probar lo que depende del tiempo: esperar a que la
clienta termine de escribir, juntar los mensajes, no hablarle encima a la dueña. La prueba más
exigente genera charlas al azar; su salida en la corrida del 25/11/2026:

```
120 charlas · 585 mensajes de clientas · 251 respuestas del bot · 0 con problemas
```

**Las evals** usan el mismo prompt que producción (un test falla si la copia que usan las evals
deja de coincidir con el del código). Cada caso es un mensaje de clienta y una aserción sobre el
JSON que devuelve el modelo:

```yaml
# tests/promptfoo/casos_cobertura.yaml
- description: 'cob: 12 cupcakes decorados — tipo cupcakes, no torta'
  vars: {message: 'hola! quiero encargar 12 cupcakes decorados para el 20 de agosto'}
  assert:
    - type: javascript
      value: |
        const j = JSON.parse(output.slice(output.indexOf('{'), output.lastIndexOf('}') + 1));
        const t = String(j.entidades?.tipo_producto ?? '').toLowerCase();
        return j.carril === 'prepara' && /cupcake/.test(t) && !/^torta$/.test(t);
```

---

[Volver al README](../README.md) · [Manual de usuario](manual/README.md)
