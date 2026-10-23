# Pulse personal: móvil, NAS y Coach con openGym

## Estado

**En descubrimiento.** Esta hoja de ruta recoge lo decidido en la conversación del 23 de octubre de 2026. La primera prioridad móvil ya está definida; las demás notas de uso, el modelo de NAS y los formatos de InBody siguen pendientes. Ninguna fase posterior autoriza por sí sola publicar datos de salud ni enviar mensajes por Telegram.

## Objetivo

Usar Pulse como vista y asistente personal de salud, conservando openGym como herramienta para planificar y registrar entrenamientos. Pulse reunirá las señales de Google Health/Fitbit, el detalle de fuerza de openGym y, más adelante, mediciones InBody verificadas. El Coach podrá consultar esas fuentes y, si se habilita expresamente, enviar resúmenes útiles por Telegram.

## Principios de trabajo

- Cada fuente conserva su función: Google Health aporta señales fisiológicas; openGym aporta rutinas, sesiones y series; InBody aporta mediciones puntuales. Cada dato mostrado conserva fuente, fecha, unidad y estado de sincronización.
- El importador de openGym y las herramientas nuevas del Coach empiezan en **solo lectura**. Las acciones que escriben datos o envían mensajes requieren una decisión posterior.
- Una actividad presente en Google y openGym aparece una vez en Pulse, con los detalles de ambas fuentes y sin duplicar la carga de entrenamiento.
- La UI se decide con problemas observados en demo y notas personales, y se valida en un iPhone real antes de darla por terminada.
- Se desarrolla en el fork personal mediante ramas pequeñas. `upstream` queda disponible para incorporar mejoras del proyecto original sin mezclar secretos ni datos personales en Git.

## Estado de partida verificado

- Fork: `https://github.com/dufferx/pulse`, derivado de `adityaongit/pulse`. En este checkout `origin` apunta al fork y `upstream` al proyecto original.
- Pulse ofrece datos demo y un barrido Playwright móvil; `playwright.config.ts` separa servidores de prueba. La UI principal usa `src/app/(app)/` y los componentes compartidos de `src/components/shells/`.
- Home ya permite añadir, quitar y reordenar **métricas dentro de “My Dashboard”** mediante `EditDashboard.tsx` y `saveDashboard`. El resto de bloques de `src/app/(app)/(home)/page.tsx` tiene orden fijo en la composición actual. Por tanto, la petición de personalización amplía una función existente; no parte de cero.
- El Coach ya usa herramientas cerradas sobre el usuario autenticado en `src/server/coach/tools.ts`. La tabla `exercises` de Pulse guarda sesiones resumidas; no guarda ejercicios, series, repeticiones ni cargas de openGym.
- Pulse corre con Postgres en `compose.yaml`; `docs/setup.md` cubre NAS y Tailscale, y `docs/admin.md` describe respaldo/restauración de Postgres y actualización con `scripts/deploy.sh`.
- openGym ofrece modo móvil local y modo emparejado con un servidor propio; el modo local no sincroniza automáticamente con el NAS. Su API permite leer el estado completo de una cuenta (`GET /api/data`), y su MCP local ofrece consultas de solo lectura sobre entrenamientos y rutinas. Fuentes: [móvil](https://github.com/DuarteSantos8/openGym/blob/main/docs/MOBILE.md), [API](https://github.com/DuarteSantos8/openGym/blob/main/api/openapi.yaml), [MCP](https://github.com/DuarteSantos8/openGym/blob/main/mcp/README.md).
- La escritura de openGym en Apple Health no se adopta como canal principal de integración: Pulse corre en el servidor y actualmente importa desde Google Health. Leer HealthKit desde una app iOS requeriría permiso y desarrollo específicos.

## Fase 1 — Descubrir y mejorar la UI móvil de Pulse

**Prioridad confirmada por el usuario.** Home se siente abrumadora. Quiere elegir cuáles de las secciones y widgets **existentes** aparecen y en qué orden, con una edición visual sencilla inspirada en la organización de widgets de la pantalla de inicio de Apple. No solicita métricas nuevas. Aún no ha probado Pulse lo suficiente para cerrar otras mejoras.

**Entradas.** Auditoría con 180 días de datos demo y notas personales adicionales del usuario. Registrar cada nota como: pantalla, tarea que intentaba hacer, fricción actual, resultado deseado, frecuencia y prioridad. Se pueden adjuntar capturas sin datos sensibles.

**Auditoría.** Revisar Home, recuperación, sueño, esfuerzo, salud, actividades, diario, Coach, configuración y onboarding en 375×667, 390×844 y 430×932; incluir navegación, áreas táctiles, texto, gráficos, teclado, orientación, zonas seguras, carga, errores, datos ausentes y estados provisionales. Separar hallazgos reproducibles de hipótesis. Medir las pantallas que se sientan lentas; el proyecto ya documenta un caso de lentitud de renderizado móvil en `docs/plans/2026-10-09-002-app-speed-measured.md`.

**Auditoría preliminar estática.** El agente encontró asuntos que necesitan validación en una demo aislada o un iPhone: descubribilidad del botón del Coach marcado solo con «P», navegación desde páginas de detalle, acceso a Reports/Trends/Activities desde More, legibilidad de algunos controles pequeños y facilidad de arrastrar en el editor actual. También observó que capturas antiguas del editor y de Trends ya no representan el código actual. No se hizo prueba dinámica después de detectar un conflicto de nombres de Docker Compose con la instalación real; la pila original fue restaurada y `/healthz` respondió 200. Ninguno de esos asuntos se presenta aún como bug confirmado por uso.

**Primer cambio de producto.** Inventariar todos los bloques que ya aparecen en Home (diales, tarjetas, My Day, My Dashboard, gráfico, reporte y otros estados condicionales). Diseñar un modo de edición para mostrar/ocultar y ordenar bloques, además de las métricas que el editor actual ya maneja; incluir vista previa, guardar, cancelar y restaurar una disposición inicial. Resolver en diseño cómo se comportan bloques sin datos, de días anteriores y con distintos tamaños de pantalla. Las decisiones visuales exactas se revisan con el usuario antes de implementar.

**Diseño y entrega.** Ordenar los demás hallazgos por impacto y frecuencia; acordar las tareas móviles siguientes; crear bocetos o prototipo; implementar por pantallas en ramas pequeñas, reutilizando shells, componentes y tokens de Pulse. Cada cambio visible también actualiza `site/` según `AGENTS.md`. Validar con `pnpm typecheck && pnpm lint && pnpm test`, `pnpm e2e` y prueba manual en iPhone/PWA.

**Salida.** Home puede reducirse y reorganizarse usando su contenido existente; la disposición persiste entre sesiones y se comprueba en teléfono. Además: registro priorizado de problemas con evidencia, notas del usuario incorporadas y recorridos móviles sin errores bloqueantes. El alcance de otras mejoras se decide después de probar más la app.

## Fase 2 — Desplegar Pulse y openGym en el NAS

**Preparación.** Elegir hardware y sistema; confirmar arquitectura CPU, RAM, espacio, Docker/Compose, acceso SSH, energía y política de actualizaciones. Definir desde el principio dos direcciones HTTPS estables, porque Pulse necesita URL/redirect de Google OAuth y openGym vincula sus passkeys a su origen. Mantener las aplicaciones y sus volúmenes separados.

**Despliegue.** Instalar Tailscale o el acceso privado elegido; desplegar Pulse con Postgres y openGym con sus datos persistentes; configurar el OAuth de Google para la nueva URL; crear o restaurar las cuentas; emparejar la app iOS de openGym con **su** servidor si se desea sincronización. Decidir con un inventario si la instancia actual de Pulse se migra mediante dump/restauración o si se inicia limpia.

**Operación.** Respaldar Postgres de Pulse, datos JSON/media de openGym, archivos InBody futuros y configuración necesaria. Cifrar las copias fuera del NAS; probar una restauración real en un entorno aislado. Vigilar espacio, salud de contenedores y fallos de sincronización. Fijar versiones para desplegar y usar ramas/entorno de prueba para incorporar actualizaciones de `upstream` antes de promoverlas.

**Salida.** Ambas apps accesibles desde iPhone y escritorio, datos persistentes, respaldo y restauración probados, OAuth y passkeys funcionando y procedimiento de actualización/reversión escrito.

## Fase 3 — Unir openGym con Pulse

**Prueba de concepto.** Con ambas fuentes ya en el NAS, comparar dos accesos de solo lectura: (A) API autenticada de openGym; (B) lectura del volumen de datos o su MCP local dentro del NAS. Seleccionar según aislamiento entre usuarios, facilidad de mantenimiento, formato disponible y tratamiento seguro de credenciales. No asumir que el MCP local es una API de red.

**Modelo.** Importar sesiones, ejercicios, series, repeticiones, peso, RPE/RIR cuando exista, rutinas y cambios/eliminaciones. Guardar identificador de origen, fecha/hora con zona, unidad, fecha de sincronización y vínculo opcional con una actividad Google. La importación será repetible sin duplicados y no modificará openGym.

**Producto.** Añadir a Pulse una vista de fuerza con historial y progreso, y un detalle de actividad que combine los datos de ambas fuentes cuando se correspondan. Después añadir herramientas de lectura del Coach para historial, sesión, rutina y progreso. No introducir series de openGym directamente en la tabla resumida `exercises` ni alterar los algoritmos de recuperación/esfuerzo sin validación específica.

**Salida.** La misma sesión puede verse en openGym y en Pulse; Pulse muestra el detalle de fuerza y las métricas Google sin doble conteo; el Coach responde preguntas cruzadas con datos trazables; una sincronización fallida muestra su estado sin inventar números.

## Fase 4 — Incorporar análisis InBody

**Descubrimiento.** Obtener muestras redactadas de los formatos reales del usuario (PDF, imagen, CSV u otro), fechas, unidades y campos relevantes. Acordar qué mediciones conviene conservar y cómo se muestran sus cambios en el tiempo.

**Flujo.** Subida privada, extracción/OCR cuando haga falta, revisión humana de cada campo antes de guardar, conservación del documento original y del valor corregido, con fecha y procedencia. Mostrar tendencias y dar al Coach una herramienta de lectura de mediciones confirmadas. Los campos dudosos permanecen sin valor confirmado.

**Salida.** Un archivo real puede importarse, corregirse, consultarse y borrarse; el Coach distingue las mediciones InBody de las estimaciones de otros dispositivos.

## Fase 5 — Personalizar y evaluar el Coach

Primero probar el Coach existente con las fuentes conectadas y registrar respuestas útiles, insuficientes o incorrectas. Definir objetivos personales, preferencias de entrenamiento, tono, límites y qué información se comparte con el proveedor de modelo. Añadir herramientas de consulta antes de ampliar instrucciones. Evaluar con preguntas propias: progreso de fuerza, relación entrenamiento/sueño/recuperación, tendencias InBody y planificación semanal. Cada consejo debe mostrar qué datos lo sustentan y reconocer datos ausentes. Mantener la aprobación explícita para cualquier escritura, como ya hace el Coach de Pulse.

**Salida.** Un conjunto de preguntas reales pasa una revisión manual; el Coach no confunde correlación con causa, no cita datos inexistentes y respeta las preferencias configuradas.

## Fase 6 — Bot de Telegram y reportes proactivos

**Primera versión.** Vincular un chat de Telegram con la cuenta Pulse mediante un código de un solo uso; aceptar mensajes solo del chat autorizado. Permitir preguntas que usan las mismas herramientas de lectura del Coach. Enviar un resumen diario y otro semanal en la zona horaria del perfil, con horario configurable, pausa y cancelación. Evitar informes duplicados y mostrar cuándo los datos están atrasados.

**Privacidad.** Decidir explícitamente si Telegram recibe solo resúmenes breves y enlaces privados o también detalles de salud. El contenido de Telegram sale del NAS hacia ese servicio; el usuario debe poder ver y controlar lo que se envía. No poner secretos, archivos InBody completos ni respuestas de API en logs o mensajes.

**Salida.** Mensajes de prueba autorizados, reportes útiles y puntuales, opción de pausar/desvincular, y ninguna respuesta a chats ajenos.

## Dependencias y orden

`Notas + auditoría demo → prioridades y UI móvil → NAS con ambas apps → importación openGym → InBody → evaluación/mejora del Coach → Telegram`.

La primera evaluación del Coach actual puede empezar en paralelo a la preparación del NAS, pero las respuestas cruzadas necesitan la integración de openGym. Los reportes por Telegram se diseñan después de comprobar que las respuestas del Coach son fiables.

## Decisiones pendientes (máximo tres para esta etapa)

1. **Notas móviles adicionales:** la Home configurable es prioridad confirmada; las otras mejoras esperan más uso y una auditoría dinámica aislada.
2. **NAS:** el usuario aún no eligió equipo. Modelo/sistema/RAM y si se migran los datos de Pulse actuales o se comienza una instancia limpia determinan la Fase 2.
3. **Datos y mensajería:** muestra de InBody y preferencia de privacidad para Telegram. Se resuelve antes de las fases 4 y 6; no bloquea la auditoría móvil.

## Próxima acción

Completar la auditoría dinámica en una demo con nombre de proyecto/volúmenes aislados; después prototipar la Home configurable sobre el inventario de bloques existentes y validar su interacción con el usuario. Añadir las demás notas conforme el usuario pruebe Pulse. Convertir la Fase 1 en una lista cerrada de cambios y criterios de aceptación antes de implementarla; mantener las demás fases como hoja de ruta hasta resolver sus entradas.
