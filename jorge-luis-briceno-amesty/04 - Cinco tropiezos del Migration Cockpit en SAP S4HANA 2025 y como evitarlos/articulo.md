<style>
body { text-align: justify; hyphens: auto; }
img { display: block; margin: 0 auto; }
p:has(> img) { text-align: center; }
</style>

# Cinco incidencias frecuentes del Migration Cockpit en S/4HANA y cómo evitarlas

El equipo Basis entrega el sistema y el consultor abre Migrate Your Data para crear su primer proyecto. En lugar del asistente aparece un mensaje que pide ciertos ajustes, o directamente falta el botón de creación. Incidencias como esta no tienen que ver con los datos del negocio y, sin embargo, consumen los días que el plan reservaba para limpiarlos y probarlos.

Cinco preguntas resumen lo que conviene tener claro antes de migrar el primer dato en SAP S/4HANA 2025 on-premise. ¿Qué enfoque conviene a cada objeto? ¿Por qué un sistema recién instalado no deja crear el proyecto? ¿Qué hacer si Customer o Supplier no aparecen? ¿Cómo se mantiene al día el contenido sin desestabilizar las pruebas? ¿Y qué necesita SAP para resolver un caso a la primera? La **Figura 1** sitúa cada incidencia en la fase del proyecto donde suele aparecer.

![Figura 1. Las cinco fases del proyecto de migración y la incidencia típica de cada una.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/04%20-%20Cinco%20tropiezos%20del%20Migration%20Cockpit%20en%20SAP%20S4HANA%202025%20y%20como%20evitarlos/figura-1.png)

*Figura 1. Las cinco fases del proyecto de migración y la incidencia típica de cada una.*

## 1. ¿Staging tables o Direct Transfer?

Migrate Your Data reúne dos enfoques. Con **staging tables**, los datos llegan desde plantillas XML o CSV, o desde una herramienta ETL, de extracción, transformación y carga, a unas tablas intermedias donde se pueden depurar. Con **Direct Transfer**, el cockpit los lee por RFC de un sistema SAP de origen que tenga instalado el complemento DMIS. En los dos casos, la escritura final pasa por las API estándar de cada objeto, las mismas que usa la aplicación de negocio.

La elección depende del origen de los datos y del catálogo de la versión, más que de la costumbre del equipo. De acuerdo con SAP, en la versión 2025 FPS02 hay 236 objetos para staging tables y 150 para Direct Transfer desde SAP ERP. Que el origen sea un SAP ERP no garantiza, por tanto, que el objeto exista en Direct Transfer. Conviene saber además que un proyecto trabaja con un solo enfoque: si el alcance necesita los dos, hacen falta dos proyectos, con sus valores de mapeo mantenidos en paralelo.

La decisión se parece a la que toma un desarrollador de integraciones al elegir entre una interfaz por ficheros y una llamada RFC directa: el fichero admite cualquier emisor y deja un punto intermedio donde revisar lo que llega, y la llamada directa ahorra el trasiego, pero exige que los dos extremos hablen el mismo idioma. Incluso con un origen SAP admitido, puede convenir cargar un objeto por staging tables cuando exige una limpieza a fondo o su volumen es muy pequeño, y en ese caso va a un segundo proyecto. La **Figura 2** ordena esas preguntas en un árbol de decisión.

![Figura 2. Árbol de decisión entre staging tables y Direct Transfer para cada objeto.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/04%20-%20Cinco%20tropiezos%20del%20Migration%20Cockpit%20en%20SAP%20S4HANA%202025%20y%20como%20evitarlos/figura-3.png)

*Figura 2. Árbol de decisión entre staging tables y Direct Transfer para cada objeto.*

## 2. ¿Por qué el sistema no deja crear el proyecto?

El síntoma tiene dos caras: un mensaje que pide ciertos ajustes o un asistente sin el botón Create Project. Detrás hay tres causas que se descartan en orden, de la más rápida a la que exige más coordinación con otros equipos, como resume la **Figura 3**.

![Figura 3. Tres comprobaciones cuando el cockpit no deja crear el proyecto.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/04%20-%20Cinco%20tropiezos%20del%20Migration%20Cockpit%20en%20SAP%20S4HANA%202025%20y%20como%20evitarlos/figura-4.png)

*Figura 3. Tres comprobaciones cuando el cockpit no deja crear el proyecto.*

La primera es el **servicio OData** `MIG_MC_ODATA_SRV`. La KBA 3039598, un artículo de la base de conocimiento de SAP, propone una prueba en `/IWFND/GW_CLIENT` sobre su `$metadata`. Si la respuesta es un 403, el equipo Basis activa el servicio en `/IWFND/MAINT_SERVICE` con el alias LOCAL. Si es un 200, lo que falta es que Basis le asigne el alias de sistema. Cualquier desarrollador Fiori conoce la escena: una aplicación nueva que falla porque nadie registró su servicio en Gateway.

La segunda es la **modificabilidad**. Desde SAP S/4HANA 2020 los proyectos solo se crean en sistemas abiertos a cambios, lo que el equipo Basis comprueba en `SCC4` y en `SE06`. Es la misma barrera que encuentra quien intenta crear una clase ABAP en el sistema de calidad: el proyecto se diseña en desarrollo y viaja por transporte. La tercera son los **roles**, que asigna el responsable de seguridad: `SAP_BR_CONFIG_EXPERT_DATA_MIG` en el frontend, `SAP_CA_DMC_MC_USER` en el backend y las autorizaciones de cada objeto, porque las API comprueban permisos como si el usuario creara los datos a mano.

## 3. ¿Qué hacer si Customer o Supplier no aparecen?

En un proyecto real de staging tables, el asistente devolvía al buscar Customer las variantes que amplían clientes existentes y otros objetos relacionados, pero no el objeto principal que crea clientes, como muestra la **Figura 4**. Con Supplier ocurría lo mismo.

![Figura 4. Búsqueda de Customer en el asistente sin el objeto principal en la lista.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/04%20-%20Cinco%20tropiezos%20del%20Migration%20Cockpit%20en%20SAP%20S4HANA%202025%20y%20como%20evitarlos/figura-6.png)

*Figura 4. Búsqueda de Customer en el asistente sin el objeto principal en la lista.*

La causa estaba en el contenido de migración de esos dos objetos; el consultor había hecho bien su trabajo. La sensación se parece a la de una orden de transporte importada con errores, en la que el objeto figura en la documentación de la versión, pero no está disponible en el mandante. La solución de SAP tuvo tres pasos: implementar con `SNOTE`, la transacción de Note Assistant, la SAP Note 3208825, ejecutar el report `DMC_MC_REPAIR_CONTENT_MIG_OBJ` para los dos objetos y comprobar el resultado en el asistente. No es un caso aislado. La SAP Note 3562046 trata la misma ausencia tras un upgrade a SAP S/4HANA 2023. Por eso, cuando un objeto estándar no aparece, lo primero que se sospecha es el contenido.

## 4. ¿Cómo se mantiene al día el contenido?

SAP corrige el cockpit y sus objetos con SAP Notes que llevan instrucciones de corrección basadas en transporte, las TCI, agrupadas en tres notas compuestas: la 2780378 para la tecnología, la 2537549 para staging tables y la 2747566 para Direct Transfer. Revisarlas a mano es tedioso. Por eso existe el **Note Analyzer** del cockpit, la transacción `CNV_NA_MC`, que lista las correcciones que faltan para que el equipo Basis las implemente. Su pantalla de resultados aparece vacía hasta que se pulsa Refresh, y una lista vacía antes de ese clic no significa que el sistema esté al día.

Implementar una corrección no cambia por sí sola los proyectos que ya existen. En staging tables, la función Check for Updates detecta los objetos afectados, que quedan marcados con Update Required y se actualizan desde el proyecto. En Direct Transfer, desde la versión 2023, cada objeto muestra su estado de versión y la versión nueva se incorpora desde el Migration Object Modeler, la transacción `LTMOM` con la que se amplían los objetos.

Un proyecto se comporta como una aplicación con sus dependencias fijadas en el `package.json`: publicar una versión nueva de una biblioteca no cambia la aplicación ya construida. De ahí la regla práctica: desarrollo, calidad y producción con el mismo nivel de correcciones, y nada de upgrades a mitad de la migración salvo que sean inevitables. La **Figura 5** resume la cadena completa.

![Figura 5. Cadena de correcciones del cockpit y de su contenido.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/04%20-%20Cinco%20tropiezos%20del%20Migration%20Cockpit%20en%20SAP%20S4HANA%202025%20y%20como%20evitarlos/figura-8.png)

*Figura 5. Cadena de correcciones del cockpit y de su contenido.*

## 5. ¿Qué necesita SAP para resolver el caso a la primera?

Cuando nada de lo anterior resuelve el problema llega el momento de abrir un caso, y la primera pista para abrirlo bien está en el propio mensaje. Cada mensaje del cockpit lleva una clase y un número, y esa pareja indica quién habla. Las clases que empiezan por `DMC_` o `/LTB/MC` son de la herramienta. `CNV_DMC_SIN` es habitual en el contenido de migración. Cualquier otra suele venir de la API de la aplicación que escribe los datos, y su soporte corresponde al equipo de ese proceso de negocio; antes de abrir el caso conviene comprobar si la API rechaza datos que no casan con el Customizing del sistema destino.

Un desarrollador hace esa misma lectura ante una traza de error: antes de abrir una incidencia, mira en qué módulo se lanzó la excepción. Los fallos de la herramienta se abren en CA-LT-MC, o en CA-LT-PE si afectan a la maquinaria de Direct Transfer; los del contenido, en CA-GTF-MIG para staging tables y en CA-DT-MIG para Direct Transfer; y los de las API, en el componente de la aplicación. La **Figura 6** recorre el camino del mensaje al componente de soporte correcto.

![Figura 6. Del mensaje de error al caso de soporte en el componente correcto.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/04%20-%20Cinco%20tropiezos%20del%20Migration%20Cockpit%20en%20SAP%20S4HANA%202025%20y%20como%20evitarlos/figura-11.png)

*Figura 6. Del mensaje de error al caso de soporte en el componente correcto.*

Antes de abrir el caso se buscan SAP Notes y KBA y se ejecuta el Note Analyzer, porque es lo primero que pedirá el soporte. El caso lleva una sola incidencia, el mensaje con su clase y su número, los pasos para reproducirlo, lo que ya se probó, el sistema con la conexión remota preparada y el impacto en el negocio. Ese impacto pesa en la prioridad, y en un sistema de pruebas se explica con el calendario del proyecto: una frase como «la carga de proveedores bloquea el ensayo general del 15 de marzo» comunica más que cualquier adjetivo.

## 6. Lista de comprobación antes del primer dato

Cada incidencia deja una comprobación que puede hacerse antes de que el problema aparezca, del mismo modo que la lista de verificación de un transporte a producción evita confiar en la memoria del equipo. Las diez que siguen, con el responsable habitual entre paréntesis, conviene completarlas antes de subir el primer archivo o seleccionar el primer registro:

1. Elegir el enfoque de cada objeto con el catálogo de la versión delante (consultor de migración).
2. Comprobar que el `$metadata` de `MIG_MC_ODATA_SRV` responde con un 200 y que el servicio tiene alias (equipo Basis).
3. Confirmar en `SCC4` y `SE06` que el sistema de desarrollo es modificable (equipo Basis).
4. Asignar los roles del cockpit y las autorizaciones de cada objeto (responsable de seguridad).
5. Acordar el paquete transportable de los proyectos de staging tables (consultor de migración).
6. En Direct Transfer, verificar el DMIS del origen con la SAP Note 2572945 (equipo Basis del origen).
7. Ejecutar el Note Analyzer e implementar las notas y TCI que falten (equipo Basis).
8. Verificar que todos los objetos del alcance aparecen en el asistente (consultor de migración).
9. Igualar el nivel de correcciones en desarrollo, calidad y producción (equipo Basis).
10. Tener lista la plantilla del caso a SAP, con el mensaje, los pasos, el sistema, el impacto y el componente (consultor de migración).

## Referencias

- [Data Migration, SAP S/4HANA, SAP Help Portal](https://help.sap.com/S4_OP_DM)
- [Migration Objects for SAP S/4HANA, SAP Help Portal](https://help.sap.com/S4_OP_MO)
- [SAP S/4HANA Migration Cockpit, página temática de SAP Community](https://community.sap.com/topics/s4hana-migration-cockpit)
- [Get Support, SAP Support Portal](https://support.sap.com/en/my-support/incidents.html)
- [KBA 3039598: Migration Cockpit Fiori App, Cannot Create Project](https://me.sap.com/notes/3039598)
- [SAP Note 3043614: Modifiable Systems and Transport Concept](https://me.sap.com/notes/3043614)
- [SAP Note 3208825: objetos SIF_CUSTOMER_2 y SIF_VENDOR_2](https://me.sap.com/notes/3208825)
- [SAP Note 3016862: Note Analyzer del Migration Cockpit](https://me.sap.com/notes/3016862)
- [SAP Note 3562046: Customer y Supplier ausentes en staging tables tras un upgrade a SAP S/4HANA 2023](https://me.sap.com/notes/3562046)
- [SAP Note 2780378: notas compuestas de la tecnología del Migration Cockpit](https://me.sap.com/notes/2780378)
- [SAP Note 2537549: notas compuestas de los objetos de staging tables](https://me.sap.com/notes/2537549)
- [SAP Note 2747566: notas compuestas de los objetos de Direct Transfer](https://me.sap.com/notes/2747566)
- [SAP Note 2572945: compatibilidad de DMIS con SAP S/4HANA](https://me.sap.com/notes/2572945)
- Densborn, F., Finkbohner, F., Höft, M., Klöß, P., Mathäß, K. y Rubarth, B. (2026). *Migrating to SAP S/4HANA* (4.ª ed.). Rheinwerk Publishing, SAP PRESS.

## Conclusión

Las cinco preguntas tienen respuesta concreta. El enfoque se elige por el origen y por el catálogo de la versión. Un proyecto que no se crea apunta al servicio OData, a la modificabilidad o a los roles. Un objeto ausente apunta al contenido, que se repara con la nota y el report que indica SAP. El contenido se mantiene al día con el Note Analyzer y con el mismo nivel de correcciones en todo el landscape. Y un caso que llega al componente correcto, con el impacto bien explicado, se resuelve antes.

Evitar estas incidencias está al alcance de cualquier equipo, con o sin experiencia previa en el cockpit, siempre que las revise antes de que el calendario apriete. Vale la pena pensar en el último proyecto de migración: ¿cuántos días se fueron en alguna de estas cinco situaciones, y cuántos se habrían ahorrado con la lista revisada el primer día con el equipo Basis?
