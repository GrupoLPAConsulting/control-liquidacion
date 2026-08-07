# Control de Liquidación — LPA Consulting

Aplicación web para auditar automáticamente las liquidaciones mensuales de sueldos, comparándolas contra las escalas salariales oficiales, los convenios colectivos y los topes legales vigentes.

**App publicada:** https://grupolpaconsulting.github.io/control-liquidacion/ *(acceso protegido con contraseña)*

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
- La contrapartida: no hay multiusuario real ni historial compartido entre personas — cada instalación vive en el navegador de quien la usa. Ver [Próximos pasos](#próximos-pasos-y-decisiones-pendientes).

## Qué controla

La app aplica los siguientes controles sobre cada legajo, comparando el mes cargado más reciente contra el anterior:

| # | Control | Qué detecta |
|---|---|---|
| 1 | **Impuesto a las Ganancias** | Retención (`/4T2`) que supera el 35% del Total Remunerativo |
| 2 | **Tope MOPRE** | Aportes `/321` `/351` `/361` por encima del tope legal del período |
| 3 | **Tope MOPRE sobre SAC** | En meses de aguinaldo, aportes `/325` `/355` `/365` contra la mitad del tope |
| 4 | **Básico vs. escala** | El básico liquidado no coincide con el oficial de la categoría |
| 5 | **Presentismo** (FATLYF) | No cobra el 12% de escala, o el valor difiere |
| 6 | **Antigüedad** | Años derivados del adicional no dan un número entero; tramo del adicional incorrecto |
| 7 | **Concepto 0468** | No remunerativo exclusivo de FATLYF liquidado fuera de convenio, o ausente donde corresponde |
| 8 | **Servicio Extraordinario (0210)** | En FATLYF, debe ser el 11% del básico de escala; en APUAYE, hora extra variable (no se controla importe) |
| 9 | **0251 (A Cta. Futuros Aumentos)** | No acompañó el % de la última paritaria |
| 10 | **0022 (Asignación Personal)** | Mismo control que el anterior |
| 11 | **Aumentos por convenio** | El sueldo no reflejó la paritaria del período |

La clasificación de conceptos en remunerativos / no remunerativos / descuentos se lee dinámicamente de cada archivo cargado (usa las columnas `TOTAL REM` / `TOTAL NO REM` / `TOTAL DESC` del propio archivo), con una lista de respaldo fija para archivos en formato crudo que no traen esos marcadores.

## Convenios soportados

- **FATLYF** (FA)
- **APUAYE** (AY)
- **Fuera de convenio** (FC)

Cada uno con sus propias reglas de escala, antigüedad y conceptos de convenio.

## Funcionalidades principales

- **Lectura flexible de archivos:** acepta tanto el volcado crudo de SAP como el formato resumen/pivot de un archivo de control, detectando automáticamente cuál es.
- **Escalas salariales con historial:** se cargan subiendo el archivo oficial del convenio (Excel con una hoja por período); la app detecta los meses que trae y pide confirmar el período de cada uno. Quedan guardadas sin pisar las anteriores.
- **Parámetros protegidos:** topes, tasas y escalas están en solo lectura por defecto, con un botón explícito de "Habilitar edición" para evitar modificaciones accidentales.
- **Historial de meses:** al cargar un mes, la app ofrece guardarlo en el navegador (`IndexedDB`) para no tener que volver a subir el archivo en sesiones futuras. Es opt-in: se pregunta antes de guardar, y hay una pantalla dedicada para ver y borrar lo guardado.
- **Exportación a Excel:** genera un informe formateado (colores por severidad, columnas dimensionadas, filtros, hoja de metodología) del mes más reciente cargado, usando el anterior como referencia. Es un escritor de XLSX construido a mano, sin librerías.
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
- **Control de horas extras / tope de 30 h mensuales.** La fórmula del valor-hora (qué conceptos suman a la base y qué divisor corresponde, 176 u 184 según régimen horario) fue reconstruida y verificada contra datos reales de SAP (transacción `PC_PAYRESULT`) para FATLYF y Fuera de Convenio. Pendiente de implementar el control en la app.
- **Control de embargos.** Hay una hoja de referencia con los conceptos y bases de cálculo, pero requiere el dato del porcentaje/importe que ordena cada juzgado por legajo (vive en SAP, infotipos 14/15), que hoy no está disponible para cruzar automáticamente.
- **Dominio propio.** La URL actual depende de la cuenta de GitHub; para un despliegue institucional conviene evaluar un dominio propio de LPA Consulting.

## Stack técnico

- HTML / CSS / JavaScript vanilla — sin frameworks ni dependencias de build
- Lectura/escritura de `.xlsx` implementada a mano (ZIP + XML) sobre `DecompressionStream`
- Persistencia: `localStorage` (parámetros) + `IndexedDB` (historial de meses)
- Cifrado: `Web Crypto API` (AES-GCM + PBKDF2)
- Publicación: GitHub Pages
- Automatización del tope MOPRE: GitHub Actions (Python)

## Historial de versiones

La app usa un número de versión visible en el encabezado para verificar que un despliegue se aplicó correctamente. Evolucionó de v2.1 a v3.0 de forma incremental, con cada control validado contra archivos reales de liquidación (comparación al centavo, cientos de legajos) antes de darse por cerrado.

## Créditos

Desarrollado para LPA Consulting. Auditoría de nóminas y definición de reglas de negocio: Sergio García, bajo supervisión de Victoria Croulliere. Implementación: asistida por Claude (Anthropic).
