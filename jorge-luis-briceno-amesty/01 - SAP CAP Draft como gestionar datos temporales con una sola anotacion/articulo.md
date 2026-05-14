<style>
body { text-align: justify; hyphens: auto; }
img { display: block; margin: 0 auto; }
p:has(> img) { text-align: center; }
</style>

# SAP CAP Draft: ¿cómo gestionar datos temporales con una sola anotación?

- [1. Introducción](#1-introducción)
- [2. Fundamentos del Draft en SAP CAP](#2-fundamentos-del-draft-en-sap-cap)
    - [2.1. Ciclo de vida del Draft](#21-ciclo-de-vida-del-draft)
    - [2.2. Cómo habilitar el Draft](#22-cómo-habilitar-el-draft)
- [3. Ejemplo demostrativo: Employees y Contacts](#3-ejemplo-demostrativo-employees-y-contacts)
    - [3.1. Modelo de datos y servicio](#31-modelo-de-datos-y-servicio)
    - [3.2. Estructura de base de datos sin Draft](#32-estructura-de-base-de-datos-sin-draft)
    - [3.3. Inserción de datos sin Draft](#33-inserción-de-datos-sin-draft)
    - [3.4. Verificación en tablas persistentes](#34-verificación-en-tablas-persistentes)
- [4. Activación del Draft](#4-activación-del-draft)
    - [4.1. Nuevas tablas generadas](#41-nuevas-tablas-generadas)
    - [4.2. Inserción con Draft activo](#42-inserción-con-draft-activo)
    - [4.3. Los tres campos de control del Draft](#43-los-tres-campos-de-control-del-draft)
- [5. Datos en las tablas Draft vs. tablas persistentes](#5-datos-en-las-tablas-draft-vs-tablas-persistentes)
    - [5.1. El GET convencional no muestra los datos](#51-el-get-convencional-no-muestra-los-datos)
    - [5.2. Verificación en las tablas Draft](#52-verificación-en-las-tablas-draft)
    - [5.3. Consulta con IsActiveEntity=false](#53-consulta-con-isactiveentityfalse)
- [6. Activación del borrador con draftActivate](#6-activación-del-borrador-con-draftactivate)
    - [6.1. La acción draftActivate](#61-la-acción-draftactivate)
    - [6.2. Verificación del resultado](#62-verificación-del-resultado)
    - [6.3. Estado final de las tablas](#63-estado-final-de-las-tablas)
- [Conclusión](#conclusión)

---

## 1. Introducción

En algún momento, al desarrollar una aplicación, todo desarrollador ha enfrentado la necesidad de almacenar datos de manera temporal hasta que estén totalmente completos. Para comprender este escenario, conviene pensar en un caso cotidiano: imagine que acude a una entidad bancaria para abrir una cuenta. Los formularios que estas instituciones requieren para registrar a un nuevo cliente suelen ser bastante extensos, lo que puede llevar tiempo tanto para la institución como para el cliente que los completa en línea. Suponga ahora que, por alguna razón, no cuenta con toda la información solicitada y necesita esperar hasta el día siguiente para completarla. En ese momento surgen dos posibilidades: al día siguiente los datos que ingresó aún están guardados, o, por el contrario, es necesario volver a completarlos desde cero.

Esta misma situación se traslada de forma directa al desarrollo de aplicaciones empresariales. Los procesos que involucran formularios extensos o flujos de entrada de datos prolongados requieren un mecanismo que permita almacenar la información de forma temporal, es decir, que los datos no lleguen a las tablas persistentes de la base de datos hasta que el usuario final lo decida explícitamente. Sin este mecanismo, el desarrollador tendría que implementar manualmente toda la lógica de almacenamiento temporal, control de sesiones de edición y gestión de conflictos cuando varios usuarios intentan editar el mismo registro.

En el ecosistema SAP, el Cloud Application Programming Model (CAP) resuelve esta necesidad de manera elegante: basta con habilitar la anotación `@odata.draft.enabled` en la proyección del servicio para que el framework genere automáticamente toda la infraestructura de borradores. No es necesario implementar lógica adicional para las operaciones CRUD básicas, a menos que se requiera alguna acción muy específica. Este artículo explora en detalle cómo funciona este mecanismo, qué genera internamente en la base de datos y cómo interactúan las tablas persistentes con las tablas de borrador.

## 2. Fundamentos del Draft en SAP CAP

Según la documentación oficial de SAP, SAP Fiori admite sesiones de edición con estados de borrador almacenados en el servidor, de forma que los usuarios puedan interrumpir su trabajo y continuarlo más tarde, posiblemente desde diferentes lugares y dispositivos. CAP, junto con los elementos de SAP Fiori, proporciona soporte listo para usar de esta funcionalidad. La recomendación oficial es utilizar siempre el borrador cuando la aplicación necesite la entrada de datos por parte de los usuarios finales.

En términos concretos, un borrador es una versión temporal de una entidad de negocio que aún no se ha guardado explícitamente como versión activa. Esta separación entre la versión temporal y la versión activa es lo que permite que el sistema gestione tres escenarios fundamentales:

- Conservar los cambios no guardados si se interrumpe una actividad de edición, permitiendo al usuario reanudar el trabajo más tarde.
- Evitar la pérdida de datos si una aplicación finaliza inesperadamente (cierre accidental del navegador, caída de la conexión, reinicio del sistema).
- Actuar como mecanismo de bloqueo para evitar que varios usuarios editen el mismo objeto al mismo tiempo, y para que cada usuario sepa cuándo hay cambios sin guardar realizados por otro.

Para un desarrollador CAP, esta funcionalidad equivale a lo que en otros frameworks se implementaría con una combinación de tablas temporales, bloqueos optimistas y sesiones de usuario, todo resuelto por el framework con una sola línea de configuración.

### 2.1. Ciclo de vida del Draft

El flujo de trabajo del Draft sigue una secuencia bien definida que conviene comprender antes de pasar al código. La **Figura 1** muestra este ciclo de vida de forma esquemática.

![Figura 1. Diagrama del ciclo de vida del Draft: versión activa, objeto borrador principal y sub-objeto borrador.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/01%20-%20SAP%20CAP%20Draft%20como%20gestionar%20datos%20temporales%20con%20una%20sola%20anotacion/figura-1.png)

*Figura 1. Diagrama del ciclo de vida del Draft: versión activa, objeto borrador principal y sub-objeto borrador.*

El proceso se desarrolla en cuatro pasos:

1. En una página de objeto, el usuario elige **Edit**. Este clic marca el inicio de la sesión de edición.
2. El sistema crea una versión borrador del objeto. A partir de este momento, los cambios que realice el usuario no afectan a la versión activa.
3. El usuario realiza cambios en la versión borrador del objeto y navega a una página de detalles (por ejemplo, para editar registros secundarios asociados). Cuando elige **Apply**, los cambios se aplican al borrador, no a la versión activa.
4. Cuando el usuario elige **Save**, los cambios del borrador se copian a la versión activa del objeto. Solo en este momento los datos pasan a ser permanentes.

Esta arquitectura de dos capas (borrador y versión activa) es comparable con el flujo de trabajo de un sistema de control de versiones como Git: los cambios se acumulan en una rama de trabajo (el borrador) y solo pasan a la rama principal (la versión activa) cuando se confirman explícitamente con un commit (el Save). Si se descarta la rama de trabajo, los datos originales permanecen intactos.

### 2.2. Cómo habilitar el Draft

Para habilitar el borrador en una entidad expuesta por un servicio, basta con anotarla con `@odata.draft.enabled` en la definición del servicio:

```cds
annotate LogaliGroupService.Courses with @odata.draft.enabled;
```

Un detalle técnico importante: el borrador se debe aplicar a la **proyección** y no a la entidad original. De esta manera, solo la proyección maneja los estados del borrador, y la entidad base permanece limpia de esa responsabilidad.

> No es posible proyectar desde entidades habilitadas para borrador, ya que las anotaciones se propagan. Se debe habilitar el borrador para la proyección y no para la entidad original, o deshabilitar el borrador en la proyección usando `@odata.draft.enabled: null`.

La anotación `@odata.draft.enabled: null` indica que una proyección específica no debe manejar los estados del borrador, incluso si la entidad original lo tiene habilitado. Esta granularidad permite que un mismo modelo de datos soporte servicios con borrador y servicios sin borrador simultáneamente, algo muy útil cuando una entidad se expone a través de múltiples servicios con propósitos distintos (por ejemplo, un servicio de captura de datos con borrador y un servicio de solo lectura para reportes sin él).

## 3. Ejemplo demostrativo: Employees y Contacts

Para comprender con mayor facilidad el comportamiento del Draft, lo ideal es construir un escenario paso a paso que permita observar qué ocurre en la base de datos antes y después de habilitarlo. El escenario propuesto define una entidad `Employees` con una relación de uno a muchos con `Contacts`, un patrón habitual en aplicaciones empresariales donde cada empleado puede tener múltiples medios de contacto.

### 3.1. Modelo de datos y servicio

El primer paso es definir las entidades en el archivo `schema.cds` dentro de la carpeta `db`:

```cds
namespace com.logaligroup;

using {cuid} from '@sap/cds/common';

entity Employees : cuid {
    numberDocument : String(9);
    firstName      : String(40);
    lastName       : String(40);
    toContacts     : Composition of many Contacts
                         on toContacts.employee = $self;
};

entity Contacts : cuid {
    email       : String(120);
    phoneNumber : String(14);
    employee    : Association to Employees;
};
```

La relación entre ambas entidades utiliza `Composition of many`, lo que significa que los contactos pertenecen al empleado y no tienen existencia independiente. Esta distinción es relevante para el Draft porque CAP propaga automáticamente el estado de borrador a las entidades compuestas: cuando se crea un borrador de un empleado, sus contactos también entran en estado de borrador.

A continuación, se crean las proyecciones del servicio en el archivo `service.cds` dentro de la carpeta `srv`:

```cds
using {com.logaligroup as entities} from '../db/schema';

service LogaliGroup {
    entity EmployeesSet as projection on entities.Employees;
    entity ContactsSet  as projection on entities.Contacts;
};
```

Al ejecutar el proyecto con `cds watch`, el servidor muestra los endpoints disponibles. La **Figura 2** presenta la página de bienvenida del servidor `@sap/cds` con las dos proyecciones expuestas.

![Figura 2. Página de bienvenida del servidor @sap/cds con los endpoints EmployeesSet y ContactsSet.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/01%20-%20SAP%20CAP%20Draft%20como%20gestionar%20datos%20temporales%20con%20una%20sola%20anotacion/figura-12.png)

*Figura 2. Página de bienvenida del servidor @sap/cds con los endpoints EmployeesSet y ContactsSet.*

### 3.2. Estructura de base de datos sin Draft

Antes de habilitar el Draft, conviene observar el estado inicial de la base de datos. Al consultar las proyecciones mediante GET, ambas devuelven un array vacío porque aún no se han insertado registros, como muestran la **Figura 3** y la **Figura 4**.

![Figura 3. Respuesta GET de EmployeesSet sin registros.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/01%20-%20SAP%20CAP%20Draft%20como%20gestionar%20datos%20temporales%20con%20una%20sola%20anotacion/figura-20.png)

*Figura 3. Respuesta GET de EmployeesSet sin registros.*

![Figura 4. Respuesta GET de ContactsSet sin registros.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/01%20-%20SAP%20CAP%20Draft%20como%20gestionar%20datos%20temporales%20con%20una%20sola%20anotacion/figura-21.png)

*Figura 4. Respuesta GET de ContactsSet sin registros.*

A nivel de base de datos, la herramienta SQLTOOLS del entorno de desarrollo muestra la estructura generada por CAP. Como se observa en la **Figura 5**, existen dos tablas (`com_logaligroup_Contacts` y `com_logaligroup_Employees`) correspondientes a las entidades del namespace, y dos vistas (`LogaliGroup_ContactsSet` y `LogaliGroup_EmployeesSet`) correspondientes a las proyecciones del servicio.

![Figura 5. Vista de SQLTOOLS con las tablas y vistas generadas por CAP sin Draft habilitado.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/01%20-%20SAP%20CAP%20Draft%20como%20gestionar%20datos%20temporales%20con%20una%20sola%20anotacion/figura-22.png)

*Figura 5. Vista de SQLTOOLS con las tablas y vistas generadas por CAP sin Draft habilitado.*

Esta es la estructura mínima que CAP genera para cualquier servicio: las tablas almacenan los datos persistentes y las vistas exponen las proyecciones del servicio sobre esas tablas. Sin Draft, cualquier registro insertado pasa directamente a las tablas persistentes.

### 3.3. Inserción de datos sin Draft

Para verificar este comportamiento directo, se realiza una inserción mediante POST al endpoint `EmployeesSet`. La **Figura 6** muestra la solicitud y la respuesta exitosa con código 201 Created.

![Figura 6. Solicitud POST a EmployeesSet con un registro de empleado y su contacto asociado, respuesta 201 Created.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/01%20-%20SAP%20CAP%20Draft%20como%20gestionar%20datos%20temporales%20con%20una%20sola%20anotacion/figura-23.png)

*Figura 6. Solicitud POST a EmployeesSet con un registro de empleado y su contacto asociado, respuesta 201 Created.*

Tras la inserción, los datos son inmediatamente visibles a través de GET. La **Figura 7** confirma que el empleado aparece en la colección `EmployeesSet` con todos sus campos.

![Figura 7. Respuesta GET de EmployeesSet con el registro del empleado creado.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/01%20-%20SAP%20CAP%20Draft%20como%20gestionar%20datos%20temporales%20con%20una%20sola%20anotacion/figura-24.png)

*Figura 7. Respuesta GET de EmployeesSet con el registro del empleado creado.*

De igual manera, la **Figura 8** muestra el contacto asociado visible en `ContactsSet` con la referencia al empleado a través del campo `employee_ID`.

![Figura 8. Respuesta GET de ContactsSet con el contacto asociado al empleado.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/01%20-%20SAP%20CAP%20Draft%20como%20gestionar%20datos%20temporales%20con%20una%20sola%20anotacion/figura-25.png)

*Figura 8. Respuesta GET de ContactsSet con el contacto asociado al empleado.*

### 3.4. Verificación en tablas persistentes

La prueba definitiva de que los datos llegaron directamente a las tablas persistentes se obtiene consultando la base de datos con SQLTOOLS. La **Figura 9** muestra la tabla `com_logaligroup_Employees` con un registro.

![Figura 9. Tabla persistente com_logaligroup_Employees con el registro del empleado.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/01%20-%20SAP%20CAP%20Draft%20como%20gestionar%20datos%20temporales%20con%20una%20sola%20anotacion/figura-26.png)

*Figura 9. Tabla persistente com_logaligroup_Employees con el registro del empleado.*

La **Figura 10** confirma lo mismo para la tabla `com_logaligroup_Contacts`.

![Figura 10. Tabla persistente com_logaligroup_Contacts con el registro del contacto.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/01%20-%20SAP%20CAP%20Draft%20como%20gestionar%20datos%20temporales%20con%20una%20sola%20anotacion/figura-2.png)

*Figura 10. Tabla persistente com_logaligroup_Contacts con el registro del contacto.*

Hasta este punto, el comportamiento es el esperado para un servicio sin Draft: los registros fluyen directamente del POST a las tablas persistentes, sin ninguna tabla intermedia. No hay presencia de tablas de borrador ni de campos adicionales en las respuestas. Este es el punto de referencia que permite apreciar, en la siguiente sección, todo lo que cambia al habilitar una sola anotación.

## 4. Activación del Draft

Para habilitar el Draft, se agrega la anotación `@odata.draft.enabled` a la proyección raíz del servicio. En este caso, la proyección padre es `EmployeesSet`, ya que `ContactsSet` es una composición dependiente que hereda automáticamente el comportamiento de borrador. El archivo `service.cds` queda de la siguiente manera:

```cds
using {com.logaligroup as entities} from '../db/schema';

service LogaliGroup {
    @odata.draft.enabled
    entity EmployeesSet as projection on entities.Employees;
    entity ContactsSet  as projection on entities.Contacts;
};
```

Nótese que la anotación se coloca únicamente en `EmployeesSet`. No es necesario (ni correcto) anotarla también en `ContactsSet`, porque al ser una composición del empleado, CAP propaga automáticamente el estado de borrador a los contactos.

### 4.1. Nuevas tablas generadas

Al reiniciar el servidor después de agregar la anotación, la estructura de la base de datos cambia significativamente. La **Figura 11** muestra el nuevo estado en SQLTOOLS.

![Figura 11. Vista de SQLTOOLS tras habilitar el Draft, con nuevas tablas _drafts y DraftAdministrativeData.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/01%20-%20SAP%20CAP%20Draft%20como%20gestionar%20datos%20temporales%20con%20una%20sola%20anotacion/figura-3.png)

*Figura 11. Vista de SQLTOOLS tras habilitar el Draft, con nuevas tablas _drafts y DraftAdministrativeData.*

Se observan tres tablas nuevas que no existían antes:

| Tabla | Propósito |
|---|---|
| `LogaliGroup_EmployeesSet_drafts` | Almacena las versiones borrador de los empleados. |
| `LogaliGroup_ContactsSet_drafts` | Almacena las versiones borrador de los contactos. |
| `DRAFT_DraftAdministrativeData` | Controla el ciclo de vida de cada sesión de borrador (quién la creó, cuándo, si está en proceso). |

Adicionalmente, aparece una nueva vista `LogaliGroup_DraftAdministrativeData` que expone los datos administrativos del borrador a través del servicio OData.

La tabla `DRAFT_DraftAdministrativeData` merece atención especial. La **Figura 12** presenta su estructura completa.

![Figura 12. Estructura de la tabla DRAFT_DraftAdministrativeData con sus campos de control.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/01%20-%20SAP%20CAP%20Draft%20como%20gestionar%20datos%20temporales%20con%20una%20sola%20anotacion/figura-4.png)

*Figura 12. Estructura de la tabla DRAFT_DraftAdministrativeData con sus campos de control.*

El campo más importante para comprender el funcionamiento del Draft es `DraftUUID`: un identificador único que vincula cada registro de borrador con su sesión de edición. Los campos restantes controlan aspectos de auditoría y seguridad, como la fecha de creación (`CreationDateTime`), el usuario que creó el borrador (`CreatedByUser`), la fecha del último cambio (`LastChangeDateTime`) y si el borrador fue procesado (`DraftIsProcessedByMe`). Esta estructura es la que permite a CAP saber, en todo momento, quién está editando qué registro y desde cuándo.

### 4.2. Inserción con Draft activo

Con el Draft habilitado, el comportamiento de una inserción POST cambia radicalmente. Al crear un nuevo empleado de la misma forma que antes, la respuesta del servidor sigue siendo 201 Created, pero ahora incluye tres campos adicionales que no aparecían sin Draft. La **Figura 13** muestra esta diferencia.

![Figura 13. Solicitud POST a EmployeesSet con Draft activo, mostrando los campos HasActiveEntity, HasDraftEntity e IsActiveEntity.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/01%20-%20SAP%20CAP%20Draft%20como%20gestionar%20datos%20temporales%20con%20una%20sola%20anotacion/figura-5.png)

*Figura 13. Solicitud POST a EmployeesSet con Draft activo, mostrando los campos HasActiveEntity, HasDraftEntity e IsActiveEntity.*

La respuesta confirma la creación exitosa, pero los tres nuevos campos revelan que el registro no se almacenó en la tabla persistente sino en la tabla de borrador. Estos tres campos son el mecanismo central a través del cual CAP comunica el estado de cada registro.

### 4.3. Los tres campos de control del Draft

Cada registro gestionado por el Draft incluye tres campos booleanos que, combinados, definen completamente su estado actual:

| Campo | Significado | Valores |
|---|---|---|
| `HasActiveEntity` | Indica si el borrador tiene una versión activa correspondiente en la tabla persistente. | `true`: existe una versión activa del registro (se está editando un registro existente). `false`: no existe versión activa (es un borrador nuevo que aún no se ha publicado). |
| `HasDraftEntity` | Indica si un registro activo tiene un borrador pendiente de edición. | `true`: existe un borrador para este registro activo (alguien lo está editando). `false`: no hay borrador pendiente. |
| `IsActiveEntity` | Indica si la versión consultada es la activa o la de borrador. | `true`: el registro es la versión activa (persistente). `false`: el registro es la versión borrador (temporal). |

En la respuesta de la **Figura 13**, los tres campos muestran `false`, lo que significa: no hay versión activa de este registro (`HasActiveEntity: false`), no hay borrador pendiente desde la perspectiva de un registro activo (`HasDraftEntity: false`), y la versión actual es un borrador (`IsActiveEntity: false`). Dicho de otra manera, se trata de un registro completamente nuevo que solo existe como borrador.

Para un desarrollador acostumbrado a trabajar con estados de entidad en otros contextos (por ejemplo, registros con un campo `status` de tipo `enum`), la lógica de estos tres booleanos puede parecer redundante. Sin embargo, la separación en tres campos permite a los clientes Fiori Elements construir la interfaz de usuario de forma declarativa: mostrar un icono de candado cuando `HasDraftEntity` es `true` en la versión activa, ocultar el botón Save cuando `IsActiveEntity` es `true` (no hay nada que guardar), o mostrar un mensaje de advertencia cuando el usuario intenta editar un registro cuyo `HasDraftEntity` ya es `true` (otro usuario lo está editando).

## 5. Datos en las tablas Draft vs. tablas persistentes

### 5.1. El GET convencional no muestra los datos

El comportamiento más revelador del Draft aparece al intentar consultar los datos recién insertados con un GET convencional. A diferencia del escenario sin Draft, donde los datos eran visibles inmediatamente, ahora el GET devuelve un array vacío para ambas proyecciones. La **Figura 14** y la **Figura 15** muestran este resultado.

![Figura 14. Respuesta GET de EmployeesSet vacía tras la inserción con Draft activo.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/01%20-%20SAP%20CAP%20Draft%20como%20gestionar%20datos%20temporales%20con%20una%20sola%20anotacion/figura-6.png)

*Figura 14. Respuesta GET de EmployeesSet vacía tras la inserción con Draft activo.*

![Figura 15. Respuesta GET de ContactsSet vacía tras la inserción con Draft activo.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/01%20-%20SAP%20CAP%20Draft%20como%20gestionar%20datos%20temporales%20con%20una%20sola%20anotacion/figura-7.png)

*Figura 15. Respuesta GET de ContactsSet vacía tras la inserción con Draft activo.*

Este resultado, que a primera vista puede resultar desconcertante, es el comportamiento correcto y esperado. El GET convencional consulta las versiones activas de los registros, es decir, aquellos con `IsActiveEntity: true`. Como el registro recién creado solo existe como borrador (`IsActiveEntity: false`), no aparece en la consulta estándar. Los datos están en la base de datos, pero en las tablas de borrador, no en las persistentes.

### 5.2. Verificación en las tablas Draft

Para confirmar que los datos efectivamente se almacenaron en las tablas de borrador, se consultan directamente con SQLTOOLS. La **Figura 16** muestra la tabla `LogaliGroup_EmployeesSet_drafts` con el registro del empleado.

![Figura 16. Tabla LogaliGroup_EmployeesSet_drafts con el registro del empleado en estado borrador.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/01%20-%20SAP%20CAP%20Draft%20como%20gestionar%20datos%20temporales%20con%20una%20sola%20anotacion/figura-8.png)

*Figura 16. Tabla LogaliGroup_EmployeesSet_drafts con el registro del empleado en estado borrador.*

La **Figura 17** muestra la tabla `LogaliGroup_ContactsSet_drafts` con el contacto asociado, incluyendo la referencia al `DraftAdministrativeData_DraftUUID` que vincula ambos registros a la misma sesión de borrador.

![Figura 17. Tabla LogaliGroup_ContactsSet_drafts con el registro del contacto en estado borrador.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/01%20-%20SAP%20CAP%20Draft%20como%20gestionar%20datos%20temporales%20con%20una%20sola%20anotacion/figura-9.png)

*Figura 17. Tabla LogaliGroup_ContactsSet_drafts con el registro del contacto en estado borrador.*

Los registros están presentes en ambas tablas de borrador, con los campos `IsActiveEntity` y `HasActiveEntity` reflejando que son borradores nuevos sin versión activa correspondiente. El campo `DraftAdministrativeData_DraftUUID` vincula cada registro a los datos administrativos que controlan su ciclo de vida.

### 5.3. Consulta con IsActiveEntity=false

Para acceder a los datos del borrador a través de la API OData, es necesario indicar explícitamente que se quiere consultar la versión borrador del registro. Esto se logra añadiendo el parámetro `IsActiveEntity=false` junto con el ID del registro en la URL. La **Figura 18** muestra la consulta para el empleado.

![Figura 18. Consulta GET de EmployeesSet con IsActiveEntity=false, mostrando los datos del borrador.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/01%20-%20SAP%20CAP%20Draft%20como%20gestionar%20datos%20temporales%20con%20una%20sola%20anotacion/figura-10.png)

*Figura 18. Consulta GET de EmployeesSet con IsActiveEntity=false, mostrando los datos del borrador.*

De igual forma, la **Figura 19** muestra la consulta equivalente para el contacto.

![Figura 19. Consulta GET de ContactsSet con IsActiveEntity=false, mostrando los datos del borrador del contacto.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/01%20-%20SAP%20CAP%20Draft%20como%20gestionar%20datos%20temporales%20con%20una%20sola%20anotacion/figura-11.png)

*Figura 19. Consulta GET de ContactsSet con IsActiveEntity=false, mostrando los datos del borrador del contacto.*

Ambas respuestas devuelven los datos completos del registro, pero con `IsActiveEntity: false`, confirmando que se está accediendo a la versión borrador. Mientras el registro no se active, esta es la única forma de consultarlo a través de la API.

## 6. Activación del borrador con draftActivate

### 6.1. La acción draftActivate

Para que los datos del borrador pasen a las tablas persistentes y se conviertan en registros activos, es necesario invocar la acción `draftActivate`. Esta acción es una bound action (acción vinculada) que CAP genera automáticamente para cada entidad habilitada con Draft. Se invoca mediante un POST a la URL del registro borrador, añadiendo la acción al final de la ruta.

La URL sigue el patrón:

```
POST /odata/v4/<servicio>/<Entidad>(ID=<uuid>,IsActiveEntity=false)/<Servicio>.draftActivate
```

La **Figura 20** muestra la respuesta del servidor tras la activación. Se destacan dos elementos clave: la acción `LogaliGroup.draftActivate` en la URL, y el cambio del campo `IsActiveEntity` de `false` a `true`, resaltado en la imagen.

![Figura 20. Resultado de la acción draftActivate con IsActiveEntity cambiando de false a true.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/01%20-%20SAP%20CAP%20Draft%20como%20gestionar%20datos%20temporales%20con%20una%20sola%20anotacion/figura-13.png)

*Figura 20. Resultado de la acción draftActivate con IsActiveEntity cambiando de false a true.*

Este cambio es el punto de inflexión del ciclo de vida del Draft: a partir de este momento, los datos existen en las tablas persistentes y son visibles para cualquier consulta GET convencional.

### 6.2. Verificación del resultado

Tras la activación, el GET convencional a `EmployeesSet` ya devuelve el registro con todos sus campos y con `IsActiveEntity: true`, como muestra la **Figura 21**.

![Figura 21. Respuesta GET de EmployeesSet tras la activación, con IsActiveEntity en true.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/01%20-%20SAP%20CAP%20Draft%20como%20gestionar%20datos%20temporales%20con%20una%20sola%20anotacion/figura-14.png)

*Figura 21. Respuesta GET de EmployeesSet tras la activación, con IsActiveEntity en true.*

Lo mismo ocurre con `ContactsSet`, como confirma la **Figura 22**.

![Figura 22. Respuesta GET de ContactsSet tras la activación, con IsActiveEntity en true.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/01%20-%20SAP%20CAP%20Draft%20como%20gestionar%20datos%20temporales%20con%20una%20sola%20anotacion/figura-15.png)

*Figura 22. Respuesta GET de ContactsSet tras la activación, con IsActiveEntity en true.*

### 6.3. Estado final de las tablas

La verificación completa requiere contrastar el estado de las tablas persistentes y las tablas de borrador después de la activación.

Las tablas persistentes ahora contienen los registros. La **Figura 23** muestra `com_logaligroup_Employees` con el empleado activado.

![Figura 23. Tabla persistente com_logaligroup_Employees con el registro tras la activación del borrador.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/01%20-%20SAP%20CAP%20Draft%20como%20gestionar%20datos%20temporales%20con%20una%20sola%20anotacion/figura-16.png)

*Figura 23. Tabla persistente com_logaligroup_Employees con el registro tras la activación del borrador.*

La **Figura 24** confirma lo mismo para `com_logaligroup_Contacts`.

![Figura 24. Tabla persistente com_logaligroup_Contacts con el contacto tras la activación del borrador.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/01%20-%20SAP%20CAP%20Draft%20como%20gestionar%20datos%20temporales%20con%20una%20sola%20anotacion/figura-17.png)

*Figura 24. Tabla persistente com_logaligroup_Contacts con el contacto tras la activación del borrador.*

Por otro lado, las tablas de borrador quedan vacías después de la activación. La **Figura 25** y la **Figura 26** muestran respectivamente `LogaliGroup_EmployeesSet_drafts` y `LogaliGroup_ContactsSet_drafts` con cero registros.

![Figura 25. Tabla LogaliGroup_EmployeesSet_drafts vacía tras la activación del borrador.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/01%20-%20SAP%20CAP%20Draft%20como%20gestionar%20datos%20temporales%20con%20una%20sola%20anotacion/figura-18.png)

*Figura 25. Tabla LogaliGroup_EmployeesSet_drafts vacía tras la activación del borrador.*

![Figura 26. Tabla LogaliGroup_ContactsSet_drafts vacía tras la activación del borrador.](https://nphrrbjuwfjfoqnxviaj.supabase.co/storage/v1/object/public/articulos/jorge-luis-briceno-amesty/01%20-%20SAP%20CAP%20Draft%20como%20gestionar%20datos%20temporales%20con%20una%20sola%20anotacion/figura-19.png)

*Figura 26. Tabla LogaliGroup_ContactsSet_drafts vacía tras la activación del borrador.*

Este resultado es coherente con el diseño del mecanismo: una vez que el borrador se activa, sus datos se copian a las tablas persistentes y se eliminan de las tablas de borrador. No se mantiene duplicidad entre ambas capas. El ciclo queda cerrado: los datos temporales cumplieron su función de staging area y el registro activo es ahora la única fuente de verdad.

## Conclusión

Las tablas de Draft en CAP constituyen una característica que permite la edición temporal de datos antes de su confirmación final. Su principal valor radica en proporcionar una experiencia de edición donde los usuarios pueden realizar cambios sin que estos se reflejen inmediatamente en los datos persistentes de la base de datos principal.

El mecanismo se resume en tres puntos fundamentales que conviene retener:

Cuando un usuario inicia la edición de un registro, el sistema crea una copia temporal en una tabla de borrador. Los cambios se aplican a esta copia, permitiendo que el usuario revise y modifique antes de confirmar.

Para cada proyección con habilitación de Draft, CAP genera automáticamente una tabla de borrador con el nombre del servicio seguido del sufijo `_drafts`. Los datos en la tabla de borrador no son visibles desde la tabla original hasta que se ejecuta la confirmación final mediante la acción `draftActivate`.

Una vez que el usuario confirma los cambios, el campo `IsActiveEntity` pasa de `false` a `true`, los datos se copian a la tabla persistente y el borrador se elimina.

Todo este comportamiento, que en un desarrollo manual implicaría decenas de líneas de código entre tablas temporales, triggers de limpieza, bloqueos de concurrencia y lógica de activación, se obtiene en CAP con una sola anotación: `@odata.draft.enabled`. Para cualquier desarrollador que trabaje con aplicaciones SAP Fiori que requieran captura de datos, dominar este mecanismo no es solo una cuestión de productividad sino una decisión arquitectónica que define cómo la aplicación gestiona la integridad de los datos desde el primer formulario.
