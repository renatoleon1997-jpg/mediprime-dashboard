# Mediprime · Panel de ventas y publicidad

Dashboard estático (un solo `index.html`) que cruza las ventas registradas en el Drive de Mediprime con la inversión de las cuentas publicitarias de Meta **Mediprime Soles** y **MediPrime Soles - Cliente Final**.

Periodo actual: **16 al 30 de setiembre de 2026**.

## Métricas que muestra

| Métrica | Fórmula | Valor del periodo |
|---|---|---|
| Facturación | Suma de VALOR (hoja "VENTAS SEPTIEMBRE") | S/ 114,015.00 |
| Número de ventas | Filas de venta en el rango | 50 (68 unidades) |
| Ticket promedio | Facturación ÷ ventas | S/ 2,280.30 |
| Inversión Meta | Suma de ambas cuentas | S/ 7,118.89 |
| CAC | Inversión ÷ ventas | S/ 142.38 |
| ROAS | Facturación ÷ inversión | 16.0x |
| ROI publicitario | (Facturación − inversión) ÷ inversión | 1,502% |

> El ROI se calcula sobre ingresos, no sobre margen, porque la hoja no incluye el costo de mercadería. Además se asume que todas las ventas vienen de la publicidad; si hay ventas orgánicas o de tienda, el CAC real es mayor y el ROI menor.

## Estructura

```
/
├── index.html   # dashboard completo (estilos, logo y datos embebidos)
└── README.md
```

El `index.html` funciona solo: trae un snapshot de los datos (extraído el 01/10/2026). Si configuras Supabase, lee los datos en vivo y usa el snapshot solo como respaldo.

## Publicar en GitHub Pages

1. Sube `index.html` y `README.md` a la raíz del repositorio (rama `main`).
2. En GitHub: **Settings → Pages → Build and deployment → Source: Deploy from a branch**, rama `main`, carpeta `/ (root)`.
3. En 1–2 minutos queda en `https://<usuario>.github.io/<repositorio>/`.

## Mover a un dominio propio

1. Crea un archivo `CNAME` en la raíz con el dominio, por ejemplo `panel.mediprime.pe`.
2. En el DNS del dominio:
   - Subdominio: registro **CNAME** `panel` → `<usuario>.github.io`
   - Dominio raíz: registros **A** hacia `185.199.108.153`, `185.199.109.153`, `185.199.110.153`, `185.199.111.153`
3. En **Settings → Pages**, escribe el dominio en *Custom domain* y activa **Enforce HTTPS**.
4. En Supabase, agrega el nuevo dominio en **Authentication → URL Configuration** si más adelante usas login.

## Conectar Supabase (datos en vivo)

### 1. Configurar el `index.html`

Busca el bloque `CONFIG` al final del archivo y completa la URL del proyecto (Supabase → Project Settings → API):

```js
const CONFIG = {
  SUPABASE_URL: "https://pzmldthbfhbkorfehoih.supabase.co",
  SUPABASE_KEY: "sb_publishable_Gdpr6ROLF_rAik5j1VkI9w_wzAkqr9V",
  ...
};
```

La clave `sb_publishable_…` está diseñada para usarse en el navegador, así que puede ir en un repo público. **Nunca** pongas aquí la clave `sb_secret_…` (service role).

### 2. Crear las tablas

Ejecuta esto en el **SQL Editor** de Supabase:

```sql
create table public.ventas (
  id bigint generated always as identity primary key,
  fecha date not null,
  producto text not null,
  cantidad int not null default 1,
  valor numeric(12,2) not null,
  created_at timestamptz default now()
);

create table public.inversion_diaria (
  id bigint generated always as identity primary key,
  fecha date not null,
  cuenta text not null,            -- 'Mediprime Soles' | 'MediPrime Soles - Cliente Final'
  inversion numeric(12,2) not null,
  unique (fecha, cuenta)
);

create table public.campanas (
  id bigint generated always as identity primary key,
  campana text not null,
  cuenta text not null,
  inversion numeric(12,2) not null,
  resultados int not null default 0,
  tipo_resultado text not null default 'conversaciones', -- 'conversaciones' | 'leads'
  periodo_desde date,
  periodo_hasta date
);
```

### 3. Seguridad (RLS)

Con la clave publishable, cualquiera que abra la página puede ejecutar las consultas que permitan tus políticas. Activa RLS y deja **solo lectura**:

```sql
alter table public.ventas enable row level security;
alter table public.inversion_diaria enable row level security;
alter table public.campanas enable row level security;

create policy "lectura publica ventas"    on public.ventas           for select using (true);
create policy "lectura publica inversion" on public.inversion_diaria for select using (true);
create policy "lectura publica campanas"  on public.campanas         for select using (true);
```

No crees políticas de `insert`, `update` ni `delete` para el rol `anon`. Carga los datos desde el panel de Supabase (Table Editor → Import CSV) o desde un proceso con la clave secreta.

Si las cifras de facturación no deben ser públicas, protege el panel con Supabase Auth y cambia `using (true)` por `using (auth.role() = 'authenticated')`.

### 4. Cambiar el periodo

Edita `DESDE` y `HASTA` en `CONFIG`. Con Supabase conectado, el panel se recalcula solo con los datos de ese rango.

## Fuentes del snapshot

- **Ventas:** Google Sheet "VENTAS MEDIPRIME", hoja "VENTAS SEPTIEMBRE", 16–30 set 2026.
- **Inversión:** Meta Ads, cuentas Mediprime Soles (S/ 6,582.99) y MediPrime Soles - Cliente Final (S/ 535.90), mismo rango, zona horaria de la cuenta.

## Tecnología

HTML + CSS + JavaScript sin build. Chart.js 4.4.1 (cdnjs), supabase-js 2 (jsDelivr), tipografía Lexend (Google Fonts).
