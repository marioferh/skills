---
name: bpfman-fedora-bump
description: Prepara bumps de dependencias Rust y correcciones de CVE del paquete bpfman en Fedora dist-git. Úsalo para revisar un bug de bpfman, generar su parche y vendor, validar con fedpkg, Mock y rpmlint, o actualizar la rama de un PR de Fedora existente.
---

# Bumps de bpfman en Fedora

Aplica este flujo al estado real del repositorio. Habla con Mario en español; redacta commits, descripciones de PR y comentarios para mantenedores en inglés.

## Alcance y referencias

- Si Mario sigue la guía paso a paso, ejecuta el paso pedido y sus prerrequisitos necesarios. Una pregunta sobre un comando pide una explicación, no ejecutarlo. Si solicita el flujo completo, continúa dentro del alcance autorizado.
- El skill no concede autorización para subidas futuras. Usa las autorizaciones de la conversación para lookaside, commit/push y comentarios; no solicites de nuevo una autorización existente. Preparar un comentario no implica publicarlo ni cerrar el bug.
- Respeta el flujo personal de PR: prepara la rama y el formulario para que Mario revise. No envíes el formulario ni crees un PR, tampoco un borrador por API/CLI, salvo que pida explícitamente cambiar esa preferencia. Una petición de actualizar un PR existente sí autoriza subir los cambios correspondientes a su rama.
- Cuando se pida abrir el formulario, prefiere Firefox externo si está disponible, con título y descripción preparados cuando la plataforma lo permita. Usa las rutas reales de Pagure; no traslades rutas de GitHub a Fedora.

Consulta la sección pertinente de la guía y explica las adaptaciones necesarias:

- [Inicio y entorno](https://danielmellado.github.io/bpfman_package_web/docs/getting-started/)
- [Parches, vendor y sources](https://danielmellado.github.io/bpfman_package_web/docs/patching/)
- [Compilación](https://danielmellado.github.io/bpfman_package_web/docs/building/)
- [Pull requests](https://danielmellado.github.io/bpfman_package_web/docs/pull-requests/)

Para retomar el caso de rustls, investigar CMake/PIE o entender los descartes de CVE, lee [el caso trabajado](references/caso-rustls-2026.md). Sus versiones son históricas, no objetivos de futuros bumps.

## 1. Identificar el bug y el estado local

Lee el Bugzilla, el aviso de Red Hat y el advisory upstream/RustSec. Confirma componente, ecosistema, versiones afectadas/corregidas y condiciones relevantes. Un nombre coincidente no prueba que sea la misma biblioteca. Distingue bibliotecas nativas de bindings Rust y comprueba qué enlaza el paquete.

Si la evidencia indica que bpfman no está afectado, explica el motivo y prepara un comentario cuando se solicite. No hagas un bump innecesario ni presentes una propuesta de NOTABUG como decisión oficial.

Inspecciona `AGENTS.md`, estado Git, rama, remotos, spec, sources y parches. Obtén `Version`, `Source0`, `Source1`, `PatchN`, `%prep`, licencias y tests del spec vigente. La versión de bpfman, las versiones de sus crates y `Release` son datos distintos.

La configuración de Mario usa `origin` para `https://src.fedoraproject.org/forks/mariofer/rpms/bpfman.git` y `upstream` para `https://src.fedoraproject.org/rpms/bpfman.git`. Verifica antes de usarla; conserva cambios ajenos y no fuerces SSH si HTTPS ya funciona. No presupongas que un directorio llamado `bpfman-upstream` es un clon: puede contener fuentes extraídas sin `.git`.

Para una rama nueva, parte de `upstream/rawhide` actualizado y usa el nombre solicitado o el prefijo configurado. Para un PR existente, lee su rama real de origen y destino: una rama de nombre parecido no lo actualiza.

## 2. Construir la baseline

Usa fuentes verificadas que correspondan exactamente a `Version`/`Source0`: el tag correspondiente en un clon limpio, o el tarball con checksum verificado en un directorio separado. Rawhide es la base de empaquetado; el tag identifica las fuentes.

Aplica todos los parches activos en el orden efectivo de `%prep`. Para el caso habitual de `%autosetup -p1`, usa cada archivo concreto; evita globs que mezclen versiones. Investiga rechazos o fuzz inesperado.

Guarda `Cargo.lock.baseline` después de los parches previos y antes del bump. Conserva también una baseline de cada manifest que debas cambiar, manteniendo su ruta relativa. No sobrescribas estas baselines al reintentar.

## 3. Resolver el bump con Cargo

Elige versiones corregidas y compatibles según advisories actuales y peticiones del revisor. Ejecuta actualizaciones dirigidas con `cargo update -p CRATE --precise VERSION`, sustituyendo los parámetros por los valores verificados.

Si otro crate impone un rango incompatible, identifica la cadena con metadata/tree y modifica únicamente los manifests y features necesarios. No edites manualmente versiones/checksums del lock ni hagas un `cargo update` global por comodidad.

Revisa el diff y ejecuta `cargo metadata --locked` o `cargo tree --locked`. Busca todas las versiones del crate afectado: actualizar una no elimina otra vulnerable. Distingue lo incluido en lock/vendor de lo habilitado para el target y features de Fedora.

## 4. Generar y comprobar el parche

Incluye el diff de todos los archivos necesarios respecto a sus baselines: lock y manifests modificados. Mantén la convención de nombres y los identificadores correctos del bug/CVE. Al revisar el mismo bump, reemplaza su parche anterior y conserva una sola entrada activa en el spec.

Para cada archivo usa rutas aplicables mediante `patch -p1`, por ejemplo:

```bash
diff -u --label c/Cargo.lock --label w/Cargo.lock Cargo.lock.baseline Cargo.lock
```

Concatena los diffs de manifests con sus rutas relativas. `diff` devuelve 1 cuando encuentra diferencias, lo cual es esperado; un código mayor indica error. No borres los espacios que representan líneas vacías de contexto de unified diff aunque Git los señale como whitespace dentro del archivo `.patch`.

Verifica en fuentes limpias: aplica todos los parches previos y después el nuevo, y compara los resultados con los archivos generados por Cargo. Si se pide restaurar/reaplicar en el árbol de trabajo, respalda los cambios y restaura todos los archivos afectados, no solo el lock. En una extracción usa el tarball original, no `git checkout`.

## 5. Regenerar vendor y el tarball

Reconstruye el directorio generado, conservando antes cualquier modificación manual necesaria:

```bash
cargo vendor --locked --versioned-dirs
```

Aplica las limpiezas exigidas por la guía y el spec vigente. Para fiat-crypto 0.2.9, el flujo trabajado elimina `src/p434_64.rs` y su declaración de módulo en `src/lib.rs`; comprueba atributos asociados y rutas si cambia la versión. Revisa la limpieza requerida de caracteres de control en los datos de prueba de idna.

Respeta los ajustes de `%prep` para checksums de vendor, licencias y permisos de archivos `.rs`. No desactives verificaciones globales de Cargo ni edites los checksums manualmente sin entender el flujo del spec.

Valida usando el vendor sin red:

```bash
cargo metadata --locked --offline --format-version 1 \
  --config 'source.crates-io.replace-with="vendored-sources"' \
  --config 'source.vendored-sources.directory="vendor"'
```

Conserva el `.cargo/config.toml` existente. Empaqueta desde el directorio padre, con `vendor/` como raíz y el nombre exacto esperado por Source1. Comprueba que contiene las versiones resueltas en el lock.

## 6. Actualizar dist-git y lookaside

Actualiza el SHA512 del vendor en `sources`, preservando las fuentes originales. Añade/actualiza el PatchN y revisa las necesidades reales de licencias. Respeta `%autorelease` y `%autochangelog`; un bump de crates no cambia por sí mismo Version de bpfman.

`fedpkg new-sources` reemplaza la lista de fuentes. Consulta la ayuda instalada y conserva todos los archivos requeridos al subir; en el caso de bpfman 0.5.4 se usó:

```bash
fedpkg --release rawhide new-sources bpfman-0.5.4-vendor.tar.xz v0.5.4.tar.gz
sha512sum --check sources
```

Resuelve los nombres desde el spec/sources real. Sube cuando esté autorizado y confirma el resultado: un hash local no prueba que el tarball exista en lookaside. Verifica `.gitignore`; los tarballs no deben entrar en el commit.

Un URL comentado a un vendor en GitHub no es un Source1 activo. Con el archivo en lookaside no hace falta subir otra copia a GitHub salvo petición o necesidad explícita del proyecto.

## 7. Validar build, tests y CI

Comprueba el spec con `rpmspec --parse`, los checksums y el diff de empaquetado. En una rama de trabajo usa explícitamente:

```bash
fedpkg --release rawhide mockbuild -- --postinstall
```

No heredes excepciones de CMake sin comprobar su necesidad en las versiones actuales. Respeta las exclusiones justificadas por el spec y no cuentes tests excluidos como ejecutados.

Si Mock está bloqueado, identifica el proceso y no interrumpas el build del usuario. Puedes aprovechar uno en curso si verificas que su SRPM contiene las mismas fuentes, parches, vendor y configuración de build; explica diferencias de NVR/metadatos. Un build de versiones anteriores no valida el bump actual.

Los logs de `results_bpfman` pueden acumular ejecuciones: identifica la última y confirma por separado compilación, tests e instalación de `--postinstall`. Valida también con rpmlint; Mock exitoso no garantiza que los controles de empaquetado pasen.

Para fallos de Testing Farm, lee `results.xml`, identifica el test fallido y sigue sus enlaces a logs. Confirma el commit evaluado, diferencia errores de advertencias y compara con el estado anterior. Corrige la causa dentro del alcance, sin ocultarla mediante filtros. En errores de licencias, consulta el texto oficial antes de proponer cambios. No repitas sin cambios un fallo de autenticación o infraestructura.

## 8. Commit, push y revisión

Añade explícitamente spec, sources, parche nuevo, eliminación del parche reemplazado y `.gitignore` si cambió. Excluye baselines, caches, vendor extraído, logs y RPMs. Describe el resultado final, con versiones y referencia al bug correctas.

Con autorización, sube al fork y a la rama verificados. Para actualizar un PR, añade un commit sobre su rama remota y conserva el historial; no uses force-push por defecto. Si el resultado está preparado en otra rama, compara ambos árboles y traslada solo los cambios esperados.

Confirma el SHA remoto y que el PR muestra el nuevo commit. El push no cambia automáticamente título/descripción: revisa su coherencia y comunica lo pendiente si no puedes corregirlo dentro del acceso disponible. No crees otro PR para actualizar el existente.

Termina con cambios, rama/commit, estado real de lookaside, enlace de revisión, validación obtenida y limitaciones materiales pendientes.
