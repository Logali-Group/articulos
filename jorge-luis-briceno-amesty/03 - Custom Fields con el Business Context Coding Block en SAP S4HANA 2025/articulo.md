<style>
body { text-align: justify; hyphens: auto; }
img { display: block; margin: 0 auto; }
p:has(> img) { text-align: center; }
</style>

# ¿Cómo agregar un Custom Field a la app Upload General Journal Entries (F2548)?

- [1. Introducción](#1-introducción)
- [2. El escenario y los tres pilares de la extensibilidad](#2-el-escenario-y-los-tres-pilares-de-la-extensibilidad)
    - [2.1. El sistema del ejercicio](#21-el-sistema-del-ejercicio)
    - [2.2. Los tres pilares de la extensibilidad](#22-los-tres-pilares-de-la-extensibilidad)
- [3. La ficha técnica de la aplicación en la SAP Fiori Apps Reference Library](#3-la-ficha-técnica-de-la-aplicación-en-la-sap-fiori-apps-reference-library)
    - [3.1. La nueva biblioteca de referencia](#31-la-nueva-biblioteca-de-referencia)
    - [3.2. Información técnica del F2548](#32-información-técnica-del-f2548)
    - [3.3. La sección App Extensibility](#33-la-sección-app-extensibility)
- [4. El Business Context y el registro de extensibilidad](#4-el-business-context-y-el-registro-de-extensibilidad)
    - [4.1. Qué es un Business Context](#41-qué-es-un-business-context)
    - [4.2. La transacción SCFD_REGISTRY](#42-la-transacción-scfd_registry)
    - [4.3. El detalle del contexto y los includes INCL_EEW_COBL](#43-el-detalle-del-contexto-y-los-includes-incl_eew_cobl)
- [5. La creación del campo custom](#5-la-creación-del-campo-custom)
    - [5.1. La app Custom Fields y el diálogo New Field](#51-la-app-custom-fields-y-el-diálogo-new-field)
    - [5.2. Qué hacer si el Business Context no aparece en el listado](#52-qué-hacer-si-el-business-context-no-aparece-en-el-listado)
    - [5.3. Qué hacer si el tipo de dato no aparece en el listado](#53-qué-hacer-si-el-tipo-de-dato-no-aparece-en-el-listado)
    - [5.4. La transacción SCFD_EUI y los campos ya existentes](#54-la-transacción-scfd_eui-y-los-campos-ya-existentes)
- [6. Anatomía del campo generado](#6-anatomía-del-campo-generado)
    - [6.1. El prefijo ZZ1_ y el sufijo por contexto](#61-el-prefijo-zz1_-y-el-sufijo-por-contexto)
    - [6.2. La propagación hacia ACDOCA](#62-la-propagación-hacia-acdoca)
    - [6.3. Cabecera frente a posición: el alcance del coding block](#63-cabecera-frente-a-posición-el-alcance-del-coding-block)
- [7. Las pestañas de habilitación de usos](#7-las-pestañas-de-habilitación-de-usos)
    - [7.1. El significado de cada pestaña](#71-el-significado-de-cada-pestaña)
    - [7.2. APIs, BAPIs y SOAP APIs](#72-apis-bapis-y-soap-apis)
    - [7.3. El método jerárquico para decidir qué habilitar](#73-el-método-jerárquico-para-decidir-qué-habilitar)
    - [7.4. Los usos habilitados en el ejercicio](#74-los-usos-habilitados-en-el-ejercicio)
- [8. Publicación del campo](#8-publicación-del-campo)
- [9. Auditoría con SCFD_FIELDVIEWER](#9-auditoría-con-scfd_fieldviewer)
- [10. Verificación en el diccionario ABAP y en las CDS views](#10-verificación-en-el-diccionario-abap-y-en-las-cds-views)
- [11. La plantilla Excel y el posteo del asiento](#11-la-plantilla-excel-y-el-posteo-del-asiento)
    - [11.1. La plantilla estándar](#111-la-plantilla-estándar)
    - [11.2. El catálogo de columnas de la plantilla](#112-el-catálogo-de-columnas-de-la-plantilla)
    - [11.3. La plantilla con datos del asiento](#113-la-plantilla-con-datos-del-asiento)
    - [11.4. El resultado del posteo](#114-el-resultado-del-posteo)
- [12. Lógica de validación con Custom Logic](#12-lógica-de-validación-con-custom-logic)
- [13. Notas SAP de interés](#13-notas-sap-de-interés)
- [14. Referencias](#14-referencias)
- [Conclusión](#conclusión)

---

## 1. Introducción

Todo consultor financiero de SAP se ha encontrado alguna vez con esta petición: el departamento de contabilidad necesita clasificar sus asientos con una dimensión propia del negocio que el estándar no contempla, por ejemplo una "unidad estratégica" que agrupe las partidas por iniciativas corporativas. La petición parece pequeña, pero abre de inmediato una cadena de preguntas que este artículo se propone responder una a una. ¿Qué es exactamente un Business Context y dónde está declarado dentro del sistema? ¿Por qué el campo termina llamándose `ZZ1_ESU_COB` si nadie escribió ese nombre? ¿Qué opciones quedan cuando el Business Context que se necesita no existe, o cuando el dato debe vivir en la cabecera del asiento y no en cada posición: Developer Extensibility, extensibilidad clásica? ¿Cómo se decide, entre centenares de artefactos habilitables, cuáles activar y cuáles no? Y una vez publicado el campo, ¿existe un lugar único donde auditar absolutamente todo lo que el framework generó, y se le puede además añadir lógica de validación sin salir de la extensibilidad de key user?

Para responder esas preguntas con evidencia y no con teoría de manual, el artículo documenta de principio a fin la creación de un campo personalizado sobre el Business Context **Accounting: Coding Block** y su consumo desde la aplicación Fiori **Upload General Journal Entries (F2548)**, la app con la que un contable sube asientos masivos desde una plantilla de Excel. El ejercicio completo se realizó sobre un sistema **SAP S/4HANA 2025 On-Premise**, de modo que todo lo descrito aplica a instalaciones locales y de Private Cloud, no solo a la edición Public Cloud donde nació la extensibilidad de key user.

El recorrido pasa por la ficha técnica de la app, el registro de extensibilidad `SCFD_REGISTRY`, la creación y habilitación del campo, la auditoría con `SCFD_FIELDVIEWER` (una transacción que merece ser mucho más conocida), la verificación en ADT, el posteo real de un asiento con la dimensión nueva, el alcance de cabecera frente a posición del coding block y la lógica de validación posterior con Custom Logic. Cada pregunta de la apertura encontrará su respuesta en el camino, y la conclusión las recapitula para que sirvan de lista de verificación en proyectos reales.

## 2. El escenario y los tres pilares de la extensibilidad

Antes de crear el primer campo conviene fijar el terreno: qué sistema se está extendiendo y qué familia de herramientas de extensibilidad corresponde usar. Elegir mal en este punto es como elegir mal el tipo de proyecto en SAP Business Application Studio: el trabajo posterior puede funcionar, pero queda construido sobre la base equivocada y tarde o temprano obliga a rehacer.

### 2.1. El sistema del ejercicio

El ejercicio se ejecutó sobre un SAP S/4HANA 2025 On-Premise con su stack inicial. La verificación se hace desde el SAP GUI, en el menú *System > Status*, abriendo el detalle de producto instalado. La **Figura 1** muestra el diálogo Installed Software del sistema utilizado, con los cinco productos que componen la instalación.

![Figura 1. Diálogo Installed Software con SAP S4HANA On-Premise 2025 y su stack inicial.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/03%20-%20Custom%20Fields%20con%20el%20Business%20Context%20Coding%20Block%20en%20SAP%20S4HANA%202025/figura-1.png)

*Figura 1. Diálogo Installed Software con SAP S4HANA On-Premise 2025 y su stack inicial.*

La pestaña Installed Product Versions confirma S4HANA ON PREMISE 2025, ABAP PLATFORM 2025, SAP S/4HANA FOUNDATION 2025 y los componentes de front-end SAP FIORI FOR S4HANA 2025 y SAP FIORI FES 2025. Este dato no es decorativo: la disponibilidad de Business Contexts, de tipos de dato y de usos habilitables en la app Custom Fields depende de la versión del producto, por lo que cualquier paso a paso de extensibilidad debe declarar sobre qué release se ejecutó.

### 2.2. Los tres pilares de la extensibilidad

SAP organiza la extensibilidad de S/4HANA en tres grandes familias, y cada petición de extensión debe clasificarse primero en una de ellas. La siguiente tabla resume los tres pilares con su herramienta principal y su caso de uso típico.

| Pilar | Quién lo ejecuta | Herramientas | Caso de uso típico |
|---|---|---|---|
| Key User Extensibility (in-app) | Key user o consultor funcional | Apps Fiori del grupo Key User Extensibility (Custom Fields, Custom Logic, Custom CDS Views, UI Adaptation) | Añadir campos, lógica ligera y adaptaciones de UI dentro de los límites que el estándar expone |
| Developer Extensibility (on-stack) | Desarrollador ABAP | ADT en Eclipse, ABAP Cloud, RAP, ABAP clásico en on-premise | Modelos de datos propios, lógica compleja, extensiones de BOs liberados |
| Side-by-Side Extensibility | Desarrollador BTP | SAP BTP, CAP, SAPUI5, integración vía APIs liberadas | Aplicaciones desacopladas que consumen el core por OData o eventos |

La elección entre pilares se puede visualizar como la elección entre modificar una clase, heredar de ella o consumirla por interfaz en un desarrollo orientado a objetos: cuanto más cerca del núcleo se trabaja, más potencia se obtiene y más acoplamiento se asume. Para el caso de este artículo (un campo adicional en el asiento contable que el propio estándar declara como extensible) el pilar correcto es el primero, porque la aplicación F2548 documenta oficialmente su punto de extensión de key user y no hay lógica compleja involucrada. El rol protagonista es por tanto el key user, aunque, como se verá en las secciones de auditoría y verificación, el desarrollador ABAP participa después comprobando en el SAP GUI y en ADT lo que el framework generó.

## 3. La ficha técnica de la aplicación en la SAP Fiori Apps Reference Library

Toda extensión de una app estándar empieza por su ficha técnica, igual que toda corrección de un programa empieza por leer su documentación y su código. La fuente única de esa ficha es la SAP Fiori Apps Reference Library, y conviene conocer su situación actual porque la herramienta está en plena transición.

### 3.1. La nueva biblioteca de referencia

Hoy, 10 de julio de 2026, la SAP Fiori Apps Reference Library presenta un aspecto completamente renovado. La biblioteca clásica, disponible en `https://fioriappslibrary.hana.ondemand.com/`, tiene fecha de retirada: SAP la descontinuará en diciembre de 2026. Su sustituta es la nueva SAP Fiori Apps Reference Library, publicada en `https://fal.cloud.sap`, que corre sobre la infraestructura de SAP Cloud ALM (por eso el navegador termina mostrando una URL del dominio `pr.alm.me.sap.com` con el launchpad y el intent `FALApp-display`). A diferencia de la clásica, la nueva biblioteca exige autenticarse con un SAP Universal ID, S-User o P-User. La **Figura 2** muestra la nueva experiencia con la ficha del F2548 abierta.

![Figura 2. Nueva SAP Fiori Apps Reference Library con la ficha de Upload General Journal Entries.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/03%20-%20Custom%20Fields%20con%20el%20Business%20Context%20Coding%20Block%20en%20SAP%20S4HANA%202025/figura-2.png)

*Figura 2. Nueva SAP Fiori Apps Reference Library con la ficha de Upload General Journal Entries.*

En la captura se aprecian los filtros aplicados: suite SAP S/4HANA (Private Cloud and On-Premise) y versión 2025 On-Premise, exactamente el sistema del ejercicio. El panel derecho abre la pestaña SAP Fiori Launchpad con la configuración de la app, donde consta el componente SAPUI5 `fin.gl.journalentry.upload` y el componente de aplicación `FI-FIO-GL-TRA`. Si en cambio se accede por la URL clásica, la biblioteca recibe al visitante con el aviso de transición que muestra la **Figura 3**.

![Figura 3. Biblioteca clásica con el aviso de retirada prevista para diciembre de 2026.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/03%20-%20Custom%20Fields%20con%20el%20Business%20Context%20Coding%20Block%20en%20SAP%20S4HANA%202025/figura-3.png)

*Figura 3. Biblioteca clásica con el aviso de retirada prevista para diciembre de 2026.*

### 3.2. Información técnica del F2548

De la ficha del F2548 en la biblioteca se extraen los datos técnicos que gobiernan el resto del ejercicio. En Target Mappings, el objeto semántico es `AccountingDocument` con la acción `uploadGLDocument`, el intent con el que el launchpad resuelve la navegación hacia la app. En Configuration constan el componente SAPUI5 `fin.gl.journalentry.upload`, la aplicación BSP `FIN_GLJE_UPLD` y el nodo ICF `/sap/bc/ui5_ui5/sap/fin_glje_upld`.

La sección de servicios OData es la más valiosa para la extensibilidad, porque enumera los servicios que la app consume y, por tanto, los candidatos naturales a recibir el campo custom:

- `FAC_GL_DOCUMENT_POST_SRV`, el servicio de posteo de asientos, asociado al rol `SAP_FIN_GLDOCPOST_APP`.
- `FAC_FINANCIAL_DOCUMENT_SRV_01`, el servicio de gestión y visualización de asientos (el mismo que usa Manage Journal Entries), asociado al rol `SAP_FIN_GLDOCMANAGE_APP`.
- `CA_OC_OUTPUT_REQUEST_SRV`, `CV_ATTACHMENT_SRV` y `FI_CORRESPONDENCE_SRV`, servicios transversales de output, adjuntos y correspondencia.

Esta lista es la primera pieza del método jerárquico de decisión que se desarrolla en la sección 7.3: los servicios que la propia ficha declara son los que la app necesita, y por tanto los primeros que deben recibir el campo.

### 3.3. La sección App Extensibility

La ficha técnica remite a la documentación de SAP Help Portal, donde cada app extensible tiene una página llamada App Extensibility. Para Upload General Journal Entries, esa página declara sin ambigüedad el punto de extensión, como muestra la **Figura 4**.

![Figura 4. Página App Extensibility del F2548 con el Business Context Coding Block declarado.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/03%20-%20Custom%20Fields%20con%20el%20Business%20Context%20Coding%20Block%20en%20SAP%20S4HANA%202025/figura-4.png)

*Figura 4. Página App Extensibility del F2548 con el Business Context Coding Block declarado.*

El contenido es breve pero decisivo: los key users pueden extender la app mediante la aplicación Custom Fields and Logic usando el Business Context **Coding Block**, y los campos añadidos aparecen en el elemento de UI Additional fields mediante adaptación de la interfaz en tiempo de ejecución. Esa única tabla de la documentación responde de antemano la pregunta que suele consumir horas de prueba y error: no hay que adivinar el Business Context correcto, la página App Extensibility de cada app lo declara. Este es siempre el primer lugar donde mirar. Con el contexto identificado sobre el papel, el siguiente paso es encontrarlo dentro del sistema, y ahí es donde entra en juego la primera pregunta de la introducción.

## 4. El Business Context y el registro de extensibilidad

¿Qué es exactamente un Business Context y dónde vive dentro del sistema? Esta sección responde la primera pregunta de la apertura con el concepto y con su evidencia física: el registro de extensibilidad.

### 4.1. Qué es un Business Context

Un Business Context es la vista de extensibilidad de un objeto de negocio: define qué entidad se puede extender (el coding block contable, la partida del asiento, el material, el business partner), sobre qué include de base de datos se generan los campos y qué artefactos de consumo (interfaces de usuario, vistas CDS, APIs, formularios) pueden recibirlos. Este concepto es comparable con el contrato de una interfaz en desarrollo de software: igual que una clase solo expone a sus consumidores los métodos que su interfaz declara, un objeto estándar solo expone a la extensibilidad de key user las superficies que su Business Context registra. Todo lo que está dentro del contrato sobrevive a los upgrades; todo lo que se haga por fuera es responsabilidad del cliente.

El coding block merece una aclaración para quien viene del mundo SAPUI5 y no del financiero: es la estructura de imputación contable (`COBL`) que comparten todos los procesos que terminan generando asientos, desde una factura de proveedor hasta una salida de mercancías. Extender el coding block es como extender una superclase de la que heredan decenas de procesos: el campo aparece en todos los documentos contables a la vez, lo cual es exactamente lo que se busca cuando la dimensión nueva debe acompañar a cada partida.

### 4.2. La transacción SCFD_REGISTRY

Los Business Contexts los entrega SAP con el producto: el cliente no crea contextos nuevos, sino campos dentro de los contextos existentes. El catálogo completo vive en el **Extensibility Registry**, accesible con la transacción `SCFD_REGISTRY`, cuya pantalla inicial muestra la **Figura 5**.

![Figura 5. Extensibility Registry de la transacción `SCFD_REGISTRY` con el catálogo de Business Contexts.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/03%20-%20Custom%20Fields%20con%20el%20Business%20Context%20Coding%20Block%20en%20SAP%20S4HANA%202025/figura-5.png)

*Figura 5. Extensibility Registry de la transacción `SCFD_REGISTRY` con el catálogo de Business Contexts.*

El registro es mucho más que una lista: sus pestañas superiores (Business Context, Extensible CDS View, Extensible Abstract Entity, Extensible OData Model, BAdI Registration, Data Transfer, Business Scenario, SAP GUI Context, Extensible Function Module, Extensible SOAP Service) catalogan cada tipo de artefacto que participa del framework de extensibilidad. En la pestaña Business Context, cada fila declara el identificador del contexto, su descripción, si es relevante para mejora estructural o de lógica, el include de persistencia, el **sufijo** de tres letras y el include de campos transitorios. El botón Where Used permite además rastrear dónde se usa cada contexto. Localizar el contexto del ejercicio es inmediato con la búsqueda estándar del SAP GUI: la **Figura 6** muestra la fila `FINS_CODING_BLOCK` encontrada con Ctrl+F.

![Figura 6. Fila `FINS_CODING_BLOCK` del registro con su include `INCL_EEW_COBL` y sufijo COB.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/03%20-%20Custom%20Fields%20con%20el%20Business%20Context%20Coding%20Block%20en%20SAP%20S4HANA%202025/figura-6.png)

*Figura 6. Fila `FINS_CODING_BLOCK` del registro con su include `INCL_EEW_COBL` y sufijo COB.*

La fila resaltada condensa en una línea lo que el resto del artículo desarrollará: contexto `FINS_CODING_BLOCK`, descripción Accounting: Coding Block, relevante para mejora estructural y de lógica, include de persistencia `INCL_EEW_COBL`, sufijo `COB` e include transitorio `INCL_EEW_COBL_TR`. La misma pantalla revela de paso los contextos vecinos del mundo financiero, con sus propios includes y sufijos: `FINS_JOURNAL_ENTRY_ITEM` con `INCL_EEW_ACDOC` y sufijo `JEI`, o `FINS_MARKET_SEGMENT` con `INCL_EEW_MARKET_SEGMENT_PS` y sufijo `MSE`.

### 4.3. El detalle del contexto y los includes INCL_EEW_COBL

Al hacer doble clic sobre la fila, el registro abre la vista de detalle del contexto, que la **Figura 7** muestra completa y que funciona como la ficha de identidad técnica del Coding Block.

![Figura 7. Detalle del contexto `FINS_CODING_BLOCK` con contrato C0, sufijo COB, includes y cuotas.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/03%20-%20Custom%20Fields%20con%20el%20Business%20Context%20Coding%20Block%20en%20SAP%20S4HANA%202025/figura-7.png)

*Figura 7. Detalle del contexto `FINS_CODING_BLOCK` con contrato C0, sufijo COB, includes y cuotas.*

La pantalla merece leerse bloque a bloque porque cada campo responde una duda práctica:

- **Contract C0 (Add Custom Fields via Key User App)**: el Release State es `Released` y la Availability es `Available in Cloud and OnPremise`. El contrato C0 es la formalización de lo que coloquialmente se llama "el contexto está habilitado": SAP libera el contexto para que la app de key user pueda crear campos sobre él. Un contexto sin release C0 existe en el registro pero no aparece en el desplegable de la app Custom Fields.
- **Field Suffix**: el valor `COB` explica de una vez el misterio del nombre del campo. El sufijo no lo inventa la app ni lo elige el usuario: está declarado aquí, en el registro, y el framework lo añade a cada campo del contexto.
- **Los tres includes**: el Persistence Include `INCL_EEW_COBL` es la estructura donde se generan físicamente los campos custom; al estar incluida dentro de la estructura de imputación `COBL`, todo campo generado ahí viaja automáticamente a las tablas que incorporan el coding block, `BSEG` y `ACDOCA` entre ellas. El Transient Fields Include `INCL_EEW_COBL_TR` alberga la variante transitoria de los campos, la que se usa en estructuras de proceso donde el valor se transporta en memoria sin persistirse. El Field Control Include `INCL_EEW_COBL_FC` contiene los campos de control que gobiernan la visibilidad y editabilidad campo a campo. La familia `INCL_EEW_*` (Enterprise Extension Workbench) es la infraestructura moderna que sustituye, para la extensibilidad gestionada, al include clásico `CI_COBL` que se editaba a mano en el mundo ECC.
- **Cloud Quota Definition**: el contexto admite un máximo de 20 campos con una capacidad total de 400 caracteres unicode. Estas cuotas son las que el key user consume con cada campo nuevo; el campo del ejercicio, de 10 caracteres, deja 390 disponibles.
- **Field Deletion Disclaimer**: el mensaje T100 `FINS_ACDOC_DB 010` ("Make sure that this field is not used in external reporting") es el aviso que el sistema mostrará ante un intento de borrado, un recordatorio de que un campo del coding block puede estar referenciado por reporting externo.

Con el contrato leído directamente del registro, el terreno está preparado para crear el campo.

## 5. La creación del campo custom

La teoría del contrato se convierte ahora en práctica: crear el campo, resolver los dos bloqueos típicos del diálogo de creación y conocer la herramienta que cubre el caso de los campos que ya existen en las tablas.

### 5.1. La app Custom Fields y el diálogo New Field

La creación se realiza desde el Fiori launchpad. En el sistema del ejercicio, las apps de extensibilidad están agrupadas en el space **Key User Extensibility**, como muestra la **Figura 8**, donde el tile Custom Fields convive con Custom Business Objects, Custom Logic, Custom CDS Views y las apps de transporte de extensiones.

![Figura 8. Space Key User Extensibility del launchpad con el tile Custom Fields resaltado.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/03%20-%20Custom%20Fields%20con%20el%20Business%20Context%20Coding%20Block%20en%20SAP%20S4HANA%202025/figura-8.png)

*Figura 8. Space Key User Extensibility del launchpad con el tile Custom Fields resaltado.*

Para acceder a estas apps el usuario necesita el catálogo de negocio de extensibilidad (`SAP_CORE_BC_EXT`). Ahora, dentro de la app, el botón de creación abre el diálogo New Field que muestra la **Figura 9**, el punto donde se toman las tres decisiones que definen el campo: contexto, identificador y tipo.

![Figura 9. Diálogo New Field con el contexto Coding Block, el identificador ESU y el tipo Text.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/03%20-%20Custom%20Fields%20con%20el%20Business%20Context%20Coding%20Block%20en%20SAP%20S4HANA%202025/figura-9.png)

*Figura 9. Diálogo New Field con el contexto Coding Block, el identificador ESU y el tipo Text.*

Los valores del ejercicio son los siguientes: Business Context `Accounting: Coding Block (FINS_CODING_BLOCK)`, etiqueta `Strategic Unit`, identificador `ESU` (que la app compone automáticamente como `ZZ1_ESU`, con el prefijo fijo visible en gris), tipo `Text` y longitud `10`. El propio diálogo muestra un aviso que anticipa la sección 5.3: existen restricciones definidas para este contexto, de modo que algunos tipos de campo, Business Objects o vistas CDS pueden no estar disponibles, con remisión a la SAP Note 2453614, la FAQ oficial de extensibilidad del Universal Journal.

### 5.2. Qué hacer si el Business Context no aparece en el listado

La primera pregunta habitual surge cuando el desplegable de Business Context no ofrece el contexto esperado. El diagnóstico inmediato sigue cuatro comprobaciones ordenadas de la más frecuente a la más rara:

1. **Confirmar que la app es extensible por ese contexto**: la página App Extensibility de la aplicación en SAP Help Portal (sección 3.3) declara qué contexto usa cada app. Si la app no tiene página App Extensibility, lo más probable es que no esté habilitada para extensión de key user; antes de darlo por cerrado, conviene buscar en los blogs oficiales de SAP y en SAP Community, donde a menudo se documentan escenarios de extensión que la ayuda formal todavía no recoge.
2. **Consultar el registro de extensibilidad**: la transacción `SCFD_REGISTRY` muestra si el contexto existe y, en su detalle, si el contrato C0 está en Release State `Released` (sección 4.3). Un contexto sin ese release no aparece en la app aunque exista en el registro.
3. **Revisar autorizaciones**: sin el catálogo `SAP_CORE_BC_EXT` asignado vía rol de negocio, la app Custom Fields no muestra contextos o directamente no abre.
4. **Revisar los casos conocidos**: hay escenarios documentados de campos que dejan de ser visibles en la app pero siguen existiendo en el backend (KBA 3589387), verificables con la transacción `SCFD_FIELDVIEWER` de la sección 9.

Ahora bien, cuando las cuatro comprobaciones confirman que el contexto sencillamente no existe para el objeto que se quiere extender, la pregunta cambia de naturaleza: ya no es un problema de configuración sino una decisión de arquitectura. ¿Corresponde saltar a Developer Extensibility o retroceder a la extensibilidad clásica? La respuesta honesta es que no hay un ganador universal, hay un orden de evaluación:

1. **Verificar primero Developer Extensibility**: la ausencia de contrato de key user (C0) no implica ausencia de contrato de desarrollador. Son contratos independientes, y en la práctica se dan todas las combinaciones: objetos con ambos, objetos con key user pero sin interfaz liberada para desarrollo, y objetos sin key user pero con Business Object Interface liberada para extensión. La verificación se hace en el **SAP Business Accelerator Hub** (`hub.sap.com`, que redirige a `api.sap.com`), donde cada Business Object Interface documenta si admite Developer Extensibility y con qué contrato de estabilidad; y dentro del sistema, en ADT, con el árbol Released Objects y el estado de release del objeto (contrato C1 para uso y extensión desde ABAP Cloud). Si la interfaz está liberada, el camino correcto es la extensión de desarrollador: nodos y campos vía extensión del BO, con las garantías de upgrade del contrato.
2. **La extensibilidad clásica como último recurso deliberado**: en un sistema on-premise el arsenal clásico sigue disponible (appends, includes de cliente como `CI_COBL`, BAdIs, exits), y para el coding block en concreto la KBA 2143232 documenta el procedimiento clásico sin modificación. Es una opción legítima cuando ni el key user ni el desarrollador tienen contrato liberado y el requisito no admite espera, pero debe asumirse con los ojos abiertos: no hay contrato de estabilidad, cada upgrade exige regresión manual, y el campo queda fuera de la gobernanza del framework (no aparecerá en `SCFD_FIELDVIEWER`, no se habilita por pestañas, no se audita con Where-Used).
3. **Registrar la decisión y no mezclar mecanismos**: el error más caro no es elegir clásico, es extender el mismo objeto por dos vías a la vez. Un campo del coding block creado por `CI_COBL` clásico y otro por `INCL_EEW_COBL` gestionado conviven técnicamente, pero duplican la superficie de mantenimiento y complican operaciones tan delicadas como el borrado de campos (KBA 3456968). La recomendación profesional es evaluar en el orden key user, desarrollador, clásico, documentar por qué se descartó cada nivel superior, y mantener cada objeto extendido por una única vía.

### 5.3. Qué hacer si el tipo de dato no aparece en el listado

La segunda pregunta habitual aparece al abrir el desplegable Type: faltan tipos que la app sí ofrece en otros contextos (importes, cantidades, fechas, asociaciones a Business Objects). La causa no es un error sino el contrato del contexto: cada Business Context restringe los tipos de dato según la naturaleza de las tablas que extiende. El coding block, al propagarse a estructuras compartidas por todo el flujo contable, es de los más restrictivos, como resume la siguiente comparación con el otro contexto financiero disponible para el asiento.

| Criterio | Accounting: Coding Block | Journal Entry Item |
|---|---|---|
| Tipos de dato | Solo texto y numérico (CHAR y NUMC) | Todos los tipos del framework |
| Número máximo de campos | 20 campos (cuota del registro) | Hasta 120 campos |
| Capacidad total | 400 caracteres unicode (cuota del registro) | Sin cuota tan estricta |
| Include de persistencia | `INCL_EEW_COBL` dentro de `COBL` | `INCL_EEW_ACDOC` |
| Alcance de la propagación | Todo el flujo contable: entrada de documentos, `COBL`, `BSEG`, `ACDOCA` | La partida del asiento: `ACCIT`, `ACDOCA` y `ACDOCP` |
| Sufijo del campo | `_COB` | `_JEI` |
| Uso típico | Dimensión de imputación que el usuario captura al contabilizar | Atributo derivado o informativo de la partida |

La conclusión práctica de la tabla es una regla de decisión: si el campo debe capturarse en la entrada del documento y viajar por todo el flujo contable, corresponde Coding Block y hay que aceptar sus límites de tipo; si el campo es un atributo de la partida que se deriva o se reporta, corresponde Journal Entry Item, que admite cualquier tipo. Cuando ninguno de los dos contratos alcanza (por ejemplo, un campo con lógica de validación compleja contra datos maestros propios), la petición ha superado la extensibilidad de key user y debe evaluarse con el orden de la sección 5.2. En el ejercicio, el tipo `Text` de longitud 10 encaja sin fricción en las restricciones del coding block.

### 5.4. La transacción SCFD_EUI y los campos ya existentes

Queda un tercer escenario que no es ni contexto ausente ni tipo ausente: el campo **ya existe** en la tabla, típicamente un campo Z creado años atrás por un append clásico, y lo que se quiere es incorporarlo a la gobernanza del framework para habilitarlo en interfaces y APIs como si fuera un campo de key user. Para eso existe la transacción `SCFD_EUI`, cuya pantalla de selección muestra la **Figura 10**.

![Figura 10. Pantalla de selección de `SCFD_EUI` con filtros de tabla y campo de base de datos.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/03%20-%20Custom%20Fields%20con%20el%20Business%20Context%20Coding%20Block%20en%20SAP%20S4HANA%202025/figura-10.png)

*Figura 10. Pantalla de selección de `SCFD_EUI` con filtros de tabla y campo de base de datos.*

El título de la pantalla lo dice todo: Enable Fields for Use in App Custom Fields and Logic. Los filtros permiten acotar por nombre de tabla y de campo (en la captura, tablas excluyendo `Z*` y campos `ZZ*`), limitar el número de resultados y, mediante el checkbox Allow Z* fields in namespace, incluir campos del namespace clásico de cliente. Tras ejecutar, el reporte lista los campos con su estado; los que muestran Can Be Enabled se seleccionan y se habilitan, quedando desde ese momento visibles en la app Custom Fields para gestionar sus usos por pestañas.

Conviene responder con precisión la pregunta de si esta herramienta es mejor práctica o extensibilidad clásica, porque es ambas cosas a medias y ninguna del todo: `SCFD_EUI` **pertenece al framework moderno**, no al clásico, pero su razón de ser es el brownfield. Es el puente oficial para que los campos nacidos en la era clásica (o campos estándar que el contexto no incorporó) entren en la gobernanza del framework de key user. La mejor práctica queda entonces así: para campos nuevos, siempre la app Custom Fields (sección 5.1); para campos heredados que ya viven en las tablas, `SCFD_EUI` una sola vez para adoptarlos; y nunca usar `SCFD_EUI` como excusa para seguir creando campos por append clásico y adoptarlos después, porque ese circuito duplica pasos y renuncia a las validaciones de cuota y tipo que el diálogo de creación aplica de origen.

## 6. Anatomía del campo generado

Con el campo creado conviene detenerse en su anatomía antes de habilitar usos, porque los nombres que el framework genera son la clave para localizarlo después en tablas, vistas y plantillas. Aquí se responde la segunda pregunta de la introducción: por qué el campo se llama `ZZ1_ESU_COB` sin que nadie escribiera ese nombre.

### 6.1. El prefijo ZZ1_ y el sufijo por contexto

Todo campo de key user nace en el namespace `ZZ1_`, un espacio reservado por SAP para la extensibilidad de cliente, de forma similar a como el namespace `Z*` protege los desarrollos clásicos de colisiones con el estándar en cada upgrade. El prefijo no se elige: la app lo muestra fijo en el diálogo de creación (Figura 9) y el key user solo aporta el identificador lógico, `ESU` en este caso, resultando el nombre `ZZ1_ESU` que actúa como elemento de datos y dominio.

El sufijo, como quedó demostrado en la sección 4.3, tampoco se adivina ni se elige: es el Field Suffix declarado en el registro de extensibilidad para cada Business Context. Para Accounting: Coding Block el registro declara `COB`, de modo que el componente de campo materializado es **`ZZ1_ESU_COB`**; el vecino Journal Entry Item declara `JEI` y el de segmentos de mercado `MSE`. El sufijo cumple una función de trazabilidad inmediata: ante cualquier campo `ZZ1_*` encontrado en una tabla o vista, sus tres últimas letras identifican el contexto que lo gobierna. Dentro de la propia app, el nombre completo se confirma en la pestaña General Information (tabla UI Texts, columna Field Component Name) y en la columna Field Name de la pestaña SOAP APIs.

### 6.2. La propagación hacia ACDOCA

La segunda mitad de la anatomía es entender dónde aterriza físicamente el campo. La pregunta de cómo saber que el contexto Coding Block toca la tabla `ACDOCA` tiene ahora una respuesta directa leída del registro: el Persistence Include del contexto es `INCL_EEW_COBL`, ese include forma parte de la estructura de imputación `COBL`, y `COBL` está incorporada en las tablas del flujo contable, `BSEG` y `ACDOCA` incluidas. El mecanismo es heredero directo de la extensión clásica del coding block (transacción `OXK3` sobre el include `CI_COBL`), con la diferencia de que ahora el ciclo de vida completo lo gobierna el framework de extensibilidad y no un desarrollador editando includes. El flujo completo, desde el diálogo de creación hasta la plantilla de Excel, se sintetiza en el diagrama de la **Figura 11**.

![Figura 11. Diagrama de propagación del campo ZZ1_ESU desde la app hasta el F2548.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/03%20-%20Custom%20Fields%20con%20el%20Business%20Context%20Coding%20Block%20en%20SAP%20S4HANA%202025/figura-11.png)

*Figura 11. Diagrama de propagación del campo ZZ1_ESU desde la app hasta el F2548.*

El diagrama se lee de arriba hacia abajo: la app Custom Fields registra el campo en el Business Context; al publicar, el framework genera el componente `ZZ1_ESU_COB` como append del include `INCL_EEW_COBL`, con lo que el campo entra en `COBL`, `BSEG` y `ACDOCA`; y desde ahí queda disponible para las cuatro familias de consumo que el key user habilita selectivamente: vistas CDS, servicios OData de las interfaces de usuario, APIs OData de integración y servicios SOAP de posteo entrante. La última caja cierra el círculo del ejercicio: la plantilla del F2548 incorpora la columna con el nombre técnico completo del campo.

### 6.3. Cabecera frente a posición: el alcance del coding block

Una duda que aparece en cuanto se entiende el mecanismo es si el campo recién creado vive en la cabecera del asiento o en cada una de sus posiciones. La respuesta es inequívoca y se deduce de todo lo anterior: el coding block añade campos a nivel de posición, nunca a nivel de cabecera. La razón está en su naturaleza contable. El coding block (`COBL`) es el bloque de imputación que acompaña a cada línea del asiento, porque cada partida puede imputarse a un centro de coste, un centro de beneficio o una unidad estratégica distinta de las demás líneas del mismo documento. Las estructuras donde el campo aterriza lo confirman, como se vio en la Figura 25 (la tabla `ACDOCA`) y en las vistas de partida de las Figuras 26 a 30: `BSEG` y `ACDOCA` son tablas de partidas individuales, no de cabecera. Por eso un campo sobre el contexto Accounting: Coding Block se materializa una vez por cada posición del asiento y nunca en la cabecera `BKPF` del documento.

La consecuencia práctica surge cuando el requisito es distinto: un campo de cabecera, un dato único por documento y no por partida (una referencia global del asiento, una clasificación del documento completo). La cabecera del asiento es la tabla `BKPF`, y para extenderla hace falta un Business Context diferente cuyo include de persistencia aterrice en la estructura de cabecera y no en `COBL`. Aquí la página App Extensibility de cada app vuelve a ser la autoridad: para el F2548, esa página (Figura 4) declara un único punto de extensión de key user, Additional fields sobre el contexto Coding Block, que es de posición. La app no expone un punto de extensión de cabecera, de modo que a través de su extensibilidad de key user solo se pueden añadir campos de partida. Un campo de cabecera para el asiento contable exige, por tanto, aplicar el orden de la sección 5.2: comprobar si existe un contexto de cabecera liberado para el objeto y, si no, escalar a Developer Extensibility.

Averiguar el nivel de cualquier contexto, sin depender de la memoria ni de suposiciones, se apoya en tres verificaciones ordenadas:

1. **El include de persistencia en `SCFD_REGISTRY`**: es el indicador más directo. El detalle del contexto (sección 4.3) muestra el Persistence Include; si ese include forma parte de una estructura de posición como `COBL` o `ACDOCA`, el contexto es de posición; si forma parte de la estructura de cabecera del documento, es de cabecera. Para `FINS_CODING_BLOCK` el include es `INCL_EEW_COBL`, inequívocamente de posición.
2. **La pestaña Extensible CDS View del registro**: revela qué vistas del modelo de datos toca el contexto. Un contexto de posición extiende vistas de partidas como `I_JournalEntryItem` o `I_JournalEntryItemCube`; un contexto de cabecera extendería vistas de documento. La naturaleza de las vistas confirma el nivel.
3. **La entidad del servicio OData en la ficha de la app**: en el modelo OData, un campo de posición aparece en la entidad de partidas y un campo de cabecera en la entidad de documento. Comprobar en qué entidad aparece el campo, o en cuál se querría que apareciera, delata el nivel que hace falta.

En la práctica, la regla mnemotécnica es sencilla: si el dato varía de una línea a otra del asiento, es de posición y el coding block lo cubre; si el dato es único para todo el documento, es de cabecera y hay que buscar su propio contexto o subir de pilar de extensibilidad.

## 7. Las pestañas de habilitación de usos

Crear el campo lo registra; habilitarlo lo hace útil. Aquí se responde la cuarta pregunta de la introducción: cómo decidir, entre centenares de artefactos, cuáles activar. Entender qué hay detrás de cada pestaña evita los dos errores opuestos: habilitar de menos (el campo no aparece donde se necesita) y habilitar de más (el contrato de extensión crece sin control y cada superficie extra es una dependencia más a vigilar en los upgrades).

### 7.1. El significado de cada pestaña

En términos generales, la ficha de un campo custom se organiza en pestañas que representan las superficies de consumo que su Business Context registra, y el contador que acompaña a cada pestaña indica cuántos artefactos de esa categoría ofrece el contexto, no cuántos están habilitados. Las pestañas concretas y sus contadores varían por contexto: un contexto de datos maestros ofrecerá formularios y escenarios distintos de los de un contexto contable. Lo que no varía es la semántica de cada categoría, que se describe a continuación, ilustrada con lo que el contexto Coding Block ofrece en el sistema del ejercicio.

**General Information** contiene los datos maestros del campo: longitud, textos de UI (etiqueta, tooltip, encabezado, etiquetas media y corta), datos administrativos y el Field Component Name definitivo. **User Interfaces** agrupa los artefactos que alimentan pantallas, con tres subcategorías visibles como filtros: Services (los servicios OData que consumen las apps Fiori), CDS Views (las vistas del Virtual Data Model que alimentan listados y apps analíticas) y SAP GUI (las dynpros clásicas registradas en el contexto; para el Coding Block este filtro está vacío porque el contexto no registra pantallas clásicas habilitables desde aquí). Habilitar un uso en esta pestaña extiende el artefacto correspondiente para que el campo viaje hasta la interfaz; el paso final de mostrarlo lo hace el key user con la adaptación de UI en tiempo de ejecución.

**Analytics** habilita el campo en las consultas y cubos analíticos del embedded analytics, de modo que la dimensión custom pueda usarse en informes. **Operations** lista operaciones internas que el contexto expone al framework. **Email Templates** y **Form Templates** permiten que el campo llegue a las plantillas de correo y a los formularios de salida (por ejemplo, un formulario de correspondencia financiera que deba imprimir la dimensión nueva). **Business Scenarios** habilita la transferencia del campo entre contextos de procesos relacionados, para que la dimensión fluya de un documento origen a su documento siguiente. **BAPIs** e **IDocs** cubren las interfaces clásicas RFC y de mensajería EDI/ALE. **Where-Used**, por último, no habilita nada: audita en qué artefactos está activo el campo, y es la pestaña que se consulta antes de cualquier limpieza o análisis de impacto. En el escenario del ejercicio, el peso del trabajo cae sobre User Interfaces (con más de un centenar de artefactos registrados entre servicios y vistas CDS), las APIs OData, las SOAP APIs y Analytics; las demás pestañas quedan sin usos habilitados porque ninguna integración del proceso las requiere.

### 7.2. APIs, BAPIs y SOAP APIs

Tres pestañas generan la duda más repetida porque las tres suenan a "interfaces": APIs, BAPIs y SOAP APIs. La diferencia es el protocolo y la generación tecnológica de cada familia, y la tabla siguiente las contrasta.

| Pestaña | Tecnología | Dirección típica | Cuándo habilitar el campo |
|---|---|---|---|
| APIs | Servicios OData (REST) y vistas CDS de integración | Sistemas externos leen o escriben vía HTTP | Cuando una integración moderna (BTP, middleware, app side-by-side) debe leer o escribir el campo |
| SOAP APIs | Servicios web SOAP del Extension Model (mensajería XML asíncrona o síncrona) | Posteo entrante de documentos desde otros sistemas | Cuando los asientos llegan por interfaces SOAP (integraciones A2A clásicas, SAP Integration Suite) y deben traer el campo relleno |
| BAPIs | Interfaces RFC clásicas (Business Application Programming Interface) | Programas ABAP o integraciones RFC legadas | Cuando programas clásicos o interfaces RFC existentes (por ejemplo `BAPI_ACC_DOCUMENT_POST`) deben transportar el campo en sus tablas de extensión |

La pestaña IDocs completa la familia con la mensajería EDI/ALE clásica. La lectura arquitectónica de la tabla es generacional: BAPIs e IDocs representan la integración del mundo R/3, SOAP la generación de enterprise services de SAP ERP, y OData la generación actual alineada con Fiori y BTP. El campo custom puede habilitarse en cualquiera de las cuatro, pero cada habilitación debe justificarla una integración real, existente o planificada.

### 7.3. El método jerárquico para decidir qué habilitar

Con centenares de artefactos listados entre todas las pestañas, habilitar por intuición es una receta para el desorden. El método que se propone es jerárquico: se parte del proceso de negocio y se desciende por niveles, habilitando en cada nivel solo lo que el nivel anterior justifica. El principio rector es el mismo que aplica un buen desarrollador al diseñar una API: exponer lo mínimo necesario, porque cada superficie expuesta es un contrato que hay que mantener.

1. **Identificar las apps del proceso**: listar las aplicaciones Fiori donde el usuario debe ver o capturar el campo. En el ejercicio: Upload General Journal Entries (F2548) para la captura y Manage Journal Entries para la consulta posterior.
2. **Extraer sus servicios OData de la Fiori Apps Reference Library**: la ficha técnica de cada app (sección 3.2) enumera sus servicios. Esos nombres son los que hay que buscar en la pestaña User Interfaces, filtro Services. Para el F2548, el servicio de posteo es `FAC_GL_DOCUMENT_POST_SRV`; para la consulta, `FAC_FINANCIAL_DOCUMENT_SRV_01` y `UI_JOURNALENTRY_MANAGE`.
3. **Habilitar las vistas CDS que alimentan esas mismas apps**: algunas apps Fiori no leen de un servicio plano sino de vistas CDS transaccionales o analíticas. La vista `C_MANAGEJOURNALENTRYITEMTP` (la vista de partidas de Manage Journal Entries) es el ejemplo del ejercicio.
4. **Decidir la capa analítica**: si la dimensión debe aparecer en informes, se habilitan las vistas del Virtual Data Model financiero sobre las que se construye el reporting de partidas: `I_JournalEntryItem`, `I_JournalEntryItemCube`, `I_AccountingDocumentJournal` e `I_CompactJournal`. Si el campo es puramente operativo, este nivel se omite.
5. **Cubrir las reglas de negocio**: si el campo debe participar en validaciones y sustituciones financieras, se habilita la vista del framework de sustitución y validación, `FIN_SUBVAL_JOURNALENTRYITEM`.
6. **Habilitar integraciones solo con caso de uso confirmado**: APIs OData, SOAP APIs, BAPIs e IDocs se habilitan únicamente si existe una interfaz concreta que vaya a transportar el campo. Habilitarlas "por si acaso" infla el contrato sin beneficio.
7. **Auditar el resultado**: la pestaña Where-Used confirma la lista de superficies activas, y la transacción `SCFD_FIELDVIEWER` (sección 9) ofrece la misma información con mayor profundidad desde el backend.

Este método convierte una pantalla intimidante de centenares de filas en una decisión trazable: cada uso habilitado tiene detrás una app, un informe o una interfaz que lo justifica.

### 7.4. Los usos habilitados en el ejercicio

La aplicación del método al escenario del artículo se documenta en las capturas siguientes. En la pestaña User Interfaces, la primera habilitación es la vista transaccional de Manage Journal Entries, `C_MANAGEJOURNALENTRYITEMTP`, que muestra la **Figura 12** con estado Enabled.

![Figura 12. Pestaña User Interfaces con la vista `C_MANAGEJOURNALENTRYITEMTP` habilitada.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/03%20-%20Custom%20Fields%20con%20el%20Business%20Context%20Coding%20Block%20en%20SAP%20S4HANA%202025/figura-12.png)

*Figura 12. Pestaña User Interfaces con la vista `C_MANAGEJOURNALENTRYITEMTP` habilitada.*

La misma pestaña, más abajo, concentra los servicios OData del proceso. La **Figura 13** muestra habilitados `FAC_AUDIT_JOURNAL_SRV` (Audit Journal), `FAC_FINANCIAL_DOCUMENT_SRV_01` (Manage Journal Entries), `FAC_GL_DOCUMENT_POST_SRV` (Post General Journal Entries, el servicio del F2548), `FAC_GLV_GL_ACCOUNT_LINE_ITEMS_SRV` (Display Line Items in General Ledger) y la estructura de mapeo `CTE_FIN_SUBS_TARGET_ITEM`, necesaria para que el mapeo de campos de la carga reconozca la columna nueva.

![Figura 13. Servicios OData del flujo contable habilitados en la pestaña User Interfaces.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/03%20-%20Custom%20Fields%20con%20el%20Business%20Context%20Coding%20Block%20en%20SAP%20S4HANA%202025/figura-13.png)

*Figura 13. Servicios OData del flujo contable habilitados en la pestaña User Interfaces.*

El nivel analítico y de reglas de negocio se aprecia en la **Figura 14**: la vista de sustitución y validación `FIN_SUBVAL_JOURNALENTRYITEM` y los cubos `I_ACCOUNTINGDOCUMENTJOURNAL` e `I_COMPACTJOURNAL` figuran como Enabled entre las decenas de vistas que permanecen Disabled.

![Figura 14. Vistas `FIN_SUBVAL_JOURNALENTRYITEM`, `I_ACCOUNTINGDOCUMENTJOURNAL` e `I_COMPACTJOURNAL` habilitadas.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/03%20-%20Custom%20Fields%20con%20el%20Business%20Context%20Coding%20Block%20en%20SAP%20S4HANA%202025/figura-14.png)

*Figura 14. Vistas `FIN_SUBVAL_JOURNALENTRYITEM`, `I_ACCOUNTINGDOCUMENTJOURNAL` e `I_COMPACTJOURNAL` habilitadas.*

La **Figura 15** completa el nivel analítico con `I_JOURNALENTRYITEM` e `I_JOURNALENTRYITEMCUBE`, y muestra además el servicio `UI_JOURNALENTRY_MANAGE` habilitado junto con su entidad `C_MANAGEJOURNALENTRYITEMTPType`, la pareja servicio-entidad que consume la app de gestión de asientos.

![Figura 15. Cubos `I_JOURNALENTRYITEM` e `I_JOURNALENTRYITEMCUBE` y servicio `UI_JOURNALENTRY_MANAGE` habilitados.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/03%20-%20Custom%20Fields%20con%20el%20Business%20Context%20Coding%20Block%20en%20SAP%20S4HANA%202025/figura-15.png)

*Figura 15. Cubos `I_JOURNALENTRYITEM` e `I_JOURNALENTRYITEMCUBE` y servicio `UI_JOURNALENTRY_MANAGE` habilitados.*

En la pestaña APIs, la única habilitación es la API de lectura de partidas para integraciones, `API_JOURNALENTRYITEMBASIC_SRV` con su entidad `A_JournalEntryItemBasicType`, como muestra la **Figura 16**; el resto de servicios de la lista permanece deshabilitado siguiendo el principio de mínima exposición.

![Figura 16. Pestaña APIs con `API_JOURNALENTRYITEMBASIC_SRV` habilitada para integraciones de lectura.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/03%20-%20Custom%20Fields%20con%20el%20Business%20Context%20Coding%20Block%20en%20SAP%20S4HANA%202025/figura-16.png)

*Figura 16. Pestaña APIs con `API_JOURNALENTRYITEMBASIC_SRV` habilitada para integraciones de lectura.*

En la pestaña SOAP APIs se habilitan las interfaces de posteo entrante de asientos. La **Figura 17** muestra las operaciones masivas `JournalEntryBulkLedgerCreationRequest_In` y `JournalEntryBulkCreationRequest_In`, junto con la interfaz de contabilidad predictiva `PredictiveJournalEntryManageRequestConfirmation_In`, cada una con su namespace XML de custom fields y el nombre de campo `ZZ1_ESU` en la columna Field Name.

![Figura 17. Pestaña SOAP APIs con las interfaces masivas de creación de asientos habilitadas.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/03%20-%20Custom%20Fields%20con%20el%20Business%20Context%20Coding%20Block%20en%20SAP%20S4HANA%202025/figura-17.png)

*Figura 17. Pestaña SOAP APIs con las interfaces masivas de creación de asientos habilitadas.*

La **Figura 18** completa la familia SOAP con la interfaz síncrona `JournalEntryCreateRequestConfirmation_In`, el Extension Model de posteo individual, también en estado Enabled.

![Figura 18. Interfaz SOAP síncrona `JournalEntryCreateRequestConfirmation_In` habilitada.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/03%20-%20Custom%20Fields%20con%20el%20Business%20Context%20Coding%20Block%20en%20SAP%20S4HANA%202025/figura-18.png)

*Figura 18. Interfaz SOAP síncrona `JournalEntryCreateRequestConfirmation_In` habilitada.*

## 8. Publicación del campo

Con los usos decididos, el último paso dentro de la app es completar los textos y publicar. La pestaña General Information de la **Figura 19** muestra el estado final antes de la publicación: longitud 10, textos de UI completos (etiqueta, tooltip, encabezado y etiquetas media y corta) y, en la primera columna de la tabla UI Texts, el Field Component Name `ZZ1_ESU_COB` que la sección 6.1 anticipó.

![Figura 19. Pestaña General Information con el Field Component Name `ZZ1_ESU_COB` y los textos de UI.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/03%20-%20Custom%20Fields%20con%20el%20Business%20Context%20Coding%20Block%20en%20SAP%20S4HANA%202025/figura-19.png)

*Figura 19. Pestaña General Information con el Field Component Name `ZZ1_ESU_COB` y los textos de UI.*

La acción Publish dispara la generación: el framework crea el elemento de datos, el append sobre el include `INCL_EEW_COBL`, las extensiones de las vistas CDS habilitadas y las extensiones de los servicios. En sistemas on-premise la publicación pide además el paquete y la orden de transporte con los que la extensión viajará por el landscape. Al terminar, el listado de la app muestra el campo con estado Published, como confirma la **Figura 20**.

![Figura 20. Listado de Custom Fields con el campo Strategic Unit en estado Published.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/03%20-%20Custom%20Fields%20con%20el%20Business%20Context%20Coding%20Block%20en%20SAP%20S4HANA%202025/figura-20.png)

*Figura 20. Listado de Custom Fields con el campo Strategic Unit en estado Published.*

Si tras publicar el campo no aparece en el `$metadata` de los servicios habilitados, las transacciones de limpieza de caché del Gateway, `/IWFND/CACHE_CLEANUP` en el front-end y `/IWBEP/CACHE_CLEANUP` en el backend, resuelven la mayoría de los casos.

## 9. Auditoría con SCFD_FIELDVIEWER

La quinta pregunta de la introducción (¿existe un lugar único donde ver todo lo que el framework generó?) tiene respuesta afirmativa y se llama `SCFD_FIELDVIEWER`, el Field Viewer del backend, probablemente la transacción más útil y menos conocida de todo el framework de Custom Fields. La **Figura 21** muestra su pantalla principal con el campo del ejercicio seleccionado.

![Figura 21. Field Viewer de `SCFD_FIELDVIEWER` con la definición del campo `ZZ1_ESU`.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/03%20-%20Custom%20Fields%20con%20el%20Business%20Context%20Coding%20Block%20en%20SAP%20S4HANA%202025/figura-21.png)

*Figura 21. Field Viewer de `SCFD_FIELDVIEWER` con la definición del campo `ZZ1_ESU`.*

El panel izquierdo lista los campos custom del sistema con su versión, estado y origen (para el campo del ejercicio: Active, Published, Custom Field App). El panel derecho se organiza en pestañas, y la primera, Field Definition, concentra la ficha completa: tipo TEXT y longitud 10, agregación, usuario y fecha de creación y de último cambio, el Business Context al que pertenece y los textos de UI por idioma. La barra de herramientas superior anticipa la potencia de la transacción, con acciones administrativas que la app Fiori no ofrece: Check Fields, Publish Fields, Activate Enhancements, Display Error Log, Invalidate Runtime Buffer, Convert to ABAP Managed Field y Change Field Type, un arsenal para diagnosticar y reparar campos en estados inconsistentes sin abrir el launchpad.

La joya de la transacción es la pestaña Usages, que muestra la **Figura 22**: el árbol completo de artefactos donde el campo está en uso, con las vistas CDS arriba y los modelos OData abajo.

![Figura 22. Pestaña Usages con las CDS views y los modelos OData que transportan el campo.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/03%20-%20Custom%20Fields%20con%20el%20Business%20Context%20Coding%20Block%20en%20SAP%20S4HANA%202025/figura-22.png)

*Figura 22. Pestaña Usages con las CDS views y los modelos OData que transportan el campo.*

Esta vista responde en una sola pantalla preguntas que de otro modo exigirían rastrear vista por vista en ADT: qué CDS views llevan el campo (en el ejercicio, desde `A_JOURNALENTRYITEMBASIC` hasta `I_JOURNALENTRYITEMCUBE`, pasando por las cuatro variantes de ledger de Manage Journal Entries que la app habilitó en cascada), cuál es la vista subyacente de cada una, qué include de extensión la alimenta, y qué modelos OData exponen el campo con qué tipo de entidad (la columna API marca además cuáles son interfaces de integración, como `API_JOURNALENTRYITEMBASIC_MDL`). Para un administrador que hereda un sistema con decenas de campos custom, esta pestaña es la fotografía inmediata de impacto de cada campo: dónde buscar `ZZ1_ESU_COB` en todas las interfaces y qué aplicaciones lo tienen disponible.

La tercera pestaña, Enhancements, baja un nivel más y muestra el inventario de objetos técnicos generados, como se ve en la **Figura 23**.

![Figura 23. Pestaña Enhancements con el inventario de objetos generados por la publicación.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/03%20-%20Custom%20Fields%20con%20el%20Business%20Context%20Coding%20Block%20en%20SAP%20S4HANA%202025/figura-23.png)

*Figura 23. Pestaña Enhancements con el inventario de objetos generados por la publicación.*

El listado clasifica cada artefacto por tipo: las CDS View Extensions (una por vista habilitada, con nombres autogenerados del patrón `ZZ1_` seguido de una secuencia interna), las CDS View Include Extensions sobre los includes `E_*` del modelo financiero, el Data Element y el Domain `ZZ1_ESU`, el append de Field Control sobre `INCL_EEW_COBL_FC`, los Proxy Datatype Appends que extienden las estructuras de las interfaces SOAP y el Persistence Include Append sobre `INCL_EEW_COBL`. Este inventario es la prueba material de la afirmación de la sección 6.2: nada del estándar se modifica, todo se añade en capas que el framework controla.

La cuarta pestaña, Change Log, cierra la auditoría con la traza cronológica completa que muestra la **Figura 24**.

![Figura 24. Pestaña Change Log con la traza cronológica de creación y habilitaciones del campo.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/03%20-%20Custom%20Fields%20con%20el%20Business%20Context%20Coding%20Block%20en%20SAP%20S4HANA%202025/figura-24.png)

*Figura 24. Pestaña Change Log con la traza cronológica de creación y habilitaciones del campo.*

Cada acción quedó registrada con usuario y fecha: la creación del campo, los cambios de tipo y longitud, las etiquetas añadidas, la habilitación en el Business Context y, una a una, las habilitaciones en cada vista CDS, servicio SOAP (con su namespace y nombre de campo) y modelo OData. Para entornos con requisitos de auditoría, este log convierte al campo custom en un objeto plenamente trazable: quién lo creó, cuándo, y quién habilitó cada superficie. Las pestañas restantes (Where Used, FIORI UI Usages y Value Help) completan la visión con los usos cruzados, las interfaces Fiori concretas y las ayudas de búsqueda asociadas.

## 10. Verificación en el diccionario ABAP y en las CDS views

Tras la auditoría lógica del Field Viewer, la verificación física corresponde al desarrollador ABAP en ADT. La **Figura 25** muestra la tabla `ACDOCA` (Universal Journal Entry Line Items) en el sistema, con el campo `zz1_esu_cob` visible en la lista de componentes, de tipo `char(10)` y descripción Strategic Unit.

![Figura 25. Tabla `ACDOCA` en ADT con el campo `zz1_esu_cob` generado por la extensión.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/03%20-%20Custom%20Fields%20con%20el%20Business%20Context%20Coding%20Block%20en%20SAP%20S4HANA%202025/figura-25.png)

*Figura 25. Tabla `ACDOCA` en ADT con el campo `zz1_esu_cob` generado por la extensión.*

Dos detalles de la captura merecen lectura atenta. El primero es la pareja de campos dummy `acdoc_eew_dummy` y `dummy_incl_eew_cobl`, descritos como Custom Fields: Dummy for Use in Extension Includes; son los anclajes que el estándar reserva para los includes de extensión, la evidencia estructural de que `ACDOCA` participa del contrato del coding block. El segundo es la sección Extended with al pie del listado: el campo no está editado dentro de la tabla sino aportado por un append generado (con nombre técnico autogenerado y descripción `ZZ1_ESU`), exactamente el comportamiento upgrade-safe que distingue a la extensibilidad gestionada de la modificación clásica.

Las cuatro capturas siguientes confirman la propagación a las vistas CDS habilitadas en la sección 7.4. La **Figura 26** muestra el cubo analítico `I_AccountingDocumentJournal` con el elemento `ZZ1_ESU_COB` al final de su lista de campos.

![Figura 26. Vista `I_AccountingDocumentJournal` con el elemento `ZZ1_ESU_COB` propagado.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/03%20-%20Custom%20Fields%20con%20el%20Business%20Context%20Coding%20Block%20en%20SAP%20S4HANA%202025/figura-26.png)

*Figura 26. Vista `I_AccountingDocumentJournal` con el elemento `ZZ1_ESU_COB` propagado.*

La misma comprobación sobre la vista de partidas `I_JournalEntryItem` se aprecia en la **Figura 27**, donde el campo aparece tras los campos estándar de autorización y tipología de documento.

![Figura 27. Vista `I_JournalEntryItem` con el campo `ZZ1_ESU_COB` al final del listado.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/03%20-%20Custom%20Fields%20con%20el%20Business%20Context%20Coding%20Block%20en%20SAP%20S4HANA%202025/figura-27.png)

*Figura 27. Vista `I_JournalEntryItem` con el campo `ZZ1_ESU_COB` al final del listado.*

El cubo `I_JournalEntryItemCube`, base del reporting de partidas, exhibe el mismo elemento en la **Figura 28**; la captura muestra además el tooltip de ADT confirmando que la tabla `ACDOCA` está activa en el sistema S4HANA2025.

![Figura 28. Cubo `I_JournalEntryItemCube` con el campo `ZZ1_ESU_COB` propagado.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/03%20-%20Custom%20Fields%20con%20el%20Business%20Context%20Coding%20Block%20en%20SAP%20S4HANA%202025/figura-28.png)

*Figura 28. Cubo `I_JournalEntryItemCube` con el campo `ZZ1_ESU_COB` propagado.*

La **Figura 29** repite la verificación sobre `I_CompactJournal`, el diario compacto del modelo analítico financiero.

![Figura 29. Vista `I_CompactJournal` con el elemento `ZZ1_ESU_COB` visible en ADT.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/03%20-%20Custom%20Fields%20con%20el%20Business%20Context%20Coding%20Block%20en%20SAP%20S4HANA%202025/figura-29.png)

*Figura 29. Vista `I_CompactJournal` con el elemento `ZZ1_ESU_COB` visible en ADT.*

Por último, la **Figura 30** confirma el campo en `FIN_SUBVAL_JOURNALENTRYITEM`, la vista del framework de sustituciones y validaciones, lo que habilita a las reglas financieras a leer la dimensión nueva.

![Figura 30. Vista `FIN_SUBVAL_JOURNALENTRYITEM` con el campo `ZZ1_ESU_COB` habilitado.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/03%20-%20Custom%20Fields%20con%20el%20Business%20Context%20Coding%20Block%20en%20SAP%20S4HANA%202025/figura-30.png)

*Figura 30. Vista `FIN_SUBVAL_JOURNALENTRYITEM` con el campo `ZZ1_ESU_COB` habilitado.*

El patrón que se repite en las cinco vistas es deliberado: los nombres de las vistas verificadas coinciden con los artefactos habilitados en las pestañas de la app y con el árbol de la pestaña Usages del Field Viewer, cerrando el triángulo de evidencias que el diagrama de la Figura 11 anticipaba. Cada Enable Usage de la sección 7 tiene su contraparte comprobable tanto en `SCFD_FIELDVIEWER` como en ADT.

## 11. La plantilla Excel y el posteo del asiento

La última milla del ejercicio es funcional: usar el campo desde la app F2548, cuyo flujo consiste en descargar una plantilla de Excel, completarla con los asientos y subirla para el posteo.

### 11.1. La plantilla estándar

La plantilla que la app entrega con Download Template es un libro de Excel con una hoja Data organizada por lotes. Cada lote tiene una sección Header y una sección Line Items; en cada sección hay una fila de nombres técnicos y debajo una fila de descripciones, donde el asterisco marca los campos que la plantilla trae por defecto. La cabecera transporta los datos del documento (sociedad, clase de documento, fechas, periodo, moneda, grupo de ledgers) y las posiciones transportan la imputación completa del coding block: cuenta de mayor, importes, indicador de impuestos, centro de coste, centro de beneficio, orden, elemento PEP, banco propio, sociedad asociada y segmento, entre otros.

El detalle relevante para este artículo está al final de la fila de nombres técnicos de Line Items: la columna `ZZ1_ESU_COB`. La plantilla descargada tras la publicación ya incorpora el campo custom con su nombre de componente completo, sin etiqueta descriptiva debajo, porque la app compone la plantilla a partir de la estructura extendida del servicio de posteo.

### 11.2. El catálogo de columnas de la plantilla

El propio encabezado del libro documenta el mecanismo para añadir columnas: basta con agregar el nombre técnico del campo en la fila correspondiente. La ayuda de la aplicación publica el catálogo completo de nombres técnicos admitidos, que se reproduce a continuación como material de referencia; los campos marcados con asterisco doble son los que la plantilla trae por defecto y el número entre paréntesis indica la longitud máxima del valor.

Campos de cabecera:

| Nombre técnico | Campo |
|---|---|
| BELNR | Journal Entry (10) |
| BKTXT | **Document Header Text (25) |
| BLART | **Journal Entry Type (2) |
| BLDAT | **Journal Entry Date |
| BSTAT | Journal Entry Attribute (1) |
| BUDAT | **Posting Date |
| BUKRS | **Company Code (4) |
| STODT | Planned Reversal Date |
| KURSF_EXT | **Exchange Rate (12) |
| LDGRP | **Ledger Group (4) |
| MONAT | **Fiscal period (2) |
| PARGB_HDR | **Partner Business Area (4) |
| TXKRS_EXT | Exchange Rate for Taxes (12) |
| FULFILLDATE | Tax Fulfillment Date |
| VATDATE | Tax Reporting Date |
| WAERS | **Transaction Currency (5) |
| WWERT | **Currency Translation Date |
| XBLNR | **Reference Document Number (16) |
| XMWST | Calculate Tax Automatically (1) |
| XSNET | G/L Account Amounts Entered Exclude Tax (1) |
| STGRD | Reversal Reason |

Campos de posición (selección de los más utilizados del catálogo oficial, que supera los sesenta):

| Nombre técnico | Campo |
|---|---|
| BUKRS | **Company Code (4) |
| HKONT | **G/L Account (10) |
| SGTXT | Item Text (50) |
| WRSOL | **Debit |
| WRHAB | **Credit |
| DMBTR | **Amount in Company Code Currency |
| DMBE2 | **Amount in second local currency (LC2) |
| DMBE3 | **Amount in second local currency (LC3) |
| MWSKZ | **Tax Code (2) |
| TXJCD | **Tax Jurisdiction (15) |
| KOSTL | **Cost Center (10) |
| PRCTR | **Profit Center (10) |
| AUFNR | **Order Number (12) |
| PS_POSID | **WBS Element (24) |
| VALUT | **Value Date |
| HBKID | **House Bank (5) |
| HKTID | **House Bank Account (5) |
| ZUONR | **Assignment number (18) |
| VBUND | **Trading Partner (6) |
| SEGMENT | **Segment for Segmental Reporting (10) |
| RMVCT | Financial Transaction Type (3) |
| EBELN | Purchasing Document (10) |
| EBELP | Item Number of Purchasing Document (5) |
| KDAUF | Sales Order Number (10) |
| KDPOS | Sales Order Item (6) |
| MATNR_EXT | External Long Material Number (40) |
| MENGE | Quantity (13) |
| MEINS_EXT | OData: Base Unit of Measure (3) |
| WERKS | Plant (4) |
| GSBER | Business Area (4) |
| FKBER | Functional Area (16) |
| FISTL | Funds Center (16) |
| FIPEX | Commitment Item (24) |
| GEBER | Fund (10) |
| GRANT_NBR | Grant (20) |
| PERNR | Personnel Number (8) |
| NPLNR | Network Number for Account Assignment (12) |
| KSTRG | Cost Object (12) |
| PRZNR | Business Process (12) |
| PSEGMENT | Partner Segment for Segmental Reporting (10) |
| PPRCTR | Partner Profit Center (10) |
| XREF1 | Business Partner Reference Key (12) |
| XREF2 | Business Partner Reference Key (12) |
| XREF3 | Reference key for line item (20) |
| TAX_COUNTRY | Tax Reporting Country/Region (3) |

Dos convenciones adicionales completan el catálogo: los campos de segmento de rentabilidad CO-PA se añaden con el prefijo `PROF_` delante del nombre técnico (por ejemplo `PROF_KNDNR` para el cliente o `PROF_ARTNR` para el material), y los campos custom del coding block se añaden con su nombre de componente completo, sufijo incluido, como `ZZ1_ESU_COB`.

### 11.3. La plantilla con datos del asiento

La plantilla completada del ejercicio contiene un único lote con un asiento de prueba. La cabecera declara sociedad 1010, clase de documento SA, fechas de documento y contabilización 02.02.2026, periodo 2, moneda EUR y grupo de ledgers 0L. Las posiciones forman el par debe y haber mínimo de un asiento válido: dos líneas sobre la cuenta de mayor 11001000, la primera con 990.845 en la columna de debe y la segunda con el mismo importe en la de haber, ambas con indicador de impuestos X1 y banco propio DEBK1 con cuenta DEAC1.

La versión completada difiere de la estándar en dos aspectos que ilustran las convenciones de la sección anterior. El primero es la adición manual de columnas CO-PA con el prefijo `PROF_` (`PROF_KNDNR`, `PROF_ARTNR`, `PROF_KUNRE`) y del tipo de movimiento `RMVCT`. El segundo es el valor del campo custom: ambas posiciones llevan en la columna `ZZ1_ESU_COB` el texto `AS Prueba`, nueve caracteres que respetan la longitud 10 definida al crear el campo. Con esto, la dimensión estratégica viaja en el archivo con el mismo rango que cualquier campo de imputación estándar.

### 11.4. El resultado del posteo

Tras seleccionar el archivo y ejecutar la carga, la app procesa el lote, postea el asiento y confirma el resultado, como muestra la **Figura 31**.

![Figura 31. App F2548 con el asiento 100000000 posteado y el diálogo de confirmación.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/03%20-%20Custom%20Fields%20con%20el%20Business%20Context%20Coding%20Block%20en%20SAP%20S4HANA%202025/figura-31.png)

*Figura 31. App F2548 con el asiento 100000000 posteado y el diálogo de confirmación.*

La tabla de resultados registra el lote con Batch ID `$1G3C0GS46KR`, estado Posted, número de asiento 100000000 en la sociedad 1010 y fecha de contabilización 2 de febrero de 2026, mientras el diálogo informativo confirma que 1 de 1 asientos fue contabilizado. A partir de este momento, el valor `AS Prueba` reside en `ACDOCA` en el campo `zz1_esu_cob` de ambas partidas, consultable desde Manage Journal Entries, desde los informes analíticos y desde las integraciones habilitadas.

## 12. Lógica de validación con Custom Logic

Añadir el campo resuelve la captura del dato, pero muchos requisitos no terminan ahí. La unidad estratégica quizá deba ser obligatoria para ciertas cuentas, o no aceptar valores fuera de una lista aprobada, o derivarse automáticamente de otro dato del asiento. Esto ya no es extender la estructura sino extender el comportamiento, y la pregunta natural es si puede hacerse sin salir de la extensibilidad de key user. La respuesta es afirmativa, y hay dos caminos según cuánta lógica haga falta.

El primer camino, sin escribir código, son las apps de reglas del libro mayor Manage Substitution Rules y Manage Validation Rules, la sustituta moderna de las validaciones y sustituciones clásicas de la transacción `OB28` y del entorno `GGB0`. Estas apps permiten definir, mediante condiciones y prerrequisitos configurables, reglas del tipo "si la cuenta pertenece a este rango, la unidad estratégica es obligatoria". El dato clave para este artículo es por qué el campo custom está disponible en esas reglas: precisamente porque en la sección 7.4 se habilitó la vista `FIN_SUBVAL_JOURNALENTRYITEM`, la vista del framework de sustitución y validación de partidas del asiento. Sin esa habilitación, el campo existiría en la base de datos pero las reglas no podrían leerlo. Aquella casilla marcada en la pestaña User Interfaces cobra aquí todo su sentido, y es un ejemplo perfecto de por qué el método jerárquico de la sección 7.3 insiste en habilitar cada superficie con un propósito.

El segundo camino, cuando la lógica supera lo que una regla declarativa puede expresar (una validación contra datos maestros propios, un cálculo con varias condiciones anidadas, una llamada a otro objeto de negocio), es la app Custom Logic, el tile que aparece junto a Custom Fields en el space Key User Extensibility de la Figura 8. Custom Logic no es código libre: es la implementación de BAdIs (Business Add-Ins, los puntos de extensión que SAP libera para escenarios concretos) escrita en una variante restringida y preparada para la nube de ABAP. El flujo es reconocible para cualquier desarrollador acostumbrado a implementar interfaces: la app lista los BAdIs disponibles para cada escenario de negocio, se elige el que corresponde al momento del proceso que se quiere interceptar (por ejemplo, el instante previo a grabar el asiento) y se programa la validación dentro de la firma que el BAdI expone. El campo custom del coding block es visible dentro de esa firma porque forma parte de la estructura de la partida que el BAdI recibe, del mismo modo que un método heredado ve los atributos de su clase.

La siguiente tabla contrasta los dos caminos para elegir con criterio.

| Criterio | Reglas de sustitución y validación | Custom Logic (BAdI) |
|---|---|---|
| Cómo se define | Condiciones y prerrequisitos configurables, sin código | ABAP restringido y cloud-ready sobre un BAdI liberado |
| Quién lo ejecuta | Consultor funcional o key user | Desarrollador con perfil de key user extensibility |
| Cuándo elegirlo | Validaciones y derivaciones expresables como reglas | Lógica compleja: datos maestros propios, cálculos, condiciones anidadas |
| Requisito sobre el campo custom | Haber habilitado la vista `FIN_SUBVAL_JOURNALENTRYITEM` | Que el campo forme parte de la estructura que el BAdI expone |

La regla de decisión es la misma que gobierna todo el artículo, exponer lo mínimo y subir de nivel solo cuando el anterior no alcanza: se intenta primero con una regla declarativa y solo si el requisito la desborda se recurre a Custom Logic. Para averiguar qué opción cubre un requisito concreto conviene abrir la app Custom Logic y revisar el catálogo de BAdIs que ofrece para el escenario de asientos contables, del mismo modo que antes de crear el campo se consultó la página App Extensibility: la herramienta declara sus propios puntos de extensión, no hay que adivinarlos. En ambos caminos la lógica vive en la capa de extensibilidad gestionada, se transporta como cualquier otro objeto y sobrevive a los upgrades, exactamente igual que el campo que la origina.

## 13. Notas SAP de interés

Durante la preparación del ejercicio se recopilaron cinco notas y KBAs de SAP directamente relacionadas con el escenario, que conviene tener localizadas antes de llevarlo a un proyecto real:

- **SAP Note 2143232** (How To add a customer field to coding block without modification): el procedimiento clásico de extensión del coding block, útil como referencia histórica y como plan B documentado en la sección 5.2.
- **SAP Note 3456968** (Deletion of coding block fields from CI_COBL): las implicaciones de borrar campos del coding block clásico, el tipo de operación delicada que motiva la recomendación de no mezclar mecanismos.
- **KBA 3455828** (Edit standard template in Fiori App F2548): aclara cómo trabajar con la plantilla estándar de la app y sus límites de edición.
- **KBA 2540686** (You've changed the template. Restore the template or download it again): el error típico al modificar la estructura de la plantilla más allá de lo permitido.
- **KBA 2859804** (Upload General Journal Entries: dump when download template): un caso conocido de error en la descarga de la plantilla y su corrección.

## 14. Referencias

- [App Extensibility: Upload General Journal Entries, SAP Help Portal](https://help.sap.com/docs/SAP_S4HANA_ON-PREMISE/651d8af3ea974ad1a4d74449122c620e/d9fb8c4f76ef4277bad9b836fe86f2bb.html)
- [Introducing the new SAP Fiori Apps Reference Library, SAP Community](https://community.sap.com/t5/technology-blog-posts-by-sap/introducing-the-new-sap-fiori-apps-reference-library/ba-p/14346676)
- [SAP Fiori Apps Reference Library, versión clásica](https://fioriappslibrary.hana.ondemand.com/)
- [SAP Business Accelerator Hub](https://api.sap.com/)
- [SAP Note 2453614: FAQ de extensibilidad del Universal Journal](https://me.sap.com/notes/2453614)
- [KBA 3589387: Custom Fields del contexto Accounting: Coding Block no visibles en la app](https://userapps.support.sap.com/sap/support/knowledge/en/3589387)
- [KBA 3328850: campos del coding block como atributos custom en la partida del asiento](https://userapps.support.sap.com/sap/support/knowledge/en/3328850)
- [ACDOCA Extensibility Options, S4HANA Guide](https://s4hanaguide.com/s4hana-acdoca-extensibility-guide/)
- [Enabling the CFL custom fields using SCFD_EUI, SAP Community](https://community.sap.com/t5/application-development-and-automation-blog-posts/enabling-the-cfl-custom-fields-with-f4-help-using-scfd-eui-transaction-from/ba-p/13538429)
- [Adding Custom database Field to standard Fiori apps with Custom Fields and Logic, SAP Community](https://community.sap.com/t5/enterprise-resource-planning-blog-posts-by-sap/adding-custom-database-field-to-standard-fiori-apps-in-s-4hana-with-custom/ba-p/13450040)

## Conclusión

Las preguntas de la introducción quedaron respondidas con evidencia del propio sistema. Un Business Context es el contrato de extensibilidad de un objeto de negocio, y vive físicamente en el Extensibility Registry de `SCFD_REGISTRY`, donde el contrato C0 liberado, el sufijo `COB`, los includes `INCL_EEW_COBL` y las cuotas de 20 campos y 400 caracteres explican todo el comportamiento posterior. El campo se llama `ZZ1_ESU_COB` porque el prefijo es el namespace reservado de key user y el sufijo está declarado en el registro para cada contexto, sin intervención del usuario. Cuando el contexto no existe, el orden profesional de evaluación es Developer Extensibility primero (verificando las Business Object Interfaces liberadas en el SAP Business Accelerator Hub y en ADT) y extensibilidad clásica solo como último recurso deliberado y documentado. La decisión de qué habilitar deja de ser intimidante con el método jerárquico que parte de las apps del proceso y su ficha en la Fiori Apps Reference Library. El campo del coding block es siempre de posición, nunca de cabecera, porque `COBL`, `BSEG` y `ACDOCA` son estructuras de partida; un dato de cabecera exige un contexto propio o subir de pilar, y el nivel de cualquier contexto se lee en su include de persistencia. Y la auditoría integral existe: `SCFD_FIELDVIEWER` concentra en una transacción la definición, el árbol de usos, el inventario de objetos generados y la traza cronológica de cada campo. El campo, además, no se queda inerte: admite lógica de validación sin salir de la extensibilidad de key user, con reglas declarativas de sustitución y validación (que leen el campo gracias a la vista `FIN_SUBVAL_JOURNALENTRYITEM` habilitada) o, cuando la regla no alcanza, con la app Custom Logic sobre un BAdI liberado.

Para quien deba aplicar esta técnica en su propio proyecto, la decisión más valiosa no es ninguno de los clics documentados sino el método: partir de la página App Extensibility de la aplicación objetivo, leer el contrato del contexto en `SCFD_REGISTRY` antes de crear nada, dejar que la ficha de la Fiori Apps Reference Library dicte qué servicios habilitar, y auditar el resultado con Where-Used y `SCFD_FIELDVIEWER`. Cuando llegue la próxima petición de "un campo más" en el asiento contable, el lector deberá valorar si el contrato del Coding Block le alcanza con sus veinte campos de texto y numéricos, o si la naturaleza del dato lo empuja hacia Journal Entry Item, hacia la extensibilidad de desarrollador o, en el caso límite, hacia el arsenal clásico; esa evaluación temprana, hecha con los criterios y las transacciones de este artículo, es la diferencia entre una extensión que envejece bien y una deuda técnica con interfaz bonita.
