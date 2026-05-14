<style>
body { text-align: justify; hyphens: auto; }
</style>

# Composition of One en CAP: ¿por qué falla con Draft y cuál es la alternativa?

- [1. Introducción](#1-introducción)
- [2. Associations y Compositions: dos modelos de relación](#2-associations-y-compositions-dos-modelos-de-relación)
    - [2.1. Associations: relaciones con ciclo de vida independiente](#21-associations-relaciones-con-ciclo-de-vida-independiente)
    - [2.2. Compositions: relaciones con ciclo de vida dependiente](#22-compositions-relaciones-con-ciclo-de-vida-dependiente)
- [3. Composition of One: la relación uno a uno](#3-composition-of-one-la-relación-uno-a-uno)
- [4. Por qué falla Composition of One con Draft en Fiori Elements](#4-por-qué-falla-composition-of-one-con-draft-en-fiori-elements)
    - [4.1. El mecanismo del Draft al crear una entidad](#41-el-mecanismo-del-draft-al-crear-una-entidad)
    - [4.2. El manejador personalizado como solución temporal](#42-el-manejador-personalizado-como-solución-temporal)
- [5. La alternativa recomendada: aplanar el modelo](#5-la-alternativa-recomendada-aplanar-el-modelo)
    - [5.1. De dos entidades a una](#51-de-dos-entidades-a-una)
    - [5.2. Criterio de decisión](#52-criterio-de-decisión)
- [Conclusión](#conclusión)

## 1. Introducción

Al modelar datos en el SAP Cloud Application Programming Model (CAP), resulta fundamental dominar la distinción entre Associations y Compositions. Aunque ambas herramientas sirven para establecer relaciones entre entidades, su diferencia esencial radica en el ciclo de vida de las entidades que vinculan: un espectro que va desde relaciones débiles (Associations) hasta relaciones fuertes (Compositions).

Esta distinción se vuelve especialmente crítica al enfrentarse a un escenario aparentemente simple pero lleno de matices: la composición uno a uno (Composition of One). A lo largo de este artículo se explora por qué este tipo de relación presenta desafíos significativos en aplicaciones Fiori Elements con manejo de borradores (Draft), y cuál es el enfoque recomendado para abordarlo correctamente.

## 2. Associations y Compositions: dos modelos de relación

Tanto las asociaciones como las composiciones permiten crear relaciones entre entidades en CDS, pero cada una responde a un caso de uso distinto. La diferencia fundamental no reside en la sintaxis, sino en el comportamiento que el framework asume sobre el ciclo de vida de las entidades relacionadas.

Para un desarrollador SAP familiarizado con las transacciones de negocio, esta distinción es análoga a la diferencia entre un documento de referencia que se consulta (como un maestro de materiales) y un documento dependiente que no tiene sentido fuera de su contexto (como las posiciones de un pedido de compra).

### 2.1. Associations: relaciones con ciclo de vida independiente

Las asociaciones definen relaciones débiles: cada entidad involucrada mantiene un ciclo de vida autónomo. Un ejemplo representativo en el contexto de desarrollo SAP es la relación entre departamentos y empleados. Cada entidad gestiona su propio CRUD (crear, leer, actualizar, eliminar) de forma independiente. Si se elimina un departamento, los empleados que pertenecían a él siguen existiendo en el sistema.

```cds
entity Departments : cuid {
    name      : String(100);
    employees : Association to many Employees on employees.department = $self;
};

entity Employees : cuid {
    name       : String(100);
    department : Association to Departments;
};
```

En este modelo, eliminar un departamento no afecta a la tabla de empleados. Cada entidad es dueña de su propio ciclo de vida, lo que convierte a esta relación en una referencia navegable pero sin dependencia estructural.

### 2.2. Compositions: relaciones con ciclo de vida dependiente

Las composiciones representan el extremo opuesto: relaciones fuertes donde la entidad hija depende completamente de la entidad padre. El caso más representativo en desarrollo SAP es el de una orden de venta y sus ítems. Si la orden se elimina, todos sus ítems desaparecen en cascada.

```cds
entity SalesOrders : cuid, managed {
    orderNumber : String(10);
    items       : Composition of many SalesOrderItems;
};

entity SalesOrderItems : cuid {
    product  : String(80);
    quantity : Integer;
    price    : Decimal(10, 2);
};
```

Una de las ventajas más relevantes de las composiciones es la capacidad de realizar operaciones Deep Insert: crear la entidad padre junto con todas sus hijas en una única transacción. Este comportamiento funciona de manera nativa para relaciones uno a muchos (1:N). La pregunta que surge entonces es: si las composiciones funcionan tan bien para relaciones 1:N, ¿qué ocurre cuando la relación es uno a uno?

La siguiente tabla resume las diferencias clave entre ambos tipos de relación:

| Característica | Association | Composition |
|---|---|---|
| Tipo de relación | Débil | Fuerte |
| Ciclo de vida | Independiente | Dependiente del padre |
| Eliminación en cascada | No | Sí |
| Deep Insert | No nativo | Sí (1:N nativo) |
| Ejemplo SAP | Departments - Employees | SalesOrders - Items |
| Caso de uso | Entidades autónomas que se referencian | Entidades que solo existen dentro de un padre |

## 3. Composition of One: la relación uno a uno

La respuesta a la pregunta anterior es menos alentadora de lo esperado. La documentación oficial de SAP desaconseja el uso de composiciones uno a uno (Composition of One) para entidades, ya que en la mayoría de los casos la información de la entidad hija puede ubicarse directamente en la entidad raíz. Entre las limitaciones documentadas se encuentran una compatibilidad muy restringida con el modo Draft y la ausencia de soporte nativo para modificaciones de rutas de navegación.

Considérese el siguiente modelo donde una entidad `Products` tiene una composición uno a uno con `ProductDetails`:

```cds
type decimal : Decimal(6, 3);

entity Products : cuid, managed {
    product       : String(8);
    productName   : String(80);
    description   : LargeString;
    category      : Association to Categories;
    subCategory   : Association to SubCategories;
    statu         : Association to Status;
    price         : Decimal(6, 2);
    rating        : Decimal(3, 2);
    currency      : String;
    detail        : Composition of ProductDetails;
    supplier      : Association to Suppliers;
};

entity ProductDetails : cuid {
    baseUnit   : String default 'EA';
    width      : decimal;
    height     : decimal;
    depth      : decimal;
    weight     : decimal;
    unitVolume : String default 'CM';
    unitWeight : String default 'KG';
};
```

Desde el punto de vista del modelado, esta estructura parece perfectamente lógica: los detalles físicos del producto se agrupan en una entidad separada para mantener la claridad del modelo. Sin embargo, esta decisión de diseño tiene consecuencias directas cuando la aplicación utiliza el modo Draft.

## 4. Por qué falla Composition of One con Draft en Fiori Elements

### 4.1. El mecanismo del Draft al crear una entidad

Para entender el problema es necesario comprender primero cómo funciona el modo Draft en una aplicación Fiori Elements. Cuando un usuario presiona el botón "Crear", no se genera una entidad activa de inmediato. En su lugar, el framework crea una entidad en modo borrador donde el campo `IsActiveEntity` tiene valor `false`. Esta entidad reside en una tabla temporal de borradores, no en la tabla persistente.

La interfaz de usuario necesita un objeto de datos completo para vincular sus campos de formulario. Si la entidad `Products` tiene una Composition of One con `ProductDetails`, la UI espera que la propiedad de navegación `detail` sea un objeto válido al que pueda enlazar los campos `width`, `height`, `depth` y los demás atributos físicos. El problema radica en que, por defecto, cuando CAP crea el borrador de `Products`, la propiedad de navegación `detail` queda como `null` o `undefined`. El framework no crea automáticamente un borrador para la entidad hija porque desconoce si el usuario realmente tiene la intención de completar esa información.

El resultado es que la UI intenta acceder a propiedades de un objeto que no existe, lo que produce errores de binding y comportamientos inesperados en el formulario.

Quienes hayan trabajado con el artículo anterior de esta serie sobre el [modo Draft en SAP CAP](../01%20-%20SAP%20CAP%20Draft%20como%20gestionar%20datos%20temporales%20con%20una%20sola%20anotacion/articulo.md) reconocerán que el Draft crea tablas shadow para cada entidad habilitada. En el caso de una Composition of One, el framework genera la tabla shadow del padre pero no pre-puebla la propiedad de navegación hacia la entidad hija, lo que deja a la UI sin un objeto al que vincular sus controles.

### 4.2. El manejador personalizado como solución temporal

Para resolver este problema se puede recurrir a un manejador en JavaScript que le indique explícitamente al servicio que, al crear un borrador de `Products`, debe inicializar el objeto `detail`:

```js
this.before('NEW', Products.drafts, async (req) => {
    req.data.detail ??= {
        baseUnit: 'EA',
        width: null,
        height: null,
        depth: null,
        weight: null,
        unitVolume: 'CM',
        unitWeight: 'KG'
    };
});
```

Este código intercepta el evento de creación del borrador (`NEW` sobre `Products.drafts`). Antes de que se complete la operación, verifica si la propiedad `detail` es nula o indefinida mediante el operador de asignación nula (`??=`). Si lo es, crea manualmente un objeto con los valores por defecto de cada campo. De esta manera, se pre-puebla la estructura del borrador para que la UI disponga de un objeto válido al que vincular sus campos, satisfaciendo los requisitos de binding aunque los campos estén vacíos.

Si bien esta solución funciona correctamente, el hecho de necesitar un manejador personalizado para habilitar una funcionalidad básica de la interfaz de usuario es una señal clara de que el modelo de datos podría beneficiarse de un enfoque más simple.

## 5. La alternativa recomendada: aplanar el modelo

### 5.1. De dos entidades a una

La mejor solución suele ser la más directa: aplanar el modelo. En lugar de mantener dos entidades separadas conectadas por una Composition of One, se definen todos los campos dentro de la entidad raíz:

```cds
type decimal : Decimal(6, 3);

entity Products : cuid, managed {
    product       : String(8);
    productName   : String(80);
    description   : LargeString;
    category      : Association to Categories;
    subCategory   : Association to SubCategories;
    statu         : Association to Status;
    price         : Decimal(6, 2);
    rating        : Decimal(3, 2);
    currency      : String;
    baseUnit      : String default 'EA';
    width         : decimal;
    height        : decimal;
    depth         : decimal;
    weight        : decimal;
    unitVolume    : String default 'CM';
    unitWeight    : String default 'KG';
    supplier      : Association to Suppliers;
};
```

Con este modelo aplanado, el modo Draft funciona de manera nativa sin necesidad de manejadores personalizados. Al crear un borrador de `Products`, todos los campos están disponibles directamente en la entidad y la UI puede vincularlos sin problemas. Se elimina la entidad `ProductDetails`, y con ella desaparece la complejidad del escenario de composición uno a uno.

### 5.2. Criterio de decisión

Ante una situación donde se considere utilizar una Composition of One, resulta útil formularse una pregunta directa: ¿podría la entidad hija existir o tener sentido sin su padre?

Si la respuesta es un rotundo **no**, se trata de una composición legítima. Sin embargo, cuando la relación es uno a uno, la recomendación sigue siendo aplanar el modelo e incorporar los campos directamente en la entidad padre. La justificación es práctica: el código adicional que requiere el manejador personalizado no compensa la separación conceptual que aporta la entidad hija.

Si existe alguna duda sobre la independencia de la entidad hija, podría tratarse de una asociación simple (Association to One) en lugar de una composición. A diferencia de la Composition of One, una asociación uno a uno no implica ciclo de vida dependiente ni eliminación en cascada, y funciona sin restricciones con el modo Draft.

La composición uno a muchos (1:N) sigue siendo la forma natural y recomendada de utilizar composiciones en CAP. Es en la variante uno a uno donde surgen las limitaciones que hacen preferible el enfoque aplanado.

## Conclusión

El Cloud Application Programming Model ofrece la flexibilidad necesaria para modelar relaciones complejas como las composiciones uno a uno. Sin embargo, la práctica demuestra que esta flexibilidad tiene límites concretos cuando entra en juego el modo Draft de Fiori Elements.

La necesidad de recurrir a un manejador personalizado para habilitar funcionalidades básicas de la interfaz de usuario constituye una señal clara de que el modelo de datos no está aprovechando las capacidades nativas del framework. El modelo aplanado resuelve este problema de raíz, eliminando la dependencia de código adicional y garantizando un comportamiento predecible tanto en la creación de borradores como en las operaciones CRUD estándar.

Para cualquier desarrollador que trabaje con SAP CAP y Fiori Elements, identificar estas situaciones a tiempo permite evitar soluciones complejas donde una decisión de modelado más simple habría sido suficiente. La simplicidad en el diseño de entidades no es una limitación del framework, sino una de sus fortalezas cuando se aplica con criterio.
