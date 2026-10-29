# Caso trabajado: rustls en bpfman para Fedora

Referencia histórica del 29 de octubre de 2026. Verifica versiones, estado remoto y advisories actuales antes de reutilizar estas conclusiones.

## Identidad y resolución

- [Bug 2559998](https://bugzilla.redhat.com/show_bug.cgi?id=2559998): CVE-2026-93601, restricciones de nombres con wildcards en rustls-webpki.
- [PR #8](https://src.fedoraproject.org/rpms/bpfman/pull-request/8): rama real `mariofer:codex/fix-cve-2026-93601` hacia `rawhide`.
- Base dist-git: `1b0636d`; versión de bpfman: `0.5.4`.
- El trabajo actualizado también estaba en `fix-cve-2026-93601`, pero subir a ella no actualizaba el PR. Se añadió el commit `4694a68` sobre `cec2fe9` en la rama real del PR, conservando el historial.

El punto de partida tenía rustls-webpki 0.102.8, rustls 0.23.15 y sigstore 0.10.0. Sigstore exigía webpki `^0.102`; una actualización aislada a 0.103.12 fallaba.

La solución modificó cuatro archivos:

- `Cargo.toml`: requisito de sigstore a 0.12.1.
- `bpfman/Cargo.toml`: `cosign-native-tls` pasó a `cosign`, `native-tls`.
- `bpfman-api/Cargo.toml`: `full-native-tls` pasó a `full`, `native-tls`.
- `Cargo.lock`: resolución generada por Cargo.

El revisor pidió versiones más recientes. El resultado final usó rustls 0.23.46 y rustls-webpki 0.103.16, que incluían las siguientes correcciones; no se afirmó que todos los modos vulnerables estuvieran habilitados en bpfman:

- [CVE-2026-93601](https://github.com/rustls/webpki/security/advisories/GHSA-xgp8-3hg3-c2mh).
- [RUSTSEC-2026-0346](https://rustsec.org/advisories/RUSTSEC-2026-0346.html), [0347](https://rustsec.org/advisories/RUSTSEC-2026-0347.html), [0348](https://rustsec.org/advisories/RUSTSEC-2026-0348.html), [0349](https://rustsec.org/advisories/RUSTSEC-2026-0349.html).
- [RUSTSEC-2026-0382 / CVE-2026-111943](https://rustsec.org/advisories/RUSTSEC-2026-0382.html).

Los seis parches previos se aplicaron antes de guardar baselines. El nuevo archivo fue `0007-bump-rustls-webpki-to-0.103.16-CVE-2026-93601.patch`, registrado como `Patch6`: el nombre y el índice no tenían la misma base.

## AWS-LC, CMake y PIE

La primera solución resolvía aws-lc-sys 0.42.0. Su comprobación del compilador fallaba con opciones PIE de Fedora; forzar `AWS_LC_SYS_CMAKE_BUILDER=1` en `%build` y `%check` permitió compilar.

El bump posterior resolvió aws-lc-sys 0.47.0. Su `builder/cc_builder.rs`, función `memcmp_check`, conservaba flags calculados por cc, respetaba LDFLAGS, restringía la prueba a versiones potencialmente afectadas de GCC y convertía fallos ambientales en avisos. Se retiró el override y se validó con Mock. Es una decisión de este caso, no una regla universal sobre esa variable.

## Vendor, licencias y validación

Se generó el vendor con `cargo vendor --locked --versioned-dirs`, se aplicó la limpieza de Fedora y se subió el tarball a lookaside conservando `v0.5.4.tar.gz` en sources. El vendor contenía 534 crates.

Se añadieron las expresiones de licencia necesarias para ese conjunto: `Unicode-3.0`, `(Apache-2.0 OR ISC)` y `(Apache-2.0 OR ISC OR MIT-0)`. Deben revisarse de nuevo cuando cambien las dependencias.

El [PR #3](https://src.fedoraproject.org/rpms/bpfman/pull-request/3#request_diff) tenía un URL comentado a `marioferh/ebpf_sources`; Source1 activo seguía siendo el nombre del tarball obtenido de lookaside.

Mock estaba ocupado por un build de Mario. Se compararon los digests de los archivos del SRPM y su spec con los insumos nuevos, sin interrumpirlo. Ese build terminó correctamente sin el override de CMake: compilación, 10 tests e instalación. Los seis tests `image_pull_*` estaban excluidos porque necesitaban red. Los logs contenían ejecuciones anteriores y se verificó la más reciente.

## Lección de rpmlint

El [job de Testing Farm](https://artifacts.dev.testing-farm.io/722aee5b-d2cf-4f1b-bb01-5a2d691597f8/) evaluó `4694a68` y falló por cuatro errores `incorrect-fsf-address` en `LICENSE-GPL2`, uno por arquitectura. Los otros 25 mensajes eran advertencias.

La dirección de Franklin Street también se considera obsoleta en rpmlint 2.8.0. La [copia oficial de GPLv2](https://www.gnu.org/licenses/old-licenses/gpl-2.0.txt) usa enlaces. Se preparó un parche separado para que el archivo coincidiera byte por byte con esa copia. Esta comprobación es distinta de compilar y ejecutar los tests del programa; no hay que ocultar el error con un filtro ni regenerar vendor para cambiar este archivo de licencia de bpfman.

## Bugs descartados durante la revisión

- [Bug 2560764 / CVE-2026-81856](https://bugzilla.redhat.com/show_bug.cgi?id=2560764): el [advisory](https://github.com/hexpm/hex/security/advisories/GHSA-fpmj-3cxg-p766) afectaba al gestor Hex de Elixir/Erlang, no al crate Rust hex 0.4.3. Se propuso NOTABUG por diferencia de componente, sin afirmar confirmación oficial de Red Hat.
- [Bug 2543695 / CVE-2026-84782](https://bugzilla.redhat.com/show_bug.cgi?id=2543695): la revisión encontró `SslMethod::tls()` en native-tls y no encontró uso de DTLS; bpfman enlazaba OpenSSL del sistema. El [aviso de Red Hat](https://access.redhat.com/security/cve/CVE-2026-84782) excluía TLS sobre TCP. Se propuso NOTABUG con esa evidencia, sin afirmar que todo OpenSSL de Fedora estuviera corregido.

Los comentarios se prepararon para revisión; no se publicaron ni se cerraron bugs automáticamente.
