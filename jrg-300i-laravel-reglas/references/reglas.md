# Reglas de Construcción de Proyectos Laravel

> Guía general de cómo construir un proyecto Laravel de forma **eficiente, escalable, mantenible y reutilizable**.
> Este documento es un conjunto de reglas base; aplicarlas garantiza consistencia en todo el proyecto.
> Todo cambio al proyecto debe quedar registrado (ver `ultimosCambios.md` si existe en el proyecto).

---

## Índice

1. [Arquitectura MVC](#1-arquitectura-mvc)
2. [Estructura de carpetas](#2-estructura-de-carpetas)
3. [Migraciones: orden I, P, R, A, T + ÍNDICES](#3-migraciones-orden-i-p-r-a-t--índices)
4. [Seeders](#4-seeders)
5. [Modelos (M) y Eloquent](#5-modelos-m-y-eloquent)
6. [Controladores (C) — Controladores delgados](#6-controladores-c--controladores-delgados)
7. [Vistas (V) — HTML semántico, Bootstrap y CSS](#7-vistas-v--html-semántico-bootstrap-y-css)
8. [Paleta de colores](#8-paleta-de-colores)
9. [Principios SOLID](#9-principios-solid)
10. [TypeScript](#10-typescript)
11. [Rendimiento: lo que SÍ vuelve lenta la página](#11-rendimiento-lo-que-sí-vuelve-lenta-la-página)
12. [Rendimiento: lo que NO afecta (gratis para la GPU)](#12-rendimiento-lo-que-no-afecta-gratis-para-la-gpu)
13. [Seguridad](#13-seguridad)
14. [Repositorio/Control de versiones](#14-repositoriocontrol-de-versiones)
15. [Iniciar el proyecto con acceso global (cloudflared)](#15-iniciar-el-proyecto-con-acceso-global-cloudflared)
16. [Validación de campos: reglas por campo + jQuery Validate](#16-validación-de-campos-reglas-por-campo--jquery-validate)
17. [CSS Flexbox — Reglas de uso](#17-css-flexbox--reglas-de-uso)
18. [Navegación: botones Cancelar y Volver](#18-navegación-botones-cancelar-y-volver)
19. [Auditoría de acciones de usuarios](#19-auditoría-de-acciones-de-usuarios)
20. [Despliegue en Docker (generalizado)](#20-despliegue-en-docker-generalizado)
21. [Buenas prácticas de código: 6 reglas de calidad](#21-buenas-prácticas-de-código-6-reglas-de-calidad)

---

## 1. Arquitectura MVC

Laravel usa el patrón **Modelo–Vista–Controlador (MVC)**. Cada pieza tiene UNA responsabilidad clara:

- **Modelo (M):** representa una tabla de la base de datos y su lógica de negocio. En `app/Models/`.
- **Vista (V):** solo presentación (HTML/CSS). Sin lógica de negocio. En `resources/views/`.
- **Controlador (C):** el "orquestador": recibe la petición, delega al modelo/servicio y devuelve una vista o JSON. Debe ser **delgado**.

Regla de oro: **vista = presentación, modelo = datos, controlador = coordinación**.

---

## 2. Estructura de carpetas

```
app/
├── Http/
│   ├── Controllers/        # Controladores (delgados, uno por recurso)
│   │   └── Admin/          # Controladores de panel admin/backoffice
│   ├── Requests/           # Request personalizados (validación)
│   └── Middleware/         # Middleware (auth, roles, país, etc.)
├── Models/                 # Modelos Eloquent (uno por tabla de negocio)
├── Services/               # Lógica de negocio reutilizable (fuera de controladores)
├── Providers/              # Providers (registro de servicios, etc.)
└── helpers.php             # Funciones helper globales puras
```

- **Un archivo por clase.**
- Los **Servicios** (`app/Services/`) albergan la lógica compleja y reutilizable: envío de archivos, tasas de cambio, backups, reportes, etc. Los controladores **llaman** a los servicios, no implementan esa lógica.
- Los **Requests** (`app/Http/Requests/`) contienen la validación de entrada; un controlador nunca llena el método `store()` de `if (...) validate(...)`.
- Las **rutas** se organizan por dominio en `routes/` (web, auth, api, console).

---

## 3. Migraciones: orden I, P, R, A, T + ÍNDICES

**Regla obligatoria:** las columnas de cada tabla DEBEN ir en este orden:

| Letra | Sección | Qué contiene | Ejemplo |
|-------|---------|--------------|---------|
| **I** | ID (siempre primero) | `$table->id();` | `$table->id();` |
| **P** | Personal / Datos de negocio | Campos propios de la entidad | `nombre`, `precio`, `estado`, `margen`… |
| **R** | Relaciones | FKs **desacopladas de los modelos, usando strings** | `$table->foreignId('user_id')->constrained('users')->onDelete('cascade');` |
| **A** | Auth (marcar "No aplica" si no hay) | Campos de autenticación (si los hay) | `password`, `remember_token` |
| **T** | Timestamps / Fechas | Fechas personalizadas + `timestamps()` + `softDeletes()` | `fecha_emision`, `$table->timestamps()`, `$table->softDeletes()` |

> **Nota:** Los estados (boolean) los metemos dentro de 'P' o justo antes de 'T' según prefieras.

Ejemplo modelo ([ver `create_facturas_table.php` del proyecto]):

```php
Schema::create('facturas', function (Blueprint $table) {
    // I - ID (siempre primero)
    $table->id();

    // P - Personal / Datos de negocio
    $table->string('serie');
    $table->integer('numero');
    $table->decimal('total', 10, 2);
    // ... más campos de negocio

    // Nota: Los estados (boolean) los metemos dentro de 'P'
    // o justo antes de 'T' según prefieras.
    $table->boolean('estado')->default(true);

    // R - Relaciones (desacoplado de modelos, usando strings)
    $table->foreignId('cliente_id')->constrained('clientes')->onDelete('cascade');
    $table->foreignId('trabajo_id')->nullable()->constrained('trabajos')->onDelete('set null');

    // A - Auth (No aplica en esta tabla)

    // T - Timestamps / Fechas
    $table->date('fecha_emision');
    $table->timestamps();
    $table->softDeletes();
});

// ÍNDICES para búsquedas (clave del rendimiento)
```

### Índices (obligatorio para optimizar consultas)

Importar `DB` para sentencias SQL puras de Postgres:

```php
use Illuminate\Support\Facades\DB;
```

Agregar índices **por cada consulta frecuente** (CRUD + búsquedas). Tipos recomendados:

- **B-Tree** (default): para igualdad y rangos en columnas consultadas mucho.
  `DB::statement('CREATE INDEX idx_facturas_estado ON facturas (estado)');`
- **Compuesto**: para consultas `WHERE col1 = ? AND col2 = ?`.
  `DB::statement('CREATE INDEX idx_facturas_cliente_estado ON facturas (cliente_id, estado)');`
- **UNIQUE**: para integridad + velocidad en claves naturales.
  `DB::statement('CREATE UNIQUE INDEX idx_facturas_serie_numero ON facturas (serie, numero)');`
- **Índice de expresión**: búsquedas por email sin importar mayúsculas.
  `DB::statement('CREATE INDEX idx_clientes_lower_email ON clientes (LOWER(email))');`
- **Índice parcial**: índice diminuto cuando el 80% de tus consultas filtra por una condición.
  `DB::statement("CREATE INDEX idx_trabajos_estado ON trabajos (estado) WHERE estado IN ('presupuesto','en_proceso')");`
- **BRIN**: solo en columnas **ordenadas cronológicamente y de gran tamaño** (>1M filas) — created_at, failed_at. Ocupa ~100KB frente a ~50MB de un B-Tree.
- **GIN**: búsqueda de texto completo (`to_tsvector`) en campos largos.

> **Importante (Postgres):** NO usar `NOW()` ni funciones `VOLATILE` dentro de un índice parcial (Postgres lo rechaza). Para limpiar tokens expirados usa un B-Tree normal en `created_at`.

En el `down()` eliminar los índices personalizados antes de soltar la tabla:

```php
DB::statement('DROP INDEX IF EXISTS idx_facturas_estado');
// ...
Schema::dropIfExists('facturas');
```

### NO crear migraciones "de añadido" sueltas

> **Regla obligatoria:** **NUNCA** generar archivos de migración del tipo
> `2026_08_28_211239_add_original_copia_to_facturas_table.php` (ni `add_X_to_..._table`,
> `create_..._table` para tablas que ya existen, etc.). Los campos nuevos/alteraciones
> de una tabla **se agregan en la migración que crea esa tabla** (`create_facturas_table.php`)
> y no en archivos apartes.
>
> - Cuando crees una tabla o añadas columnas, edita SIEMPRE la migración original de esa tabla.
> - Si ya se generó una migración aparte por error, su contenido debe **fusionarse**
>   dentro de la migración original de la tabla y la migración suelta debe **eliminarse**
>   (rehaciendo con `migrate:fresh --seed` en desarrollo, o con una corrección manual/backup en producción).
> - Mantener un artefacto de migración por tabla mantiene el esquema centralizado y legible.

---

## 4. Seeders

- Un **seeder por tabla** (`UsersSeeder`, `RolesSeeder`, `FacturasSeeder`, …) y un `DatabaseSeeder` que los orquesta en orden de dependencia.
- **Orden correcto:** primero las tablas "madre" (users, roles, clientes, proveedores, materiales) y luego las que dependen (trabajos, facturas, pagos, fotos).
- Los seeders **deben ser idempotentes** y enfocados a **datos de demostración** que permitan probar la app.
- Para `migrate:fresh --seed` funcionar, los seeders respetan los índices UNIQUE (no duplican claves).
- Usar `delete()`/truncate al inicio cuando aplique para evitar duplicados en ejecuciones repetidas.

---

## 5. Modelos (M) y Eloquent

- Nombre en singular y **StudlyCase** (Tabla `facturas` → Modelo `Factura`).
- Declarar `$fillable` (nunca `$guarded = []` a lo loco) y los `$casts` de tipos (`decimal`, `boolean`, `json`, `date`).
- Definir **relaciones** (`belongsTo`, `hasMany`, `belongsToMany`) y usarlas; nunca armar joins a mano en el controlador.
- **No escribir consultas SQL crudas en controladores/vistas**; encapsular la lógica compleja en **Servicios**.
- Aplicar `$hidden` para campos sensibles (password, tokens) al serializar.
- Usar **soft deletes** (`softDeletes()`) para tablas de datos de negocio que admiten papelera; declarar `deleted_at` en el modelo.
- Reglas de validación de creación/edición viven en el **Request**, no en el modelo.

---

## 6. Controladores (C) — Controladores delgados

- **Un controlador por recurso** con las acciones REST (`index`, `create`, `store`, `show`, `edit`, `update`, `destroy`).
- **Nunca debe contener lógica de negocio compleja**; se delega a `app/Services/`.
- **Nunca debe contener validación inline**; se delega al `Request` (form request) correspondiente.
- Respuesta coherente: para peticiones AJAX devuelve JSON, para el resto devuelve `redirect()`/vista.
- Métodos de acceso a datos repetitivos y consultas frecuentes se pueden encapsular (scope en modelo o servicio).
- Coherencia de nombres de rutas, controladores y vistas (`facturas.create` → `FacturaController@create` → `views/facturas/create.blade.php`).

---

## 7. Vistas (V) — HTML semántico, Bootstrap y CSS

### HTML semántico (obligatorio)

- Usar etiquetas semánticas: `<header>`, `<nav>`, `<main>`, `<section>`, `<article>`, `<aside>`, `<footer>` — no `divs` a granel.
- Una sola etiqueta `<main>` por página.
- Encabezados en orden jerárquico (`h1` → `h2` → `h3`), un solo `h1` por página.
- `label` siempre asociado a su `input` (accesibilidad), `alt` en imágenes, `aria-*` en componentes interactivos.
- Usar tablas reales `<table>`, `<thead>`, `<tbody>` para datos tabulares.

### CSS / Layout

- **Bootstrap 5** como base de componentes y grid; las vistas usan el grid de Bootstrap (`container`, `row`, col-*).
- **CSS Box** (caja) y **Media Queries** para responsividad. Modelo de caja: `content-box`/`border-box` definidos de forma global; todo elemento es una caja (margin, border, padding, content).
- Media queries para breakpoints: móvil primero (`min-width`): `576px`, `768px`, `992px`, `1200px`.
- Preferir **CSS Grid / Flexbox** antes que hacks con `float`, `position: absolute` o `margin` negativo.
- Estilos compartidos en el layout base (`layouts/app.blade.php`) como variables CSS (`:root { --c-primary: ...; }`) y clases utilitarias reutilizadas en todo el proyecto.
- **Animaciones suaves (opacity/transform), cero layouts animados** (ver sección de rendimiento).

---

## 8. Paleta de colores

Paleta oficial del proyecto (definida como variables CSS en el layout base):

| Variable | Valor | Uso |
|----------|-------|-----|
| `--c-primary` | `#2563EB` | Color primario (acciones, enlaces, activos) |
| `--c-primary-dark` | `#1D4ED8` | Hover / degradado oscuro |
| `--c-primary-light` | `#3B82F6` | Tints / focus |
| `--bs-navbar-bg` | `linear-gradient(135deg,#1E40AF,#2563EB)` | Barra superior |
| `--bs-success` | `#10B981` | Éxito / pagos completados |
| `--bs-danger` | `#EF4444` | Errores / destrucción |
| `--bs-warning` | `#F59E0B` | Advertencias / pendiente |
| `--bs-info` | `#06B6D4` | Información |
| `--bs-body-bg` | `#F8FAFD` | Fondo de la aplicación |
| `--bs-body-color` | `#1E293B` | Texto principal |
| `--bs-border-color` | `#E2E8F0` | Bordes |
| `--bs-font-sans-serif` | `'DM Sans', system-ui` | Tipografía principal |

Reglas de color:
- **Primario = azul `#2563EB`**. Gradiente de botones: `linear-gradient(135deg, var(--c-primary), var(--c-primary-dark))`.
- Usar siempre **variables CSS**, nunca colores quemados en cada vista.
- Contraste accesible: texto sobre primario = blanco `#fff`.

---

## 9. Principios SOLID

- **S – Responsabilidad única:** cada clase hace una sola cosa. Un controlador no calcula tasas de cambio; eso va en un `Service`.
- **O – Abierto/cerrado:** extender comportamiento sin modificar el código existente. Ej.: nuevo método de tasa en un Service sin tocar el controlador.
- **L – Sustitución de Liskov:** clases hijas sustituyen a la padre sin romper el contrato.
- **I – Segregación de interfaces:** interfaces pequeñas y específicas.
- **D – Inversión de dependencias:** depender de abstracciones, no de implementaciones concretas. Usar el **contenedor de servicios de Laravel** (bind singleton en `AppServiceProvider`, inyección por constructor) en lugar de instanciar dependencias a mano.

Ejemplo aplicado en el proyecto: `TasaCambioService` se registra como singleton en `AppServiceProvider` y se inyecta donde se necesita.

---

## 10. TypeScript

- Los scripts de cliente serios y de lógica compleja se escriben en **TypeScript** (tipado estático, más mantenible y menos propenso a errores), compilado a JS.
- Se compilan con bundler (Vite por defecto en Laravel 11+).
- El **JavaScript vanilla** (jQuery/JS plano en Blade) se reserva solo para mejoras de UX pequeñas e interactividad ligera dentro de las vistas (validaciones, modales, AJAX de formularios).
- Reglas: evitar `any`, definir tipos/interfaces para los datos (p. ej. respuestas AJAX), funciones puras y pequeñas, sin lógica de negocio en el front (eso va en el backend).
- No bloquear el hilo principal: las peticiones de red son asíncronas (fetch/AJAX).

---

## 11. Rendimiento: lo que SÍ vuelve lenta la página (y consume recursos)

- **`backdrop-filter: blur()` (efecto Glassmorphism):** es de las propiedades más pesadas de CSS. Obliga a la GPU a desenfocar **en tiempo real** lo que hay detrás mientras el usuario hace scroll. En móviles de gama media/baja, abusar de `blur()` en varios elementos provoca tirones y sobrecalentamiento. **Evitar o usar con mucha moderación.**
- **Animar propiedades de layout (`width`, `height`, `margin`, `padding`):** si animas el ancho o los márgenes en un `:hover`, el navegador recalcula **todo el Box Model** en cada fotograma (60 veces/segundo) → *Jank* (lag visual). **No animar layout.**
- **Sombras complejas (`box-shadow` difuminados y múltiples):** dibujar sombras gigantes o superpuestas requiere muchos cálculos de pintura (*Paint*). Usar sombras pequeñas y pocas.
- **Consultas N+1 en backend:** evitar `whereHas`/bucles que lanzan una query por fila; usar `with()` (eager loading).
- **Scripts/imágenes pesados sin optimizar:** comprimir imágenes y retrasar cargas no críticas.

---

## 12. Rendimiento: lo que NO afecta (gratis para la GPU)

- **Transformaciones y opacidad (`transform` y `opacity`):** animar `translateY()`, `scale()` o transparencias **no** recalcula el Box Model ni repinta; ocurre directamente en la GPU (etapa *Composite*) y corre a **60 FPS fluidos** incluso en teléfonos económicos. **Preferir SIEMPRE transform/opacity para animar.**
- **Estructura Bento Grid y CSS Grid / Flexbox:** el motor del navegador está hiperoptimizado para distribuir espacio. Un layout Bento o un grid de tarjetas no añade impacto negativo.
- **Bordes sólidos y colores planos (`border`, `background-color`):** pintar bordes sólidos y fondos planos es barato para la GPU.
- **Columnas indexadas:** que una columna esté indexada acelera la consulta (ver sección de migraciones/índices).

---

## 13. Seguridad

- **Nunca** exponer secretos (APP_KEY, contraseñas, tokens) en código o en repositorios.
- Usar **validación por Form Requests** (nunca confiar en la entrada del usuario).
- Escapar salidas en Blade (`{{ }}`) — Blade lo hace por defecto; no usar `{!! !!}` salvo justificación segura.
- Proteger rutas con middleware de **autenticación** y de **roles/permisos**.
- **CSRF** en todos los formularios (`@csrf`) y **verificación de propiedad** en los recursos (que un usuario solo acceda a lo suyo).
- **Visibilidad por rol:** un perfil restringido (ej. tapicero) solo debe ver/editar sus propios recursos. Aplicar la regla en 3 capas: (1) `scope`/filtro en las consultas (ej. `asignadosAlUsuario`), (2) guarda de acceso en el controlador (ej. `puedeVer()` → 403) en cada método que reciba el recurso, y (3) ocultar el elemento en las vistas con `@if(auth()->user()->isAdmin())`. Los módulos exclusivos de admin se protegen envolviendo sus rutas en `Route::middleware(['role:admin'])`.
- `APP_DEBUG=false` en producción. Conexión a BD con credenciales en `.env` (nunca en el código).

---

## 14. Repositorio/Control de versiones

- Incluir `.env` en `.gitignore` (junto a `vendor/` y `node_modules/`).
- Commits pequeños y descriptivos, en el idioma del proyecto.
- Registrar cada cambio importante en `ultimosCambios.md` con versión, fecha, descripción y archivos afectados.
- Documentación/estructura versionada junto con el código para reutilizar el proyecto como plantilla.

---

## 15. Iniciar el proyecto con acceso global (cloudflared)

Esta sección explica, paso a paso, cómo poner el proyecto en línea con **acceso global** usando un **túnel cloudflared** (trycloudflare). Sigue este orden siempre; está pensada para que una IA o un desarrollador lo entienda y lo ejecute sin ambigüedad.

### Objetivo

Que la aplicación Laravel (que corre en un servidor local) sea accesible desde **cualquier dispositivo fuera de la red local** mediante una **URL pública** de Cloudflare, sin necesidad de abrir puertos ni tener IP pública.

### Prerrequisitos (antes de empezar)

1. **PostgreSQL** activo y con las credenciales correctas en `.env` (`DB_HOST`, `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD`).
2. Migraciones aplicadas (`php8.4 artisan migrate`).
3. El binario de cloudflared instalado. En este entorno está en: `/home/jdrodriguezg/.local/bin/cloudflared`.

> **CRÍTICO:** usar **`php8.4`** para todos los comandos `php`/`artisan`. El `php` por defecto del sistema es **php7.4** y NO sirve para este proyecto.

### Paso 1 — Levantar el servidor local de Laravel

Ejecutar (si no está ya corriendo):

```bash
nohup php8.4 artisan serve --host=0.0.0.0 --port=8002 > /tmp/opencode/serve.log 2>&1 &
```

Explicación de cada parámetro:
- `nohup ... &` → el proceso sigue corriendo en segundo plano aunque se cierre la terminal.
- `--host=0.0.0.0` → escucha en todas las interfaces de red (obligatorio para que el túnel se pueda conectar).
- `--port=8002` → puerto del servidor (debe coincidir con el del túnel).
- `> /tmp/opencode/serve.log 2>&1` → guarda la salida y los errores en un log.

Verificar que está a la escucha:

```bash
ss -ltnp | grep 8002
# Debe mostrar: LISTEN 0.0.0.0:8002 ...
```

### Paso 2 — Levantar el túnel cloudflared (URL pública)

Ejecutar:

```bash
nohup /home/jdrodriguezg/.local/bin/cloudflared tunnel --url http://localhost:8002 > /tmp/opencode/tunnel.log 2>&1 &
```

Explicación:
- `--url http://localhost:8002` → el túnel redirige el tráfico público al servidor local.
- `> /tmp/opencode/tunnel.log 2>&1` → guarda la salida y, muy importante, **contiene la URL generada**.

### Obtención de la URL pública (IMPORTANTE)

Leer el log del túnel para obtener la URL generada:

```bash
cat /tmp/opencode/tunnel.log
```

Buscar la línea que contiene **"Your quick Tunnel has been created"**; justo debajo aparece la URL:

```
https://XXXX-XXXX.trycloudflare.com
```

Esa es la **URL pública de acceso global** (p. ej. `https://built-financial-choice-explicitly.trycloudflare.com`).

> **⚠️ ADVERTENCIA CLAVE:** la URL de un túnel **quick trycloudflare es ALEATORIA y EFÍMERA**. **Cambia en CADA reinicio** del proceso de cloudflared.
>
> - Si el proceso de cloudflared se cae o se reinicia (corte de luz, reboot, etc.), **hay que volver a leer la URL nueva** del log tal como se explica arriba, porque la anterior **ya no funciona**.
> - Actualizar siempre esa URL en la documentación (p. ej. en `ultimosCambios.md`) para no trabajar con una URL vieja.

### Paso 3 — Verificación

Comprobar que todo responde correctamente:

```bash
# Acceso local (debe devolver 200)
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8002/login

# Acceso global a través del túnel (debe devolver 200)
# Sustituir <URL-DEL-TUNEL> por la URL obtenida en el Paso 2
curl -s -o /dev/null -w "%{http_code}\n" https://<URL-DEL-TUNEL>/login
```

Si ambos devuelven `200`, el proyecto está en línea y accesible desde cualquier parte del mundo con la URL pública.

### Nota sobre producción

Los túneles **quick trycloudflare son GRATUITOS pero NO tienen garantía de uptime**; son ideales para pruebas y experimentación. Para producción real se recomienda un **túnel named (con nombre)** y un **dominio propio** configurado en Cloudflare. Un dominio propio cuesta alrededor de 1–15 USD/año.

---

> **Importante:** estas reglas son la base para construir **cualquier proyecto Laravel eficiente** y se pueden reutilizar como plantilla. Mantener consistencia en MVC, migraciones (I-P-R-A-T + índices), seeders, servicios, vistas semánticas, paleta de colores, SOLID, TypeScript, rendimiento, el arranque con cloudflared y el despliegue con Docker (ver sección 20).

---

## 16. Validación de campos: reglas por campo + jQuery Validate + Máscaras

### 16.1 Dependencias obligatorias (todo proyecto Laravel)

Todo proyecto Laravel **debe incluir** estas dos librerías jQuery en `public/js/` y cargarlas globalmente en `layouts/app.blade.php` después de jQuery:

| Librería | Archivo | Propósito |
|----------|---------|-----------|
| **jQuery Validate** | `public/js/jquery.validate.min.js` | Validación en tiempo real de formularios |
| **jQuery Mask** | `public/js/jquery.mask.min.js` | Máscaras de formato en inputs (teléfono, email, cédula) |

**Orden de carga obligatorio en el layout:**

```html
<script src="https://code.jquery.com/jquery-3.7.1.min.js"></script>
<script src="{{ asset('js/jquery.validate.min.js') }}"></script>
<script src="{{ asset('js/jquery.mask.min.js') }}"></script>
<script src="https://cdn.jsdelivr.net/npm/sweetalert2@11"></script>
```

> **Regla:** Estas librerías **siempre** se usan. No crear formularios sin validación JS ni máscaras de formato. Copiar los archivos `.min.js` de `public/js/` del proyecto base si no existen.

### 16.2 Uso en una vista

Cada formulario que requiera validación JS debe usar `@push('scripts')` al final del archivo:

```blade
@push('scripts')
<script>
$(function() {
    $('#miFormulario').validate({
        rules: {
            campo_nombre: { required: true, minlength: 2, maxlength: 100 },
            campo_email:  { required: true, email: true },
        },
        messages: {
            campo_nombre: { required: 'El nombre es obligatorio.', minlength: 'Mínimo 2 caracteres.' },
            campo_email:  { required: 'El email es obligatorio.', email: 'Ingrese un email válido.' },
        },
        errorClass: 'is-invalid',
        errorElement: 'div',
        errorPlacement: function(error, element) {
            error.addClass('invalid-feedback');
            element.closest('.mb-3, .col-md-6').append(error);
        },
        highlight: function(element) {
            $(element).addClass('is-invalid').removeClass('is-valid');
        },
        unhighlight: function(element) {
            $(element).removeClass('is-invalid').addClass('is-valid');
        }
    });
});
</script>
@endpush
```

### 16.3 Validadores jQuery Validate disponibles

| Validador | Descripción | Ejemplo |
|-----------|-------------|---------|
| `required` | Campo obligatorio | `{ required: true }` |
| `email` | Formato email válido | `{ email: true }` |
| `minlength(n)` | Mínimo n caracteres | `{ minlength: 2 }` |
| `maxlength(n)` | Máximo n caracteres | `{ maxlength: 100 }` |
| `min(n)` | Valor numérico mínimo | `{ min: 0 }` |
| `max(n)` | Valor numérico máximo | `{ max: 99999 }` |
| `digits` | Solo dígitos (0-9) | `{ digits: true }` |
| `number` | Número válido (acepta decimales) | `{ number: true } }` |
| `equalTo('#id')` | Igual a otro campo | `{ equalTo: '#password' }` |
| `pattern` | Expresión regular (HTML5) | Ver 16.4 |

### 16.4 Validadores custom (definir antes del `.validate()`)

```js
$.validator.addMethod('lettersOnly', function(value, element) {
    return this.optional(element) || /^[a-zA-ZáéíóúñÁÉÍÓÚÑ\s]+$/.test(value);
}, 'Ingrese solo letras.');

$.validator.addMethod('phoneVE', function(value, element) {
    return this.optional(element) || /^\d{4}-?\d{7}$/.test(value.replace(/[\s\-()]/g, ''));
}, 'Formato: 0412-0000000');

$.validator.addMethod('cedulaVE', function(value, element) {
    return this.optional(element) || /^\d{6,12}$/.test(value);
}, 'La cédula debe tener entre 6 y 12 dígitos.');
```

### 16.5 Reglas de validación por campo (estándar del proyecto)

Estas reglas aplican a **todos** los proyectos Laravel del entorno. Cada campo tiene regla en **3 capas**: BD (migración), Backend (FormRequest), Frontend (jQuery Validate + HTML5).

---

#### Cédula / Documento de identidad

| Capa | Regla |
|------|-------|
| **BD** | `string` (VARCHAR 255), `unique` |
| **Backend (FormRequest)** | `required`, `digits_between:6,12`, `unique:tabla,cedula,{id}` (ignorar自身 en update) |
| **Frontend (HTML)** | `type="text"`, `maxlength="12"`, `pattern="[0-9]{6,12}"` |
| **Frontend (jQuery)** | `required: true, cedulaVE: true` (custom) |
| **Label** | "Cédula *" (siempre con acento, nunca "DNI/CIF") |
| **Posición** | **PRIMER campo** del formulario (antes de nombre) |
| **Placeholder** | `"Ej: 12345678"` |

```php
// FormRequest
'cedula' => 'required|digits_between:6,12|unique:personas,cedula,' . $id,
```

---

#### Nombre

| Capa | Regla |
|------|-------|
| **BD** | `string` (VARCHAR 255) |
| **Backend** | `required`, `string`, `min:2`, `max:100` |
| **Frontend (HTML)** | `type="text"`, `maxlength="100"`, `pattern="[a-zA-ZáéíóúñÁÉÍÓÚÑ\s]+"` |
| **Frontend (jQuery)** | `required: true, minlength: 2, maxlength: 100, lettersOnly: true` |

```php
'nombre' => 'required|string|min:2|max:100',
```

---

#### Apellido

| Capa | Regla |
|------|-------|
| **BD** | `string` (VARCHAR 255) |
| **Backend** | `required`, `string`, `min:2`, `max:100` |
| **Frontend (HTML)** | `type="text"`, `maxlength="100"`, `pattern="[a-zA-ZáéíóúñÁÉÍÓÚÑ\s]+"` |
| **Frontend (jQuery)** | `required: true, minlength: 2, maxlength: 100, lettersOnly: true` |

```php
'apellido' => 'required|string|min:2|max:100',
```

---

#### Email / Correo electrónico

| Capa | Regla |
|------|-------|
| **BD** | `string`, `unique` |
| **Backend** | `required`, `email`, `max:255`, `unique:tabla,email,{id}` |
| **Frontend (HTML)** | `type="email"`, `maxlength="255"`, `placeholder="correo@ejemplo.com"` |
| **Frontend (jQuery)** | `required: true, email: true, maxlength: 255` |
| **Label** | "Email *" (nunca "Correo:", "Em@il:", etc.) |

```php
'email' => 'required|email|max:255|unique:clientes,email,' . $id,
```

---

#### Teléfono

| Capa | Regla |
|------|-------|
| **BD** | `string` (VARCHAR 255) |
| **Backend** | `nullable`, `string`, `min:7`, `max:15`, `regex:/^[+]?[\d\s\-()]+$/` |
| **Frontend (HTML)** | `type="text"`, `maxlength="15"`, placeholder `"0412-0000000"` |
| **Frontend (jQuery)** | `phoneVE: true` (custom, solo si tiene valor) |
| **Máscara jQuery** | `(0000)-000.00.00` plugin `jquery.mask` o jQuery Format Plugin |
| **Limpieza antes de submit** | `$(this).val($(this).cleanVal())` para enviar solo dígitos |

```php
'telefono' => 'nullable|string|min:7|max:15|regex:/^[+]?[\d\s\-()]+$/',
```

---

#### Dirección

| Capa | Regla |
|------|-------|
| **BD** | `string`, `nullable` |
| **Backend** | `nullable`, `string`, `min:5`, `max:255` |
| **Frontend (HTML)** | `type="text"` (o `<textarea rows="2">`), `maxlength="255"` |
| **Frontend (jQuery)** | `minlength: 5, maxlength: 255` (solo si tiene valor) |

```php
'direccion' => 'nullable|string|min:5|max:255',
```

---

#### Ciudad

| Capa | Regla |
|------|-------|
| **BD** | `string`, `nullable` |
| **Backend** | `nullable`, `string`, `max:80` |
| **Frontend (HTML)** | `type="text"`, `maxlength="80"` |

```php
'ciudad' => 'nullable|string|max:80',
```

---

#### Código Postal

| Capa | Regla |
|------|-------|
| **BD** | `string`, `nullable` |
| **Backend** | `nullable`, `string`, `max:10` |
| **Frontend (HTML)** | `type="text"`, `maxlength="10"` |

```php
'codigo_postal' => 'nullable|string|max:10',
```

---

### 16.6 Máscaras de formato (jquery.mask)

**Regla:** Todo campo de teléfono, email o cédula debe tener máscara visual. La librería `jquery.mask.min.js` ya está incluida globalmente (ver 16.1).

#### Máscara de Teléfono

Formato venezolano: `0412-0000000` (4 dígitos código + 7 dígitos número).

```js
// Inicializar máscara
$('#telefono').mask('0000-0000000', { placeholder: '0412-0000000' });
```

**Patrones disponibles:**

| Patrón | Ejemplo | Uso |
|--------|---------|-----|
| `0000-0000000` | `0412-8340975` | **Recomendado** — Venezuela |
| `(0000)-000.00.00` | `(0412)-834.09.75` | Alternativo Venezuela |
| `0000-0000` | `0412-8340` | Solo código (si aplica) |

#### Máscara de Email

No requiere máscara de caracteres, pero se debe usar `type="email"` en HTML para validación nativa del navegador:

```html
<input type="email" name="email" maxlength="255" placeholder="correo@ejemplo.com">
```

#### Máscara de Cédula

Solo dígitos, sin formato especial. La validación `cedulaVE` (custom) se encarga del formato:

```html
<input type="text" name="dni_cif" maxlength="12" placeholder="Ej: 12345678">
```

```js
// Solo permitir dígitos mientras escribe (opcional, la validación JS ya lo hace)
$('#dni_cif').on('input', function() {
    $(this).val($(this).val().replace(/\D/g, ''));
});
```

#### IMPORTANTE: Limpiar máscara antes de enviar

La máscara guarda formato visual (`0412-8340975`), pero en BD se debe guardar solo dígitos (`04128340975`). **Siempre** limpiar antes del submit:

```js
$('#miFormulario').on('submit', function() {
    $('#telefono').val($('#telefono').val().replace(/\D/g, ''));
});
```

O con `.cleanVal()` si se usa `jquery.mask`:

```js
$('#miFormulario').on('submit', function() {
    $('#telefono').val($('#telefono').cleanVal());
});
```

#### Ejemplo completo en vista

```blade
@push('scripts')
<script>
$(function() {
    // Máscaras
    $('#telefono').mask('0000-0000000', { placeholder: '0412-0000000' });

    // Validación
    $('#formCliente').validate({
        rules: {
            telefono: { phoneVE: true }
        }
    });

    // Limpiar antes de enviar
    $('#formCliente').on('submit', function() {
        $('#telefono').val($('#telefono').cleanVal());
    });
});
</script>
@endpush
```

### 16.7 Orden de campos en formularios de clientes

El orden correcto de los campos al crear/editar un cliente es:

1. **Cédula** (primer campo, obligatorio)
2. Nombre
3. Apellido
4. Email
5. Teléfono
6. Tipo de Cliente
7. Dirección
8. Ciudad
9. Código Postal
10. Notas
11. Estado (Activo/Inactivo)

### 16.8 Resumen de longitudes por campo (referencia rápida)

| Campo | `maxlength` HTML | `max` Backend | `min` Backend | `required` |
|-------|-------------------|---------------|---------------|------------|
| Cédula | 12 | `digits_between:6,12` | 6 | Sí |
| Nombre | 100 | 100 | 2 | Sí |
| Apellido | 100 | 100 | 2 | Sí |
| Email | 255 | 255 | — | Sí |
| Teléfono | 15 | 15 | 7 | No |
| Dirección | 255 | 255 | 5 | No |
| Ciudad | 80 | 80 | — | No |
| Código Postal | 10 | 10 | — | No |
| Notas | 500 | 500 | — | No |

---

## 17. CSS Flexbox — Reglas de uso

Flexbox se usa para alinear y distribuir elementos dentro de un contenedor. **Siempre preferir Flexbox antes que hacks con `float`, `position: absolute` o `margin` negativo.**

### 17.1 Dirección del flex container

| Regla | Clase Bootstrap | CSS nativo | Cuándo usarlo |
|-------|-----------------|------------|---------------|
| **1. `flex-row`** (default) | `d-flex` | `display:flex; flex-direction:row;` | Elementos en línea horizontal, de izquierda a derecha. **Es el default, no necesita clase extra.** |
| **2. `flex-row-reverse`** | `d-flex flex-row-reverse` | `flex-direction:row-reverse;` | Elementos en línea horizontal, de derecha a izquierda. Útil para alinear acciones a la derecha manteniendo el orden DOM. |
| **3. `flex-column`** | `d-flex flex-column` | `flex-direction:column;` | Elementos apilados verticalmente. Para formularios, tarjetas, listas verticales. |

```html
<!-- 1. Row (default) — elementos en línea -->
<div class="d-flex">
    <span>Izquierda</span>
    <span>Derecha</span>
</div>

<!-- 2. Row reverse — acciones a la derecha -->
<div class="d-flex flex-row-reverse">
    <button>Cancelar</button>
    <button>Guardar</button>
</div>

<!-- 3. Column — apilado vertical -->
<div class="d-flex flex-column">
    <label>Nombre</label>
    <input type="text">
</div>
```

### 17.2 Justificación (eje principal — horizontal en row)

| Regla | Clase Bootstrap | CSS nativo | Cuándo usarlo |
|-------|-----------------|------------|---------------|
| **4. `justify-content-*`** | `justify-content-between` | `justify-content:space-between;` | Distribuir espacio entre elementos: primero a la izquierda, último a la derecha. **El más usado.** |
| | `justify-content-start` | `justify-content:flex-start;` | Todos alineados al inicio (izquierda). |
| | `justify-content-end` | `justify-content:flex-end;` | Todos alineados al final (derecha). |
| | `justify-content-center` | `justify-content:center;` | Todos centrados. |
| | `justify-content-around` | `justify-content:space-around;` | Espacio uniforme alrededor de cada elemento. |
| | `justify-content-evenly` | `justify-content:space-evenly;` | Espacio completamente uniforme. |

```html
<!-- 4. Justify — barra de acciones: título izquierda, botones derecha -->
<div class="d-flex justify-content-between align-items-center">
    <h5 class="mb-0">Título</h5>
    <div class="d-flex gap-2">
        <button>Cancelar</button>
        <button>Guardar</button>
    </div>
</div>
```

### 17.3 Alineación (eje transversal — vertical en row)

| Regla | Clase Bootstrap | CSS nativo | Cuándo usarlo |
|-------|-----------------|------------|---------------|
| **5. `align-items-center`** | `align-items-center` | `align-items:center;` | Centrar elementos verticalmente dentro del flex container. **El más usado para alinear íconos con texto, botones con labels, tarjetas en fila.** |
| | `align-items-start` | `align-items:flex-start;` | Todos arriba. |
| | `align-items-end` | `align-items:flex-end;` | Todos abajo. |
| | `align-items-stretch` | `align-items:stretch;` | Estirar para igualar altura (default). |
| | `align-self-center` | `align-self:center;` | Centrar solo un hijo específico. |

```html
<!-- 5. Align items center — ícono alineado con texto -->
<div class="d-flex align-items-center gap-2">
    <i class="fas fa-user"></i>
    <span>Nombre del cliente</span>
</div>

<!-- Combinación más común: header de tarjeta -->
<div class="card-header d-flex justify-content-between align-items-center">
    <h5 class="mb-0"><i class="fas fa-cog me-2"></i>Configuración</h5>
    <button class="btn btn-sm btn-primary">Guardar</button>
</div>
```

### 17.4_GAP_—_espaciado_entre_elementos

| Clase Bootstrap | CSS nativo | Descripción |
|-----------------|------------|-------------|
| `gap-1` | `gap: 0.25rem;` | 4px |
| `gap-2` | `gap: 0.5rem;` | 8px |
| `gap-3` | `gap: 1rem;` | 16px |
| `gap-4` | `gap: 1.5rem;` | 24px |
| `gap-5` | `gap: 2rem;` | 32px |

> **Regla:** Usar `gap-*` en vez de `margin` en hijos para espaciar elementos flex. Es más limpio y predecible.

### 17.5 Combinaciones Flexbox más comunes en el proyecto

```html
<!-- Header de tarjeta: título + botones -->
<div class="card-header d-flex justify-content-between align-items-center">
    <h5 class="mb-0">Título</h5>
    <div class="d-flex gap-2">Botones...</div>
</div>

<!-- Fila de formulario: 2 campos lado a lado -->
<div class="row">
    <div class="col-md-6 mb-3">Campo 1</div>
    <div class="col-md-6 mb-3">Campo 2</div>
</div>

<!-- Badge + texto alineados -->
<div class="d-flex align-items-center gap-2">
    <span class="badge bg-success">Activo</span>
    <span class="text-muted small">Desde 01/01/2026</span>
</div>

<!-- Botones apilados verticalmente (sidebar) -->
<div class="d-flex flex-column gap-2">
    <a class="btn btn-primary">Opción 1</a>
    <a class="btn btn-outline-secondary">Opción 2</a>
</div>

<!-- Acciones a la derecha, contenido a la izquierda -->
<div class="d-flex justify-content-between align-items-center mb-3">
    <div>
        <h4 class="mb-0">Título</h4>
        <small class="text-muted">Subtítulo</small>
    </div>
    <a href="#" class="btn btn-primary">Acción</a>
</div>
```

### 17.6 Regla de decisión: ¿cuándo usar Flexbox?

| Situación | Solución |
|-----------|----------|
| 2+ elementos en fila, alineados verticalmente | `d-flex align-items-center` |
| Header con título a la izquierda, botones a la derecha | `d-flex justify-content-between align-items-center` |
| Elementos apilados verticalmente | `d-flex flex-column` |
| Botones/acciones en fila con espacio entre ellos | `d-flex gap-2` |
| Ícono junto a texto (checkbox, badges, labels) | `d-flex align-items-center gap-2` |
| Invertir orden visual sin cambiar DOM | `d-flex flex-row-reverse` |

---

## 18. Navegación: botones Cancelar y Volver

### 18.1 Regla general

**Todo botón "Cancelar" o "Volver" debe regresar a la página anterior real del usuario**, no a una ruta fija. Se usa `url()->previous()` de Laravel.

```php
// ❌ MAL — ruta fija, pierde el contexto
<a href="{{ route('trabajos.index') }}">Cancelar</a>

// ✅ BIEN — regresa de donde vino
<a href="{{ url()->previous() }}">Cancelar</a>
```

### 18.2 Tipos de botones de navegación

| Botón | Comportamiento | Ejemplo |
|-------|----------------|---------|
| **"Cancelar"** (en formularios) | `url()->previous()` | Cancelar creación/edición de factura, trabajo, material, cliente |
| **"Volver"** (en vistas show) | `url()->previous()` | Volver desde vista detalle de trabajo, cliente, material |
| **"Editar"** (navegación directa) | `route('entidad.edit', $id)` | Botón que lleva al formulario de edición |
| **"Ver"** (navegación directa) | `route('entidad.show', $id)` | Botón que lleva a la vista detalle |

### 18.3 Formularios que deben usar `url()->previous()`

| Vista | Botón | Antes (❌) | Ahora (✅) |
|-------|-------|-----------|-----------|
| `facturas/create` | Cancelar | `route('facturas.index')` | `url()->previous()` |
| `facturas/edit` | Cancelar | `route('facturas.show')` | `url()->previous()` |
| `trabajos/create` | Cancelar | `route('trabajos.index')` | `url()->previous()` |
| `trabajos/edit` | Volver | `route('trabajos.show')` | `url()->previous()` |
| `materiales/create` | Cancelar | `route('materiales.index')` | `url()->previous()` |
| `materiales/edit` | Cancelar | `route('materiales.index')` | `url()->previous()` |
| `clientes/create` | Volver / Cancelar | `route('clientes.index')` | `url()->previous()` |
| `clientes/edit` | Volver | `route('clientes.index')` | `url()->previous()` |
| `admin/users/create` | Volver | `route('admin.users.index')` | `url()->previous()` |
| `admin/users/edit` | Volver | `route('admin.users.index')` | `url()->previous()` |

### 18.4 Vistas show que deben tener botón "Volver"

Toda vista `show.blade.php` **debe** incluir un botón "Volver" con `url()->previous()` en el header:

```blade
<div class="btn-group">
    <a href="{{ url()->previous() }}" class="btn btn-outline-secondary">
        <i class="fas fa-arrow-left me-1"></i>Volver
    </a>
    <a href="{{ route('entidad.edit', $entidad) }}" class="btn btn-secondary">
        <i class="fas fa-edit me-2"></i>Editar
    </a>
</div>
```

### 18.5 Eliminar botones redundantes

**No duplicar información.** Si una vista ya muestra todos los datos de una entidad (ej: `trabajos/cliente.blade.php` muestra nombre, email, teléfono, ciudad del cliente), no agregar un botón "Ficha" que lleve a otra vista con la misma información.

| Vista | Botón eliminado | Razón |
|-------|-----------------|-------|
| `trabajos/cliente.blade.php` | "Ficha" (→ `clientes.show`) | Redundante: los datos del cliente ya se muestran en la tarjeta superior |

---

## 19. Auditoría de acciones de usuarios

> **Regla obligatoria:** **TODAS las acciones de TODOS los usuarios deben quedar registradas** en un módulo de **auditoría** dentro del sistema. Este registro sirve para recordar y llevar un **control minucioso** de todo lo que se haga en el sistema.

### 19.1 Qué se registra

Cada acción relevante (crear, leer, editar, eliminar, emitir/cancelar, iniciar/cerrar sesión, cambios de estado, etc.) debe guardar al menos:

- **Usuario** que realizó la acción (o `system`/`guest` si es pública).
- **Acción** (crear, actualizar, eliminar, emitir, login, logout, etc.).
- **Entidad/Recurso** afectado (ej: `factura`, `trabajo`, `cliente`, `usuario`) y su **ID**.
- **Descripción** legible de lo que se hizo.
- **Datos previos/cambios** relevantes (dato anterior → dato nuevo) cuando aplique.
- **Fecha y hora** exactas del evento (timestamp).
- **IP de la máquina** desde la que se realizó la acción (`request()->ip()`).

### 19.2 Acceso restringido al administrador

- El **módulo de auditoría SOLO lo puede ver el usuario administrador** (`role:admin`).
- Rutas del módulo bajo middleware `role:admin` (o equivalente de jerarquía).
- Debe permitir **búsqueda y filtrado** por usuario, acción, entidad, rango de fechas e IP.

### 19.3 Cómo implementarlo

- Un **modelo** `Auditoria` (+ migración y tabla `auditorias`) con los campos del punto 19.1.
- Un **servicio** centralizado: `AuditoriaService::registrar($accion, $entidad, $id, $descripcion, $datosAntes, $datosDespues)` que inyecte automáticamente `user_id`, `ip` y timestamp.
- Registrar las acciones en controladores/servicios como parte del flujo normal (no en vistas ni consultas fuera de servicios).
- La tabla `auditorias` **no debe tener soft deletes ni edición**: es un registro inmutable de seguridad; solo se consulta (nunca se elimina vía la app).
- Índices para el admin: `user_id`, `entidad`, `created_at` (y compuesto `entidad + created_at`).
- **Seedear** registros de ejemplo (Seeder `AuditoriasSeeder`) para probar la vista del admin.

### 19.4 Vista de administración

- Página `auditorias/index` (solo admin) con eventos ordenados por fecha descendente.
- Filtros: usuario, acción, entidad, IP y rango de fechas.
- Mostrar claramente: fecha/hora, usuario, acción, entidad + ID, IP y descripción.

---

## 20. Despliegue en Docker (generalizado)

> **Regla obligatoria:** TODA aplicación del entorno **debe poder desplegarse con Docker** en un comando. Esta sección define la manera generalizada: la imagen de la app y su base de datos son **contenedores separados** que se comunican por una **red Docker**. Aplica a cualquier proyecto Laravel (u otro framework) cambiando el nombre de la imagen, los contenedores y las variables de entorno.

### 20.1 Objetivo

Que la aplicación funcione en **cualquier máquina** sin instalar PHP, Composer, Nginx ni la base de datos en el host: solo hace falta Docker. Artefactos del proyecto:

| Archivo | Propósito |
|---------|-----------|
| `Dockerfile` | Construye la imagen de la app (código + PHP + Nginx + supervisor) |
| `docker-compose.yml` | Orquesta app + base de datos (si hay `docker compose`) |
| `docker-up.sh` | Sube todo en **un comando** con el CLI puro de Docker |
| `docker/entrypoint.sh` | Comandos al arrancar el contenedor (caches + migraciones + process manager) |
| `docker/default.conf` | Configuración de Nginx (front controller de Laravel) |
| `docker/supervisord.conf` | Mantiene vivos nginx + php-fpm dentro del mismo contenedor |

### 20.2 Concepto clave: imagen + BD + red

Docker aísla la app y la base de datos. NO se conectan por `localhost` (cada contenedor tiene su propio localhost); se conectan por el **nombre del contenedor** a través de una **red compartida**:

```
[docker network: app-net]
  [db-container]  PostgreSQL  ←— DB_HOST=db-container —→  [app-container] nginx+PHP+app :80
```

- `-p 8080:80` expone SOLO el puerto 80 del contenedor de la app al host.
- La BD **no se expone** al host (solo red interna), salvo que se necesite un cliente local externo.

### 20.3 Dockerfile generalizado (por capas)

```dockerfile
FROM php:8.4-fpm

# 1. Dependencias del sistema: git/zip/unzip (composer), libpq-dev (driver pgsql),
#    nginx (servidor web), supervisor (mantener 2 procesos vivos)
RUN apt-get update && apt-get install -y --no-install-recommends \
    git zip unzip libpq-dev libzip-dev nginx supervisor curl \
    && rm -rf /var/lib/apt/lists/*

# 2. Extensiones PHP que TU app necesite (pgsql, zip, etc.).
#    ⚠️ Si la app usa PostgreSQL (jsonb/GIN/to_tsvector) es OBLIGATORIO pdo_pgsql;
#    SQLite no soporta esas features y fallará al migrar.
RUN docker-php-ext-install pdo_pgsql pgsql zip

# 3. Composer desde su imagen oficial (no instalarlo con curl)
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

# 4. Código de la aplicación
WORKDIR /var/www/html
COPY . .

# 5. Config por defecto SIN secretos (las env vars del proveedor prevalecen sobre este)
RUN cp .env.example .env

# 6. Dependencias de producción (--no-dev: sin sail, pint, pail...)
RUN composer install --no-dev --optimize-autoloader \
    && php artisan key:generate --force

# 7. Permisos: los procesos web escriben en storage/ y bootstrap/cache
RUN chown -R www-data:www-data storage bootstrap/cache \
    && chmod -R 775 storage bootstrap/cache \
    && mkdir -p /run/php \
    && chown www-data:www-data /run/php

# 8. Configuración interna del contenedor (nginx, php-fpm, supervisor, entrypoint)
COPY docker/default.conf /etc/nginx/sites-available/default
COPY docker/phpfpm.conf /usr/local/etc/php-fpm.d/zz-app.conf
COPY docker/supervisord.conf /etc/supervisor/conf.d/app.conf
COPY docker/entrypoint.sh /usr/local/bin/entrypoint.sh
RUN chmod +x /usr/local/bin/entrypoint.sh

EXPOSE 80
CMD ["/usr/local/bin/entrypoint.sh"]
```

> **Nota PHP-FPM:** la imagen oficial `php:*-fpm` hace que php-fpm escuche en TCP 9000; para que lo haga por **socket** (rápido y estándar con Nginx), crear `docker/phpfpm.conf`:
>
> ```ini
> [www]
> listen = /run/php/php8.4-fpm.sock
> ```
> y copiarlo como `zz-*.conf` (se carga después que la config por defecto de la imagen).

### 20.4 `docker-compose.yml` generalizado

```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      APP_ENV: local
      APP_DEBUG: "true"
      APP_URL: http://localhost:8080
      APP_KEY: ${APP_KEY:-base64:cambialo==}
      DB_CONNECTION: pgsql
      DB_HOST: db            # ⚠️ nombre del servicio BD, NO localhost
      DB_PORT: "5432"
      DB_DATABASE: app_db
      DB_USERNAME: postgres
      DB_PASSWORD: changeme
      SESSION_DRIVER: database
      CACHE_STORE: database
      QUEUE_CONNECTION: sync
    ports:
      - "8080:80"
    volumes:
      - ./storage:/var/www/html/storage
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16
    environment:
      POSTGRES_DB: app_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: "changeme"
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 10

volumes:
  db-data:
```

### 20.5 `docker/entrypoint.sh` generalizado

El contenedor NO usa `php artisan serve`; el entrypoint cachea config/rutas/vistas, aplica migraciones y deja a supervisord manteniendo nginx + php-fpm:

```bash
#!/usr/bin/env bash
set -e

echo "==> Caching configuration"
php artisan config:cache
php artisan event:cache
php artisan route:cache
php artisan view:cache

echo "==> Running migrations"
php artisan migrate --force

echo "==> Starting supervisord"
exec /usr/bin/supervisord -c /etc/supervisor/supervisord.conf
```

> **⚠️ `route:cache` falla con closures:** si `routes/web.php` o `routes/api.php` tienen rutas definidas con `Closure` (ej. `Route::get('/', fn() => view(...))`), `route:cache` lanza error y el contenedor NO arranca. Convertir TODAS las rutas a **controladores** antes de dockerizar.

### 20.6 `docker-up.sh`: todo en un comando (CLI puro)

Para máquinas sin `docker compose` (ni plugin, ni `docker-compose`). Generalizado:

```bash
#!/usr/bin/env bash
set -e

IMAGE="tu-app"
NETWORK="app-net"
DB_NAME="app_db"
DB_USER="postgres"
DB_PASS="changeme"
APP_PORT="8080"
DB_CONTAINER="app-db"
APP_CONTAINER="app"

echo "==> 1) Red Docker (la app y la BD se hablan entre sí)"
docker network create "$NETWORK" 2>/dev/null || true

echo "==> 2) Contenedor PostgreSQL (la base de datos)"
if ! docker ps --format '{{.Names}}' | grep -q "^$DB_CONTAINER$"; then
    docker rm -f "$DB_CONTAINER" >/dev/null 2>&1 || true
    docker run -d --name "$DB_CONTAINER" --network "$NETWORK" \
        -e POSTGRES_DB="$DB_NAME" \
        -e POSTGRES_USER="$DB_USER" \
        -e POSTGRES_PASSWORD="$DB_PASS" \
        postgres:16 >/dev/null
    echo "    Base de datos arrancando... (esperando que esté lista)"
    sleep 6
else
    echo "    $DB_CONTAINER ya está corriendo."
fi

echo "==> 3) Construir la imagen de la app (Dockerfile)"
docker build -t "$IMAGE" .

echo "==> 4) Contenedor de la app (nginx + PHP + Laravel)"
docker rm -f "$APP_CONTAINER" >/dev/null 2>&1 || true
docker run -d --name "$APP_CONTAINER" --network "$NETWORK" -p "$APP_PORT:80" \
    -e APP_ENV=local \
    -e APP_DEBUG=true \
    -e APP_URL="http://localhost:$APP_PORT" \
    -e DB_CONNECTION=pgsql \
    -e DB_HOST="$DB_CONTAINER" \
    -e DB_PORT=5432 \
    -e DB_DATABASE="$DB_NAME" \
    -e DB_USERNAME="$DB_USER" \
    -e DB_PASSWORD="$DB_PASS" \
    -e SESSION_DRIVER=database \
    -e CACHE_STORE=database \
    -e QUEUE_CONNECTION=sync \
    "$IMAGE" >/dev/null

echo ""
echo "✔ Listo. Tu app está en: http://localhost:$APP_PORT"
echo "  - Las migraciones corren solas al arrancar."
echo "  - Revisa los logs con: docker logs -f $APP_CONTAINER"
echo "  - Para datos demo (opcional, UNA vez):"
echo "      docker exec $APP_CONTAINER php artisan db:seed --force"
echo ""
echo "  Detener:   docker stop $APP_CONTAINER $DB_CONTAINER"
echo "  Borrar:    docker rm -f $APP_CONTAINER $DB_CONTAINER && docker network rm $NETWORK"
```

Reglas del script:
- **Idempotente:** puede ejecutarse repetidas veces sin romper; la BD se reutiliza y el dato se conserva en el volumen.
- Si existe `docker compose`, la alternativa equivalente es: `docker compose up --build`.
- Añadir el nombre del script a `.dockerignore` para **no invalidar la caché de la capa `COPY . .`** con cada cambio del script.

### 20.7 Variables de entorno esenciales

Siempre inyectar al contenedor de la app:

| Variable | Valor | Por qué |
|----------|-------|---------|
| `APP_ENV` | `local`/`production` | Configura el entorno |
| `APP_DEBUG` | `true`/`false` | **Siempre `false` en producción** |
| `APP_URL` | `http://localhost:8080` | URL pública |
| `APP_KEY` | generada | Sin ella no hay sesiones/cifrado |
| `DB_CONNECTION` | `pgsql` | Driver que usa la app |
| `DB_HOST` | nombre del contenedor BD | `localhost` NO funciona entre contenedores |
| `DB_PORT` | `5432` | Puerto interno de Postgres |
| `DB_DATABASE` / `DB_USERNAME` / `DB_PASSWORD` | los de la BD | Coincidir con el contenedor de BD |
| `SESSION_DRIVER` / `CACHE_STORE` | `database` | Sin Redis ni volúmenes extra, sesiones/caché en BD |
| `QUEUE_CONNECTION` | `sync` | Procesar colas en el momento (apps simples) |

> Para **Deploy/Render** solo se inyecta `DB_URL` (la connection string del servicio) y `config/database.php` la lee con `url_parse`/`parse_url` para derivar host/port/db/user/pass; las demás variables son idénticas a la tabla.

### 20.8 Verificación post-despliegue

```bash
# 1) Endpoints críticos (deben responder 200)
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/up

# 2) Migraciones aplicadas (dentro del contenedor de la BD)
docker exec app-db psql -U postgres -d app_db -tAc \
  "SELECT 'tablas:'||count(*) FROM pg_tables WHERE schemaname='public' AND tablename<>'migrations'"

# 3) Servicios vivos dentro del contenedor
docker logs app | grep RUNNING

# 4) Errores comunes y su causa
#    - "SQLSTATE[08006] Connection refused"  → la app no encuentra la BD:
#      revisar DB_HOST (nombre de contenedor) y que la BD esté corriendo.
#    - "route cache / Closure not supported" → hay rutas con closures; convertirlas a controladores.
#    - "GIN / to_tsvector" con SQLite       → la app requiere PostgreSQL real (pdo_pgsql).
```

### 20.9 Reglas de oro del despliegue Docker

1. **NUNCA** commitear `.env` (los secretos van en variables de entorno del proveedor o del host).
2. La imagen debe contener únicamente **dependencias de producción** (`--no-dev`).
3. La base de datos SIEMPRE es un **contenedor separado**, nunca dentro del `FROM php`.
4. `DB_HOST` = nombre del contenedor de la BD, nunca `localhost`.
5. Las semillas demo se corren **una sola vez a mano** (`docker exec ... db:seed --force`), nunca en el entrypoint (lo haría no idempotente).
6. El entrypoint es **idempotente** (caches + `migrate --force` corren en cada arranque sin romper).
7. Preferir `key:generate` en el build o clave inyectada por env; nunca la misma en todos los entornos.
8. El contenedor es desechable: si algo se rompe, `docker rm -f` y levantar de nuevo; los datos sobreviven en volúmenes.

---

## 21. Buenas prácticas de código: 6 reglas de calidad

> **Regla obligatoria:** en automatización y desarrollo, **no basta con que un script funcione**; el código debe ser **mantenible, legible y escalable**. Estas 6 prácticas aplican a TODO el código del proyecto (backend, frontend, tests y scripts), igual que se aplican en entornos de banca donde la calidad es innegociable.

### 21.1 Usa un patrón de diseño (POM / Screenplay / capas)

Separa la lógica del negocio de la estructura de la UI:
- **POM (Page Object Model):** cada pantalla/componente es una clase que encapsula sus selectores y acciones. El test solo describe el "qué", no el "cómo".
- En Laravel aplica el mismo principio: **controladores delgados** que delegan en **Servicios** (ver sección 6 y 9), validación en **Form Requests** y consultas sopesadas en **scopes de modelo**.

↳ **Resultado:** menos código repetido, más fácil de mantener y escalar.

### 21.2 Escribe pruebas atómicas e independientes

Cada test **debe poder ejecutarse solo**, sin depender de otros tests ni de un orden específico de ejecución:
- Un test crea/limpia sus propios datos (usa `RefreshDatabase` + factories o seeders mínimos).
- No asumir estados que dejó otro test.
- Un fallo en un test **no** debe encadenar fallos en los demás.

↳ **Resultado:** menos errores encadenados y mayor confiabilidad. Al ejecutar los tests con total independencia, un fallo aislado se diagnostica rápido.

### 21.3 Usa aserciones claras y específicas

No te conformes con validar que algo **"está"**; válida **qué exactamente** está y **por qué**:
- Preferir aserciones concretas (`assertDatabaseHas`, `assertStatus(200)`, `assertEquals`) sobre comprobaciones vagas (`assertTrue(true)` o `assertSee` genérico sin contexto).
- Mensajes de aserción descriptivos.
- Nombrar los tests por su comportamiento esperado (`test_usuario_solo_ve_sus_propios_snippets`).

↳ **Resultado:** diagnósticos más rápidos cuando algo falla.

### 21.4 Versiona tu código y sigue una convención de commits

Un buen historial de cambios te salva de más de un apuro:
- **Commits pequeños y descriptivos**, en el idioma del proyecto (español).
- Convención clara de prefijos cuando aporte: `feat:`, `fix:`, `refactor:`, `chore:`, `docs:`, `test:`.
- Registrar cada cambio importante en `ultimosCambios.md` con versión, fecha y archivos afectados.
- NUNCA commitear secretos (`.env`, claves, tokens) — ver sección 13 y 14.

↳ **Resultado:** mayor trazabilidad y colaboración fluida; permite `git bisect` y reverts quirúrgicos.

### 21.5 Aplica el principio DRY (Don't Repeat Yourself)

No repitas lógica; extrae métodos, servicios y componentes reutilizables:
- Lógica repetida en controladores → `app/Services/`.
- Validaciones repetidas → Form Requests reutilizables.
- Consultas frecuentes repetidas → **scopes** en el modelo.
- Selectores/locators repetidos en automatización → **POM** (ver 21.1).
- CSS/blade repetido → componentes Blade y variables CSS del layout base.

> **Regla de oro:** si el mismo fragmento aparece 2+ veces, es candidato a ser extraído. La duplicación es la fuente más común de errores silenciosos.

↳ **Resultado:** código más limpio y con menos errores.

### 21.6 Integra en pipelines de CI/CD

Automatiza la ejecución de tus pruebas y revisiones en cada despliegue:
- Cada push/PR debe ejecutar al menos: **lint/typecheck** (si aplica), **tests** (PHPUnit/Pest) y **build** de la imagen Docker.
- En este proyecto el despliegue Docker está definido (sección 20); el CI/CD puede construirse con GitHub Actions: `composer install`, `php artisan test`, `docker build`.
- Mantener versionado el pipeline (`.github/workflows/*.yml`).

↳ **Resultado:** detección temprana de defectos y mayor confianza en cada entrega.