---
name: jrg300i-laravel-reglas
description: Reglas Laravel del proyecto (MVC, migraciones I-P-R-A-T, Services, Requests, seguridad, Docker, rendimiento). Usar SIEMPRE al editar PHP, Blade, migraciones, rutas o tests Laravel.
license: MIT
metadata:
  author: "Ing. Jobran Rodriguez"
  version: "1.0.0"
  github: "jrg300i"
---

# jrg300i · Laravel Reglas

> **Creado por:** Ing. Jobran Rodriguez ([@jrg300i](https://github.com/jrg300i))

Guía de construcción de proyectos Laravel: eficiente, escalable, mantenible y reutilizable. Todo cambio al proyecto debe quedar registrado en `ultimosCambios.md` si existe.

## Cuando activarse

- Editar o crear controladores, modelos, vistas Blade, migraciones, seeders, rutas, requests o tests
- Añadir features, refactorizar o revisar código PHP/Laravel
- Trabajar con Docker, seguridad, rendimiento o CI/CD del proyecto

## Instrucciones

1. Lee `references/reglas.md` (índice completo, 21 secciones).
2. Antes de editar, comprueba qué patrón ya usa el códigobase (consistencia primero).
3. Aplica la sección relevante al cambio que vas a hacer:
   - Migraciones → orden **I, P, R, A, T** + índices
   - Controladores delgados; lógica en `app/Services/`
   - Validación en Form Requests, no en el controlador
   - Vistas solo presentación; HTML semántico
   - Seguridad: sin secretos en git, autorización siempre
4. Si hay conflicto entre esta skill y una genérica, **manda `references/reglas.md`**.
5. Registra el cambio en `ultimosCambios.md` (versión, fecha, archivos).

## Estructura de referencia

| Archivo | Contenido |
|---------|-----------|
| `references/reglas.md` | Reglas completas del proyecto (fuente de verdad) |
