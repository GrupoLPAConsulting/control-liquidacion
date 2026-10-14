# Control de Liquidación — LPA Consulting

Aplicación web para auditar automáticamente las liquidaciones mensuales de sueldos, comparándolas contra las escalas salariales oficiales, los convenios colectivos y los topes legales vigentes.

**App publicada:** https://grupolpaconsulting.github.io/control-liquidacion/ *(acceso protegido con contraseña)*

**Versión actual:** 4.6.9

---

## Qué problema resuelve

El control de una liquidación de nómina implica cruzar, para cada legajo, decenas de conceptos contra reglas de convenio, escalas de categoría, topes de aportes y variaciones mes a mes. Hecho a mano, es lento y propenso a errores. Esta app automatiza ese cruce: se cargan los archivos de liquidación de dos meses consecutivos y el sistema calcula solo qué legajos están en regla y cuáles requieren revisión, con el criterio exacto de cada control documentado.

Nació como herramienta interna para el área de auditoría de nóminas de Edersa (eléctrica argentina) y evolucionó a un producto pensado para reutilizarse en distintas empresas y convenios.

## Cómo funciona (arquitectura)

**Es un único archivo HTML autocontenido.** No hay backend, no hay base de datos externa, no se instala nada. Todo el motor —lectura de Excel, cálculo de los controles, generación del informe final— corre en JavaScript dentro del propio navegador.

Esta decisión de diseño fue deliberada y tiene consecuencias importantes:

- **Los datos de los empleados nunca salen de la computadora del usuario.** Los archivos de liquidación se procesan localmente; nada se envía a un servidor.
- **Funciona sin conexión a internet**, una vez descargado.
- **No depende de librerías externas.** El archivo escribe y lee XLSX con un parser propio (formato ZIP + XML), y usa las APIs nativas del navegador (`DecompressionStream`, `IndexedDB`, `Web Crypto`) en vez de paquetes de terceros.
- El historial local puede trasladarse entre instalaciones mediante un respaldo completo sin cifrar. El respaldo guarda sólo datos de entrada y parámetros; la versión que lo restaura vuelve a calcular todos los resultados con sus propias reglas.

## Qué controla

La app aplica los siguientes controles sobre cada legajo, comparando el mes cargado más reciente contra el anterior:

| # | Control | Qué detecta |
|---|---|---|
| 1 | **Impuesto a las Ganancias** | Retención (`/4T2`) que supera el 35% del Total Remunerativo |
| 2 | **Tope MOPRE** | Aportes `/321` `/351` `/361` por encima del tope legal del período |
| 3 | **Tope MOPRE sobre SAC** | En meses de aguinaldo, aportes `/325` `/355` `/365` contra la mitad del tope |
| 4 | **Básico vs. escala** | El básico liquidado no coincide con el oficial de la categoría |
| 5 | **Presentismo** (FATLYF) | Audita 0306/0385 únicamente cuando alguno viene informado; la ausencia de ambos no genera aviso |
| 6 | **Antigüedad** | Años derivados del adicional no dan un número entero; tramo del adicional incorrecto |
| 7 | **Concepto 0468** | No remunerativo exclusivo de FATLYF liquidado fuera de convenio, o ausente donde corresponde |
| 8 | **Servicio Extraordinario (0210)** | FATLYF: 11% del básico; APUAYE: valores A/B de escala (14% o 42%) |
| 9 | **Horas extra 0308/0309** | Recalcula cantidad, base, factor y divisor real de HTrab; 0310 queda expresamente informativo |
| 10 | **Vacaciones** | Recalcula `/109`, `0316` y `9322` con RNHyP, bono técnico y seis meses de origen HE previos; aclara el desfase entre origen y pago |
| 11 | **Bono para vacaciones** | APUAYE usa 9RAV; FATLYF y FC usan 99BA; separa movimientos `000000` |
| 12 | **Embargos y cuotas alimentarias** | Aplica el porcentaje informado y las bases documentadas para cada CC-nómina |
| 13 | **0251 y 0022** | Controla que acompañen el % de la paritaria |
| 14 | **Aumentos por convenio** | Distingue faltantes, aumentos menores y posibles recategorizaciones |
| 15 | **Retroactivos y bajas** | Imputa retroactivos de HE a su período de origen y neutraliza liquidaciones finales detectadas por `/114` o `/T53` |

La clasificación de conceptos en remunerativos / no remunerativos / descuentos se lee dinámicamente de las hojas Control cargadas y se acumula como historial. Un concepto nuevo queda disponible en los meses siguientes; si cambia de grupo, prevalece la clasificación más reciente. Existe además una lista de respaldo fija para archivos crudos sin marcadores.

## Convenios soportados

- **FATLYF** (FA)
- **APUAYE** (AY)
- **Fuera de convenio** (FC)

Cada uno con sus propias reglas de escala, antigüedad y conceptos de convenio.

## Funcionalidades principales

- **Lectura flexible de archivos:** acepta tanto el volcado crudo de SAP como el formato resumen/pivot de un archivo de control, detectando automáticamente cuál es.
- **Escalas salariales con historial:** se cargan subiendo el archivo oficial del convenio (Excel con una hoja por período); la app detecta los meses que trae y pide confirmar el período de cada uno. Quedan guardadas sin pisar las anteriores y la última continúa vigente hasta una actualización. Si no se informa la paritaria, se calcula desde la nueva escala o se aplica 0% cuando no hubo cambio de escala.
- **Parámetros protegidos:** topes, tasas y escalas están en solo lectura por defecto, con un botón explícito de "Habilitar edición" para evitar modificaciones accidentales.
- **Historial de meses:** al cargar un mes, la app ofrece guardarlo en el navegador (`IndexedDB`) para no tener que volver a subir el archivo en sesiones futuras. Es opt-in: se pregunta antes de guardar, y hay una pantalla dedicada para ver y borrar lo guardado.
- **Respaldo completo portable:** descarga y restaura meses, Queries, escalas, topes, paritarias, categorías y parámetros. Valida integridad, evita sobrescribir períodos locales en conflicto, crea una copia previa y recalcula con la versión actual. El archivo es deliberadamente sin cifrar.
- **Lectura XLSX ampliada:** admite libros con o sin prefijos de espacios de nombres XML y conserva los ceros iniciales de las CC-nómina (`0001`, `0316`, etc.).
- **Exportación a Excel:** genera un informe formateado (colores por severidad, columnas dimensionadas, filtros, hoja de metodología) del mes más reciente cargado, usando el anterior como referencia. Es un escritor de XLSX construido a mano, sin librerías.
- **Avisos sin falsos rojos:** los centavos de embargos se tratan como redondeo; la ausencia de presentismo no genera aviso; las altas y los cambios de categoría confirmados quedan explicados, y las variaciones salariales FC no confirmadas permanecen visibles.
- **Actualización automática del tope MOPRE:** un workflow de GitHub Actions (`actualizar-tope.yml`) corre periódicamente, lee la resolución oficial de ANSES y confirma el tope por dos vías independientes (scraping de la página oficial + cálculo por fórmula de movilidad). La app lee ese archivo (`data/topes.json`) al abrirse con conexión.

## Seguridad y acceso

**El repositorio y el sitio publicado están protegidos con contraseña.** La protección se implementa por cifrado real del contenido (AES-GCM 256 bits, clave derivada con PBKDF2-SHA256 y 600.000 iteraciones vía Web Crypto API), no por una simple verificación en el código — el archivo publicado es, literalmente, texto cifrado hasta que se ingresa la clave correcta en el navegador.

Herramienta auxiliar: `Encriptador.html` (no forma parte de la app; se usa una sola vez para generar el `index.html` protegido a partir del original).

**Qué protege y qué no, para que quede explícito:**

| | |
|---|---|
| ✅ Protege | El código fuente y la lógica de los controles, ante cualquiera que encuentre el repositorio o descargue el archivo |
| ✅ Protege | Fuerza bruta de la contraseña (cada intento cuesta ~0.3s por diseño) |
| ❌ No protege | Los datos guardados en el historial local (`IndexedDB`) de una computadora ya autorizada — quedan accesibles a quien use esa máquina |
| ❌ No protege | Es una contraseña única compartida, no hay usuarios ni permisos individuales |

Esta combinación es adecuada para el estado actual (uso por una o pocas personas, en equipos de confianza) pero **no reemplaza un sistema de autenticación real con servidor**. Ver el punto siguiente.

## Próximos pasos y decisiones pendientes

Estos puntos requieren decisión de negocio / infraestructura, no son bugs:

- **Multiusuario con login por correo.** Requiere backend (servidor + base de datos), lo cual implica decidir dónde vive esa infraestructura — nube de terceros vs. servidor interno de la empresa — dado que implicaría alojar datos de sueldos fuera de la computadora del usuario. Es el cambio de arquitectura más importante si el proyecto escala a varias personas o empresas.
- **Vacaciones en meses consecutivos.** Se mantienen visibles como revisión hasta confirmar con Victoria si SAP cambia la selección de la ventana de horas extra o aplica otro criterio.
- **Rama 0212.** Se conserva la lectura respaldada por la escala y los datos reales; cualquier cambio adicional requiere definición de negocio.
- **Dominio propio.** La URL actual depende de la cuenta de GitHub; para un despliegue institucional conviene evaluar un dominio propio de LPA Consulting.

## Stack técnico

- HTML / CSS / JavaScript vanilla — sin frameworks ni dependencias de build
- Lectura/escritura de `.xlsx` implementada a mano (ZIP + XML) sobre `DecompressionStream`
- Persistencia: `localStorage` (parámetros) + `IndexedDB` (historial de meses)
- Cifrado: `Web Crypto API` (AES-GCM + PBKDF2)
- Publicación: GitHub Pages
- Automatización del tope MOPRE: GitHub Actions (Python)

## Historial de versiones

La versión actual es **v4.6.9**. Es una actualización de mantenimiento sobre la v4.6.8: conserva todas las fórmulas y reglas validadas y corrige la fuente usada en las pruebas de septiembre. La aceptación se ejecuta con la CWTR final del 30/09/2026, donde el embargo 0995 del legajo 5411 cierra al centavo y el concepto 9322 se clasifica como no remunerativo. Para vacaciones se mantienen los seis meses de origen marzo–agosto, equivalentes a pagos abril–septiembre. La distribución incluye `Control_Liquidacion.exe`, nombre fijo recomendado para el acceso directo compartido.

## Créditos

Desarrollado para LPA Consulting. Auditoría de nóminas y definición de reglas de negocio: Sergio García, bajo supervisión de Victoria Croulliere. Implementación asistida por IA y validada contra archivos reales de SAP.
