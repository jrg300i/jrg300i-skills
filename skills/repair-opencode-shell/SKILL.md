---
name: repair-opencode-shell
description: Repara la shell rota de opencode en Termux — bash/grep/glob colgados sin salida, "(no output)", "ripgrep execution failed". Fixes the broken opencode shell on Termux caused by LD_PRELOAD from the opencode-termux wrapper (upstream issue #4). Usar cuando la shell de opencode no responde, los comandos cuelgan indefinidamente o grep/glob falla; y también cuando opencode NO arranca tras parchear el wrapper (error al reiniciar). Autor: Ing. Jobran Rodriguez (jrg300i).
license: MIT
metadata:
  author: "Ing. Jobran Rodriguez (jrg300i)"
  github: "jrg300i"
  version: "1.1.0"
---

# repair-opencode-shell

> **Creado por:** Ing. Jobran Rodriguez ([@jrg300i](https://github.com/jrg300i))

Reparación rápida de la shell interna de opencode rota en Termux (Android),
más recuperación cuando opencode no arranca tras parchear el wrapper.

## Síntomas

- `bash`/`zsh` no devuelven nada y la tool hace timeout: `(no output)`
- `grep`/`glob` fallan con: `ripgrep execution failed`
- Comandos que cuelgan indefinidamente en opencode, pero **funcionan en la terminal del usuario**
- (Ocasional) segfaults de binarios Termux
- **Opencode no arranca tras parchear el wrapper** → ver sección
  «Si opencode NO arranca tras parchear el wrapper»

## Causa raíz

El wrapper del paquete npm **`opencode-termux`** exporta
`export LD_PRELOAD=$HOME/.opencode/ld-musl-aarch64.so.1`.
Ese loader musl, precargado en los hijos **bionic** de Termux (zsh, rg, grep),
los rompe: hang o segfault. Upstream: `github.com/C04-wq/opencode-termux/issues/4`.
`install.js` ya hace `patchelf --set-interpreter`, así que el preload sobra.

**Cómo se restaura solo:** cualquier `npm i -g opencode-termux` (o la
auto-update del wrapper) reescribe `bin/opencode` con el original de upstream.

## Dónde están las líneas que actualizan npm

### 1) Auto-update DENTRO del wrapper (se ejecuta en cada arranque)

Archivo (ambas rutas son el mismo archivo vía symlink):
`/data/data/com.termux/files/usr/lib/node_modules/opencode-termux/bin/opencode`
(` `/usr/bin/opencode` → symlink)

Es el bloque **líneas ~28-41** de la versión original de upstream:

```bash
LATEST="$(timeout 10 npm view opencode-termux version 2>/dev/null || true)"

if [[ "$LATEST" =~ ^[0-9]+\.[0-9]+\.[0-9]+([-.0-9A-Za-z.-]+)?$ ]] && [ "$CURRENT" != "$LATEST" ]; then
  printf '  ◇ Updating OpenCode…\n'
  if npm install -g "opencode-termux@${LATEST}" --ignore-scripts --silent; then
    exec "$(command -v opencode)" "$@"
  fi
  echo "Warning: update failed; using the installed version." >&2
fi

export LD_PRELOAD="$OPENCODE_DIR/ld-musl-aarch64.so.1"   # ← ESTA línea rompe los shells
export LD_LIBRARY_PATH="$OPENCODE_DIR"
```

- `npm view` = solo consulta la versión (1 log en `~/.npm/_logs/` por arranque).
- `npm install -g ...@${LATEST} --ignore-scripts` = la reinstalación que
  restaura el wrapper original (escribe log con `title npm install opencode-termux@X`).
- **Parche correcto:** reemplazar todo el bloque por `LATEST=""`, eliminar la
  línea `export LD_PRELOAD=...`, y conservar `LD_LIBRARY_PATH` + `SSL_CERT_FILE`.

### 2) Instalación manual de npm (la más común — no deja rastro en el wrapper)

Cualquiera puede correr en SU terminal:

```bash
npm i -g opencode-termux          # ← instala ÚLTIMA versión, wrapper 100% original
npm i -g opencode-termux@1.18.34-0 # ← igual, pase lo que pase
```

**Dónde conseguir las líneas / evidencia de quién actualizó npm:**
`~/.npm/_logs/` — leer los logs más recientes `*debug-0.log` y buscar:

```
verbose title npm i opencode-termux        ← instalación manual (restaura wrapper)
verbose title npm install opencode-termux@… ← auto-update del wrapper
verbose title npm view opencode-termux version ← solo consulta, inofensivo
```

Ejemplos reales:

- `2026-10-01T19_28_22_892Z-debug-0.log` → `npm i --global opencode-termux`
  instaló `1.18.34-0` y restauró el wrapper (causó la shell rota posterior).
- `2026-10-02T00_37_03_981Z-debug-0.log` → `npm i opencode-termux` corrido por el
  usuario para **recuperar un arranque fallido** (ver incidente abajo).
- `2026-10-02T00_37_17_866Z-debug-0.log` → `npm view … version`, exit 0:
  solo la consulta de auto-update del wrapper restaurado, inofensiva.

## Incidente 02/10/2026: el wrapper parcheado no dejaba arrancar

Cronología real (evidencia entre paréntesis):

- 00:35:51 — la sesión sale normal (`exiting loop` en `~/.local/share/opencode/log/opencode.log`).
- ~00:36 — **intento de reinicio fallido**: NO aparece un `creating instance` nuevo
  en `opencode.log` ⇒ el fallo fue ANTES de la aplicación (wrapper/npm), no en opencode.
- 00:37:03 — el usuario corrió `npm i -g opencode-termux`
  (log `2026-10-02T00_37_03_981Z`, `title npm i opencode-termux`) → **restauró el
  wrapper upstream** (con LD_PRELOAD y auto-update de vuelta).
- 00:37:17 — el wrapper restaurado hizo su `npm view` de auto-update (exit 0, solo consulta).
- 00:37:24 — arranque OK (run `8f4e8e8d`) y el **plugin cargó igual**
  (marker `.limpiar-env-activo` = 00:37:25) → `env | grep LD_PRELOAD` = `LD_PRELOAD=`
  vacío → **shell funcional pese al wrapper upstream**.

Lecciones:

1. **Reinstalar para recuperar el arranque es válido**: `npm i -g opencode-termux`
   devuelve un wrapper arrancable. El LD_PRELOAD que reintroduce lo neutraliza la
   Capa 2 (plugin) — el sistema de capas está diseñado para absorberlo.
2. El plugin es la red principal: sobrevive reinstalaciones y parches fallidos.
3. El stderr del intento fallido no se conservó: la causa exacta del wrapper
   parcheado no quedó registrada. Por eso **todo parche va con `bash -n` antes
   de reiniciar** (el fix canónico `~/opencode-wrapper-fix.sh` pasa `bash -n`).

## Si opencode NO arranca tras parchear el wrapper

Diagnóstico (si la shell está rota, usar `read`/`edit`, no bash):

1. `~/.local/share/opencode/log/opencode.log`: ¿aparece `creating instance` después
   del intento?
   - **No** → fallo previo a la app: wrapper o npm.
   - **Sí** → el problema es de config/plugin (revisar consola/`bannerError`).
2. Sintaxis del wrapper:
   `bash -n "$(readlink -f "$(command -v opencode)")"`
3. El binario musl debe correr **sin** LD_PRELOAD (el interpreter ya está fijado):
   ```bash
   env -u LD_PRELOAD ~/.opencode/opencode --version   # debe imprimir 1.x.x
   patchelf --print-interpreter ~/.opencode/opencode  # debe apuntar a ~/.opencode/ld-musl-aarch64.so.1
   ```

Recuperación (la que funcionó el 02/10/2026):

```bash
npm i -g opencode-termux    # restaura el wrapper upstream arrancable
```

→ reiniciar opencode → ejecutar **Verificación** (los requisitos duros bastan;
el estado del wrapper es deseable, no obligatorio).

Si además quieres volver a parchear: comparar contra `~/opencode-wrapper-fix.sh`,
aplicar con `edit`, pasar `bash -n` y solo entonces reiniciar. Nunca dejes un
wrapper instalado sin verificar.

## Reparación (paso a paso)

Importante: si la shell está rota, **usa `read`/`edit` (no bash)** para
reparar — es justo lo que no funciona.

1. **Localizar el wrapper real** (es un symlink a npm):
   `/data/data/com.termux/files/usr/lib/node_modules/opencode-termux/bin/opencode`
   (también visible en `/data/data/com.termux/files/usr/bin/opencode`).
2. **Leerlo** y buscar `export LD_PRELOAD`.
3. **Quitar con `edit`**:
   - la línea `export LD_PRELOAD="$OPENCODE_DIR/ld-musl-aarch64.so.1"` → eliminarla
   - el bloque de auto-update (`LATEST="$(timeout 10 npm view...)"` + `if [[ "$LATEST" =~ ...`) → reemplazarlo por `LATEST=""`
   - **conservar** `export LD_LIBRARY_PATH="$OPENCODE_DIR"` y `export SSL_CERT_FILE=...`
4. **`bash -n` del wrapper instalado** (si falla: no reiniciar, restaurar/corregir).
5. **Crear/verificar el plugin** `~/.config/opencode/plugins/limpiar-env.js`
   (doble red, sobrevive a reinstalaciones de npm):
   ```js
   import fs from "node:fs";
   try { delete process.env.LD_PRELOAD; } catch (e) {}
   export const LimpiarEnv = async () => ({
     "shell.env": async (input, output) => {
       if (output && output.env) output.env.LD_PRELOAD = "";
     },
   });
   ```
6. **Pedir al usuario reiniciar opencode** (el proceso actual ya tiene
   LD_PRELOAD en su env; solo un reinicio lo purga). Si no arranca →
   sección «Si opencode NO arranca tras parchear el wrapper».

## Verificación (tras el reinicio)

**Requisitos duros (todos obligatorios):**

- [ ] `echo SHELL-OK` responde sin timeout
- [ ] `env | grep LD_PRELOAD` → vacío o `LD_PRELOAD=`
- [ ] `grep` y `glob` funcionan (ripgrep: si falla, revisar `~/.cache/opencode/bin`)
- [ ] Existe el marker `~/.config/opencode/plugins/.limpiar-env-activo`
      con timestamp del arranque actual (prueba de que el plugin cargó)

**Deseable (wrapper parcheado) — si NO se cumple pero los duros sí, NO tocar:**

- [ ] El wrapper no contiene `export LD_PRELOAD`
- [ ] El wrapper tiene `LATEST=""` (auto-update desactivada)

## Prevención

- Si todo funciona con wrapper upstream + plugin (requisitos duros OK),
  **no parchear por parchear**: ese estado es estable y absorbe reinicios,
  auto-updates y reinstalaciones.
- Correr `npm i -g opencode-termux` a mano **no rompe nada mientras el plugin
  exista** (restaura el wrapper y el plugin compensa su LD_PRELOAD). Solo
  importa si se quiere conservar el wrapper parcheado.
- El plugin es la red principal: aunque el wrapper vuelva a estar roto, los
  shells funcionan porque el plugin borra LD_PRELOAD al cargar.
- Para bloquear la auto-update sin arriesgar el arranque: parchear SOLO
  `LATEST=""` con `edit`, pasar `bash -n` y reiniciar verificado
  (o `bash ~/reparar-opencode.sh`, que baja a 1.18.33-0 y reaplica el wrapper).

## Licencia y autoría

MIT © Ing. Jobran Rodriguez (jrg300i) — [github.com/jrg300i](https://github.com/jrg300i)
