# xcpe-mecool-km2plus
Análisis del backdoor XCPE en dos MECOOL KM2 Plus: qué hace, cómo se instala y como sacarlo

# Backdoor XCPE en dos TV box Android MECOOL KM2 Plus: notas personales

Notas de cuando desarmé el backdoor de fábrica de dos MECOOL KM2 Plus que tengo, después de las noticias de noviembre de 2026 sobre la botnet XCPE / BadBox. Todo lo hice en mis propios equipos, con ADB, PowerShell y un PC con Linux. Análisis estático más algunas revisiones en vivo en los tvbox. No ejecuté el malware en un sandbox, y cuando lo revisé los servidores de control ya los habian botado.

English version (short advisory): [ADVISORY.md](ADVISORY.md)

Si tienes una de estas tvbox y solo quieres saber qué hacer, salta mis hermosas notas personales y ve a [Revisa tu tvbox](#revisa-tu-tvbox) y [Lo que hice](#lo-que-hice).


## Resumen

- Las dos tvbox traen `com.oemservice` (nombre interno `x.cpe`), que corre como **usuario de sistema** y está firmado con la **llave de plataforma** del firmware.
- Se instala en tres etapas: un mini instalador en `/system/priv-app`, un APK completo guardado en la partición `/oem` y una copia autoactualizada en `/data/app`.
- Una de mis tvbox (la de uso diario desde 2024) tenía el backdoor **activo**, en la versión 6.5.0, con su cliente MQTT corriendo.
- Lo que puede hacer (según el código): ejecutar scripts que le manda el operador, instalar y desinstalar apps, sacar los logs del sistema, bloquear el equipo, **resetearlo de fábrica a distancia**, funcionar como relay de proxy, meter publicidad y hacer pruebas de ping, traceroute y ancho de banda.
- Los dominios de control (C2) de mis muestras dan **NXDOMAIN** al 6 de noviembre de 2026, lo que calza con el desmantelamiento reportado el 2 de noviembre. Pero el código sigue en el firmware.
- `pm uninstall --user 0 com.oemservice` aguantó dos reinicios en mis pruebas. Un reseteo de fábrica muy probablemente lo trae de vuelta.
- La muestra 6.5.0 está en VirusTotal desde el **31-12-2023**, y alguien armó un grafo en VirusTotal llamado "MeCool KM2 Plus Deluxe" en **diciembre de 2024**.
- **Actualización 10-11-2026:** la tvbox A recibió en silencio una OTA a **C1.3.3_20260611** entre el 6 y el 8 de noviembre de 2026, o sea **después** del desmantelamiento y del comunicado de MECOOL. La OTA reescribió `/oem` completa y volvió a meter **el mismo XCPE 6.9.2, byte por byte** (mismo SHA256), y el mismo mini instalador en `/system`. Ver [La OTA de noviembre](#la-ota-de-noviembre-2026).

## Equipos

| | tvbox A | tvbox B |
|---|---|---|
| Modelo | MECOOL KM2 Plus | MECOOL KM2 Plus |
| `ro.product.name` / device | `HP4414-MECOOL` / `YYC` | `HP4414-MECOOL` / `YYC` |
| `ro.product.manufacturer` | `SkyworthDigital` | `SkyworthDigital` |
| Android | 14 | 14 |
| Firmware (`oem.buildnumber`) | `C1.3.1_20260314` → `C1.3.3_20260615` (OTA, nov. 2026) | `C1.2.7_20250822` |
| Estado al encontrarla | solo el mini instalador | **6.5.0 activo, MQTT corriendo** |

El tvbox B es el más antiguo, de uso diario normal, y nunca lo había tocado antes de esto. (El `ro.build.fingerprint` de la C1.3.3 dice `C1.3.3_20260611`; el `oem.buildnumber` dice `20260615`, la fecha de la partición `/oem`.)

## Contexto

Los reportes publicos (ver [Fuentes](#fuentes)) describen XCPE como un backdoor de firmware atribuido al fabricante ODM Videostrong, usado para convertir decodificadores en proxies residenciales y para fraude publicitario, con una botnet de alrededor de 1,3 millones de equipos. Google y otras empresas anunciaron su desmantelamiento a comienzos de noviembre de 2026. Security Affairs menciona los paquetes `x.cpe` / `com.oemservice` y `com.abc.nexus`, el dominio `szbboys.com` y la reinstalación desde la partición `/oem/`. Lo que encontré calza con eso.

## Secuencia de instalación

```
/system/priv-app/app_xcpe_preset/   (mini instalador, v1.2.1, ~21 KB, sin código de red)
        │  BootReceiver → UpgradeService (servicio en primer plano con una
        │  notificación falsa: "You've received new messages")
        ▼
/oem/app_xcpe_ext_sign               (APK completo, instalado en silencio con
        │                              PackageInstaller como "actualizacion" de si mismo)
        ▼
/data/app/.../com.oemservice-.../base.apk   (versión autoactualizada)
```

En la tvbox B, `dumpsys package com.oemservice` mostró `installerPackageName=com.oemservice`: se instaló a si mismo.

La app de pruebas de fabrica del mismo firmware (`com.oem.qa`, "BoxFactoryTest") usa exactamente las mismas tres capas (`BoxFactoryTest_preset` en `/system/priv-app` → `BoxFactoryTest_sign` en `/oem` → actualizacion en `/data/app`). O sea, XCPE se monta sobre el mecanismo normal con que el fabricante despliega sus apps.

## Versiones

| Dónde | Versión (versionCode) | Primera vez en VirusTotal |
|---|---|---|
| Mini instalador, `/system/priv-app` (archivo idéntico en ambas tvbox) | 1.2.1 (2) | – |
| Corriendo en la tvbox B, `/data/app` | 6.5.0.20230804 (43) | 31-12-2023 |
| `/oem` de la tvbox B (firmware de ago. 2025) | 6.8.4 (59) | 18-05-2026 |
| `/oem` de la tvbox A (firmware de mar. 2026) | 6.9.2 (65) | 04-11-2026 |
| `/oem` de la tvbox A tras la OTA C1.3.3 (firmware de jun. 2026, instalado en nov. 2026) | 6.9.2 (65), **mismo archivo** | – |

Algo raro: la tvbox B corría la 6.5.0 aunque su propio `/oem` traia la 6.8.4. Su `lastUpdateTime` era 25-07-2024, anterior a la fecha del firmware actual. Yo creo que la 6.5.0 se instaló en 2024 con un firmware anterior, una OTA de despues dejo la 6.8.4 en `/oem`, y el mini instalador nunca reemplazó la copia que ya estaba en `/data`. El mismo par (XCPE 6.5.0 + BoxFactoryTest 6.2.7) se subió a VirusTotal con 13 segundos de diferencia el 31-12-2023, así que esa combinación ya existía en la tvbox de otra persona en esa fecha.

### Cómo cambió entre versiones

| | 6.5.0 | 6.8.4 | 6.9.2 |
|---|---|---|---|
| `AdvertiseService` (publicidad integrada) | si | no | no |
| Referencias a `com.abc.nexus` / comandos `ext/lib/*` | no | si | (no revisado) |
| `RoomProvider` / `ShareProvider` | no | si | si |

Mi lectura: al principio la publicidad venía integrada; después la sacaron a un loader aparte (`com.abc.nexus`) que descarga los modulos de monetización. Ninguna de mis tvbox tenia `com.abc.nexus` instalado.

## La OTA de noviembre 2026

El 6-11-2026 la tvbox A estaba en C1.3.1 (lo tengo anotado). El 8-11 ya estaba en C1.3.3. Se instaló sola, en silencio, al reiniciar (es un equipo A/B, no aparece la pantalla de actualización). `sys.boot.reason` quedó en `reboot,ota`.

Comparé la tvbox A antes y después, y la tvbox B que sigue en C1.2.7:

| | tvbox B (C1.2.7) | tvbox A (C1.3.1, 6-11) | tvbox A (C1.3.3, 10-11) |
|---|---|---|---|
| `/oem/app_xcpe_ext_sign` | `c9baee08…` (6.8.4) | `aa6269b9…` (6.9.2) | `aa6269b9…` (6.9.2, **idéntico**) |
| Mini instalador `/system/priv-app/app_xcpe_preset` | `88e1b6ec…` | `88e1b6ec…` | `88e1b6ec…` (**idéntico**) |
| `/oem/BoxFactoryTest_sign` | `675226b8…` | `6e95ce13…` | `6e95ce13…` |
| Fecha de los archivos en `/oem` | – | 2026-03-14 | **2026-06-15** (partición reescrita) |
| `/oem/app/SdtDeviceInfo` | sí | sí | no |
| `/system/app/AxTools` | no | ? | sí |
| `com.sdt.upgrade` | no existe | existe | existe |

O sea:

- `oem` está en `ro.product.ab_ota_partitions`. La OTA no "dejó olvidado" el backdoor: **escribió la partición `/oem` de nuevo y metió el mismo XCPE**.
- El firmware se compiló en junio, antes del desmantelamiento, pero se **entregó e instaló en noviembre**, después del desmantelamiento (2-11) y del correo de MECOOL diciendo que estaban revisando el firmware (4-11).
- Yo tenía `com.sdt.ota` desinstalado (`--user 0`) desde el 6-11, así que la OTA entró por otro lado. Lo más probable es `com.sdt.upgrade` (`SdtUpgradeService`), que no existe en la C1.2.7 y aparece en la C1.3.x.
- **Lo bueno:** los `pm uninstall --user 0` sobrevivieron la OTA. `com.oemservice`, `com.oem.qa`, `backupappmanger` y `com.sdt.ota` siguieron desinstalados.
- Con `com.sdt.ota` reactivado, ninguna de las dos tvbox ofreció más actualizaciones (10-11-2026). La tvbox B sigue en C1.2.7.

Ojo: el `firstInstallTime` de las apps de sistema nuevas sale 2008-12-31, y el timestamp de `reboot,ota` en `persist.sys.boot.reason.history` coincide con la fecha del build. Es el reloj del equipo que arranca en la fecha del firmware antes de sincronizar, no la fecha real de instalación.

## Lo que "puede" hacer

Sacado de strings del DEX y de los manifiestos. Son capacidades que están en el código, no cosas que lo haya visto hacer.

**Comandos remotos (tópicos MQTT, en la 6.5.0 y en la 6.8.4):**

| Área | Tópicos |
|---|---|
| Control del equipo | `function/reboot`, `function/lock`, `function/factory/reset`, `function/alarm`, `function/clear`, `function/remote/control`, `function/ota`, `xcpe/upgrade` |
| Ejecucion de código | `function/script` (el resultado va a `script_result.txt`), `function/cmd/read` |
| Apps | `function/app/install`, `app/uninstall`, `app/enable`, `app/clear`, `apps/read`, `apps/report`, `apps/usage/report` |
| Logs | `function/log/logcat`, `log/dumpsys`, `log/kernel`, `log/crash`, `log/register`, `log/reserved` |
| Información | `properties/read`, `properties/report`, `network/read`, `state/check`, `inform/report` |
| Tareas | `task/boot/pull`, `task/query`, `task/clear` |
| Diagnostico | `function/diagnosis` (ping, traceroute, pruebas de subida y bajada) |
| Configuración | 6.5.0: `/cpe/config/acs`, `/cpe/config/mqtt`; 6.8.4: `/router/config/acs`, `/router/config/app`, `/router/config/mqtt` |

Solo en la 6.8.4: `function/power/off`, `function/notify`, `init/boot/report`, `television/read`, `remote/control/read`, `room/read`, `room/config/read`, `share/config/read`, `cast/online/report`, `ext/lib/config`, `ext/lib/status/report`, `ext/lib/upgrade`.

**Otras cosas en el código:**
- Relay de proxy en la 6.5.0 (`prxserver`, autenticacion `chkkey`, "payload to upstream", keepalive con `persist.sys.cpe.executor.keepalive.time`).
- Su propio cliente DNS, que lee `net.dns1`–`net.dns4`.
- Publicidad en la pantalla de inicio (imagen, mensaje, video) y reemplazo de la animación de arranque (6.5.0).
- Consulta de IP y ubicación con `ip-api.com`, y subida de la lista de apps instaladas y su uso.
- Un servicio de accesibilidad (`AccessibilityKeyDetector`, `enableXcpeAccessService`) que observa las teclas del control remoto.
- Capturas de pantalla, `settings put`, instalación silenciosa de APK (incluso divididos), manipulacion del verificador de paquetes, lectura de EDID, MAC, número de serie, chip ID y contraseña del hotspot (versiones de `/oem`).
- Campos de configuración `acsPassword` / `cpePassword`. Los nombres (ACS, CPE, más diagnósticos de ping, traceroute, bajada y subida) copian TR-069 / TR-143, los protocolos que usan los proveedores de internet para administrar routers a distancia.
- No hay IPs de respaldo escritas en el código ni secretos obvios en el DEX.

**Permisos concedidos en la tvbox B** (`dumpsys`, `granted=true`): `MASTER_CLEAR`, `REBOOT`, `INJECT_EVENTS`, `READ_FRAME_BUFFER`, `WRITE_SECURE_SETTINGS`, `DELETE_PACKAGES`, `INSTALL_PACKAGE_UPDATES`, `MANAGE_DEBUGGING`, `MANAGE_NOTIFICATION_LISTENERS`, `CONFIRM_FULL_BACKUP`, `BLUETOOTH_PRIVILEGED`, `READ_OEM_UNLOCK_STATE`. Parte de eso viene de compartir el UID 1000 con el resto de las apps de sistema, pero el punto se mantiene: los tiene todos. Los manifiestos además declaran `LOCK_DEVICE`, `SHUTDOWN`, `SET_TIME`, `GET_ACCOUNTS_PRIVILEGED`, `MANAGE_ACCOUNTS`, `USE_CREDENTIALS`, `com.abc.nexus.START_PERMISSION` (6.8.4) y más.

### Las "librerias nativas"

VirusTotal avisa que "contiene ejecutables de Linux". Hay dos archivos `.so` (`lib/armeabi-v7a/libtraceroute.so` y `assets/libtraceroute.so`), pero son lo mismo: un ejecutable ARM enlazado estáticamente de **Modern traceroute for Linux 2.0.18** (Dmitry Butskoy, GPL), renombrado `lib*.so` para que Android lo extraiga con permiso de ejecución. La parte en Java lo copia, le hace `chmod 777` y lo ejecuta para los diagnósticos remotos. La IP `198.108.0.18` que trae adentro es el servidor whois de RADb que traceroute usa por defecto, no un C2. **No hay lógica de proxy ni de C2 en el código nativo.**

`assets/register_cmd.txt` es una lista de comandos de depuración de HDMI y video de Amlogic (`cat /sys/class/amhdmitx/...`), con comentarios en chino, que usa el comando `log/register`.

## Infraestructura C2

| Dominio | Dónde aparece | DNS al 06-11-2026 (local, 8.8.8.8, 1.1.1.1) |
|---|---|---|
| `extott.szbboys.com` | 6.5.0 | NXDOMAIN |
| `bmw.bestipip.com:16000` | 6.5.0 | NXDOMAIN |
| `extstb.anyevonline.com` | versiones de `/oem` | NXDOMAIN |
| `szbboys.com`, `bestipip.com`, `anyevonline.com` (raíz) | | NXDOMAIN |
| `intermedia-arts.tv` | listas de IoC, no en mis muestras | NXDOMAIN |
| `yokatvonline.com` | listas de IoC, no en mis muestras | sigue registrado (DNS de Alibaba Cloud), sin registro A |
| `mqtt.szbboys.com` (puertos 3885–3887), `ott.szbboys.com:8000` | informe de Google | – |
| `mcstb.anyevonline.com:8000`, `stb<versión><tipo>.anyevonline.com:8000` | informe de Google | – |

Lo que dice el informe de Google (GTIG, 02-11-2026) y calza con mis muestras:

- El arranque hace un POST a `http://mcstb.anyevonline.com:8000` **si `ro.mecool.device` es true**; si no, arma `http://stb<versiónAndroid><tipo>.anyevonline.com:8000`. En mis dos tvbox `ro.mecool.device` está **vacío**, así que irían por la segunda vía, y eso calza con el `extstb.anyevonline.com` que encontré en las versiones de `/oem`.
- Endpoints `/router/config/mqtt`, `/router/config/acs` y `/router/config/app`: los mismos que yo vi en la 6.8.4.
- El broker MQTT `mqtt.szbboys.com` escucha en los puertos **no estándar 3885–3887** (IPs `219.134.163.210` y `.211`).
- Otros servidores del operador: `38.145.197.135` (hospeda `anyevonline.com` y archivos) y `23.94.49.57` (archivos y C2).
- `ShareProvider` (`com.oem.share.provider`), `libtraceroute.so`, desactivar Play Protect con `package_verifier_user_consent = -1`: todo esto también aparece en mis muestras.

**Ninguno de mis hashes** (las tres versiones de XCPE, el mini instalador, BoxFactoryTest y los `libtraceroute.so`) está en la lista de IoCs de Google, así que son muestras adicionales.

Google dice que si el usuario desinstala x.cpe, "se reinstala automáticamente desde `/oem/` al reiniciar". En mis pruebas, `pm uninstall --user 0` del stub **no** se revirtió con reinicios ni con una OTA completa. Creo que Google se refiere a borrar la copia de `/data/app` (las actualizaciones), no a ocultar el paquete del sistema para el usuario 0.

Una captura de 15 minutos de la tvbox B (ARP spoof) antes de limpiarla no mostro MQTT, ni tráfico al puerto 16000, ni consultas DNS a estos dominios. Calza con que los dominios ya estaban muertos. La captura no ve TLS en puertos no estándar ni DNS cifrado.

## Firma

Todas las muestras de XCPE (mini instalador, `/oem` y versión viva) están firmadas con el mismo certificado:

```
CN=skydigital, OU=SKYWORTH, O=SKYWORTH, L=ShenZhen, ST=GuangDong, C=CN, E=skydigital@skyworth.com
SHA1 1C:6A:0B:B5:70:10:A7:7C:BF:29:A4:95:FF:55:F7:36:2F:E3:32:A3
Serie 3a4085b2e52b00573f11f380eac3da2664f6cd75, válido del 09-03-2022 al 25-07-2049
```

Esa es la **llave de plataforma** del firmware: también firma el paquete `android` y SystemUI. Prueba que el backdoor entró en el build oficial del firmware. Por sí sola no dice quién lo puso (el fabricante ODM, un subcontratista o una llave filtrada). La atribución a Videostrong viene de los reportes de Google, no de la firma.

Un vínculo a nivel de código: XCPE tiene `com.oemservice.util.InstallForAndroiP$InstallResultReceiver` escuchando `com.vs.setupcustomizer.ACTION_INSTALL_RESULT`. La app de pruebas de fabrica tiene `com.oem.commonutilits.InstallForAndroiP` (con el mismo error de tipeo) y la misma acción. Y esa app de fábrica menciona a Videostrong explícitamente (carpeta `/VideoStrong` en el USB, `uploadToVsServer`, `readFromVsServerByChipId`). Comparten código interno.

## Otros paquetes que revise

| Paquete | Veredicto |
|---|---|
| `com.sdt.ota` (SdtUpdate) | Cliente OTA de Skyworth (`/ota/system/*`, `/ota/software/*`), sin referencias a XCPE |
| `com.oem.qa` (BoxFactoryTest 6.2.7) | Herramienta de pruebas de fábrica de Videostrong: burn-in, escritura de MAC y número de serie, aprovisionamiento de llaves DRM, ejecuta `/config/shell_command.sh` e instala APKs desde un USB preparado, y expone un intent de instalación silenciosa (`com.vs.action.ACTION_INSTALL_APK_SELICENT`). No es XCPE, pero igual la saqué |
| `com.sdt.deviceinfo` | Pantalla de información del equipo, sin código de red. Sandbox de VirusTotal: "non malicious" |
| `com.sdt.producttest` | Pruebas y aprovisionamiento de fábrica |
| `com.sdt.customization` | Nada interesante |
| `com.sdt.backupappmanger` | Una detección de AhnLab con el mismo nombre que le pone a XCPE (`Downloader/Android.Agent.1228987`). No está claro si es código compartido o una firma amplia. La saqué |
| `com.sdt.remotecontrolota`, `globalkey`, `mecoolrcupair`, `frontpanelledsservice` | Se ven bien |
| `com.sdt.axtools` (AxTools 1.0, `/system/app`, aparece en firmware posterior a C1.2.7) | Lo descompilé entero: 4 clases propias, cero código de red. Al arrancar, si `persist.sys.rectify.attkey=1`, borra los datos de `com.google.android.tv.axel` y deja la propiedad en 0. En mi equipo está en 0, así que no hace nada. Corre como UID 1000 (por eso la lista de permisos se ve terrible). VirusTotal 2/67, con la **misma** detección de AhnLab (`Downloader/Android.Agent.1228987`) que backupappmanger y XCPE. Benigno. SHA256 `d7eb4282373a191fce63c5d1e893f47c76d795646249f4b044c773032f2373cc` |
| `com.sdt.upgrade` (`SdtUpgradeService`, aparece en C1.3.x) | UID de sistema, sin permisos propios. Probablemente por donde entró la OTA C1.3.3. Lo deshabilité |

Sobre AhnLab: a estas alturas `Downloader/Android.Agent.1228987` le cae a tres apps distintas firmadas con la misma llave de plataforma (XCPE, backupappmanger y AxTools, que es inofensivo). Parece una firma amplia, no una prueba por sí sola.

Reiniciar la tvbox B con `com.sdt.ota` y `backupappmanger` todavía activos **no** trajo de vuelta a XCPE, así que ninguno de los dos lo reinstala en un reinicio normal.

## VirusTotal

| Muestra | SHA256 | Detecciones |
|---|---|---|
| XCPE 6.5.0 (activa, tvbox B) | `2a443baa14be4a2a45b7ea477a8ffc326169aaecc89919a42cf386d0978db0f2` | 26/68 |
| XCPE 6.8.4 (`/oem`, tvbox B) | `c9baee088d95f24ea9945ba6f0f9f9d717680aaa4d347a42e912785338ac539f` | 25/67 |
| XCPE 6.9.2 (`/oem`, tvbox A) | `aa6269b993a6ca4f4286ed20cd6b3d7ae7ac4edc329d10ed525c602b60c4fbe5` | 18/68 |
| Mini instalador 1.2.1 (ambas) | `88e1b6ec984534def6e05421d973e7bf45fe602d929bd3edbb5aaa7e0a3edb7f` | 3/67 |
| `com.oem.qa` 6.2.7 | `af0883f04c2c1d39219460320031c846ab72f8e0d95060fc3e919ddcc7f133f2` | 2/66 |
| `com.sdt.deviceinfo` 1.0.4 | `7fc493cc711cd36178e40c4d5e28a5662d9641ca33e170c925537fcb1b14e3d3` | 2/65 |
| `com.sdt.ota` 2.5.1 | `179575c5b96012317f0a3e84eb1b87959170bfa0c32a2c141a4b3e51f8b07ed6` | no está en VT |
| `com.sdt.customization` 14 | `38aebe43c67aa8baebbb6581fd6883dfab9939296ed5c5b4fc35fcab2f8db1de` | no está en VT |
| `libtraceroute.so` (lib/) | `bd1639cd41d9b5c4a52eb3608ba256daa01575bdd8125fe0f2eaf66524c1a3a5` | – |
| `libtraceroute.so` (assets/) | `77446b7511e6389cbfe8273a2c9c8befe1f6ef4250ffb22e751b3072769eea81` | – |

Nombres de detección en los payloads de XCPE: Rising `Backdoor.XCPE/Android` (el único motor que nombra la familia), Dr.Web `Android.Backdoor.993.origin`, Kaspersky `HEUR:Backdoor.AndroidOS.Agent.gp`, ESET `Android/Agent.EYE`, AhnLab `Downloader/Android.Agent.1228987`. Microsoft no los marca en VirusTotal, pero el Defender local puso en cuarentena el `classes.dex` extraído como `Trojan:AndroidOS/Piom.V`.

## Revisa tu tvbox

Con ADB activado (Ajustes → Preferencias del dispositivo → Información → presiona Compilación 7 veces, y después Opciones de desarrollador → Depuración USB/de red):

```bash
# Si usas depuración inalámbrica, primero empareja (puerto y código salen en la pantalla de la tele):
   adb pair <ip-del-tvbox>:<puerto-de-emparejamiento>
# Después conecta (5555 o el puerto que muestre la depuracion):
   adb connect <ip-del-tvbox>:5555
#igual si te sale mas comodo puedes usar la aplicaion "ADBAppControl" ahi usas la consola, tambien es util por si quieres guardar algun apk o hacer la desinstalacion con un click, creo que no tienen version para mac o linux.

# ¿Está, y es el mini instalador o una versión completa?
adb shell "dumpsys package com.oemservice | grep -E 'versionName|codePath|installerPackageName|lastUpdateTime'"

# Qué hay en /oem
adb shell "ls -la /oem"

# ¿Está corriendo / hablando MQTT?
adb shell "ps -A | grep -i oemservice"
adb shell "logcat -d | grep -iE 'MqttService|x\.cpe|oemservice' | tail -20"

# Servicios de accesibilidad y administradores del dispositivo
adb shell settings get secure enabled_accessibility_services
adb shell "dumpsys device_policy | grep -iE 'admin|owner'"
```

Si `codePath` apunta a `/data/app/...` y la versión es 6.x, se instaló el backdoor completo, no solo el mini instalador.

## Lo que hice

```bash
adb shell pm uninstall --user 0 com.oemservice
adb shell pm uninstall --user 0 com.oem.qa
adb shell pm uninstall --user 0 com.sdt.backupappmanger
adb shell pm uninstall --user 0 com.sdt.ota          # opcional; total, sus OTAs traen de regalo el mismo backdoor (ver arriba)
adb shell pm disable-user --user 0 com.sdt.upgrade   # solo existe en C1.3.x; el otro camino de las OTAs
adb shell pm disable-user --user 0 com.sdt.producttest

# confirmar (sin -u, porque ese también lista los desinstalados)
adb shell "pm list packages | grep -iE 'oemservice|oem.qa|backupappmanger|sdt.ota'"
```

- Esto no borra nada de `/system` ni de `/oem` (para eso hace falta root). Oculta el paquete para el usuario 0. En mis pruebas aguantó los reinicios **y una OTA completa de firmware** (C1.3.1 → C1.3.3), pero **un reseteo de fábrica muy muy probablemente lo deshace porque esta en /oem**. Si alguna vez reseteas, repítelo antes de conectar la tvbox a internet, si puedes.
- El control remoto, el Wi-Fi, Chromecast y las apps de streaming siguieron funcionando.
- Como mi router no permite bloquear por dominio, agregué reglas de salida para las IP de las tvbox hacia TCP 16000, 1883, 8883, 8000 y 3885–3887 (y, si tu router deja, bloquea las IPs del operador que publicó Google: `219.134.163.210`, `219.134.163.211`, `38.145.197.135`, `23.94.49.57`). Es limitado, pero esos tvbox no tienen por que andar usando esos puertos. Si tu router o tu DNS permiten bloquear dominios, bloquea la lista de arriba. Mi conexión no tiene IPv6 público, así que no hay ruta para saltarse estas reglas; si la tuya tiene, replícalas en IPv6.
- Después de limpiar: quedo,  sin servicios de accesibilidad, sin administradores de dispositivo, sin VPN, sin tareas programadas del backdoor, sin animación de arranque reemplazada, y solo conexiones abiertas a Google, CDNs y Chromecast.

## Respuesta de MECOOL

Tras una consulta corta por el formulario de su tienda, el 4-11-2026 mandaron un comunicado estándar sobre el incidente XCPE (`service@mecool.com`, firmado por soporte técnico post-venta). Resumen de lo que dicen:

- MECOOL es una marca "operada de forma independiente"; los equipos los fabrica **Videostrong Technology Co., Ltd.**
- Videostrong "presentó una apelación y aclaración formal a Google".
- Hicieron una evaluación interna y, según ellos, **"los productos MECOOL existentes pueden seguir usándose de forma normal y segura"**.
- Van a "seguir revisando la seguridad del equipo y el firmware".

Lo que no dicen: nada sobre una OTA que saque XCPE, ni qué modelos o firmwares están afectados. Y la firma de [arriba](#firma) es `skydigital@skyworth.com`, no de Videostrong. Entre dos y cuatro días después de ese correo, mi tvbox recibió una OTA que volvió a instalar el mismo XCPE.

El reporte formal (con el [advisory en inglés](ADVISORY.md)) está pendiente de envío.

## Limitaciones

- Análisis estatico (strings, manifiestos, `dumpsys`) más una captura de red corta, hecha cuando el C2 ya estaba caído. Nunca lo vi recibir un comando.
- No descompilé el DEX a Java completo. La lista de capacidades sale de nombres de clases, strings y etc.
- Mi suposicion sobre el historial de versiones de la tvbox B es solo eso, una suposición.
- No probé si un reseteo de fábrica lo trae de vuelta. Eso es lo que dicen los reportes públicos y no quiero instalar todo nuevamente, algun dia supongo que lo tendre que hacer y antes de conectarlo a internet voy a desinstalar lo mismo de ahora.

## Fuentes

- [Shutting the door on the XCPE botnet (Google Threat Intelligence, 02-11-2026)](https://cloud.google.com/blog/topics/threat-intelligence/shutting-the-door-on-the-xcpe-botnet/)
- [Google disrupts million-device XCPE Android malware botnet, bans TV box maker (BleepingComputer)](https://www.bleepingcomputer.com/news/security/google-disrupts-million-device-xcpe-android-malware-botnet-bans-tv-box-maker/)
- [Inside XCPE, the botnet that turned set-top boxes into proxy servers (Security Affairs)](https://securityaffairs.com/202323/malware/inside-xcpe-the-botnet-that-turned-set-top-boxes-into-proxy-servers.html)
- [Tech Firms Dismantle Botnet That Turned 1.3M TV Boxes Into Proxies (Krebs on Security, 02-11-2026), página de la etiqueta](https://krebsonsecurity.com/tag/bmw-ipweb/)
- Fichas de VirusTotal de los hashes de arriba, incluido un grafo de la comunidad "MeCool KM2 Plus Deluxe" (13-12-2024).

---

Hecho en equipos propios y con fines defensivos. Aqui no hay muestras, solo hashes. Si encuentras algo que esté mal, dimelo :D
