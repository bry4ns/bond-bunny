# SRTLA: continuidad durante cambios de red

El emisor Android usado por Perhost Live tenía tres problemas al renovar o perder
una red: `onLost` eliminaba el transporte sin comprobar a qué `Network` pertenecía;
el emisor C conservaba el FD anterior cuando el nuevo socket tenía la misma IP
virtual; y el listado de conexiones se borraba y reescribía mientras C podía leerlo.

Ahora se comprueba la identidad de la red activa, se publica el listado con
`AtomicFile` y el hilo emisor adopta el socket nuevo, reiniciando únicamente la
ventana y el registro de ese enlace. Una red celular que sigue disponible conserva
su socket, ventana y paquetes pendientes. Los cambios de LinkProperties también
permiten detectar direcciones que llegan después de onAvailable.

Las notificaciones se consumen mediante intercambio atómico para no borrar un
cambio recibido durante update_conns. Las estadísticas JNI y la lista de conexiones
se protegen con un mutex: retirar un enlace no puede liberar la memoria mientras
el panel la recorre. El indicador global de reconexión solo se activa cuando no
queda ningún enlace operativo.

## Fuente nativa

`srtla-lib/src/main/cpp/srtla_send_android.c` conserva el emisor de
[dimadesu/srtla, b81ceb604e5d848aa1e6cf2157114f4cf9708b9b](https://github.com/dimadesu/srtla/tree/b81ceb604e5d848aa1e6cf2157114f4cf9708b9b)
con estos cambios locales. CMake compila este archivo; common.c, common.h,
android_compat.h y la licencia AGPL-3.0 siguen procediendo del submódulo original.
Al actualizar upstream, comparar ambos emisores y conservar las correcciones.

## Validación

`sh tests/native/run.sh` compila el emisor real con AddressSanitizer y
UndefinedBehaviorSanitizer. Solo se sustituyen el registro Android y los callbacks
JNI; los sockets UDP, select, registro de conexiones y cambios de FD son reales.

Comprueba:

- Reemplazo del socket Wi-Fi bajo la misma IP virtual.
- Conservación del socket y ventana celular.
- Eliminación de Wi-Fi sin desconectar la conexión restante.
- Estado global correcto cuando un enlace falla y otro sigue activo.
- Notificaciones atómicas, publicación concurrente de sockets y lectura concurrente
  de estadísticas durante 100 ciclos de incorporación y retirada de Wi-Fi.

La prueba pasó en el entorno de revisión. LeakSanitizer no puede ejecutarse bajo
el aislamiento de procesos de ese entorno; se usó
`ASAN_OPTIONS=detect_leaks=0 sh tests/native/run.sh`, manteniendo ASan y UBSan.
No sustituye una prueba SRTLA con servidor, Android, codecs y redes móviles reales.
