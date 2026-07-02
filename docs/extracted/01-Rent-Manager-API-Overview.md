# Rent Manager Web API — Overview

> Fuente: `Overview-12_2605_9670_23445.pdf`

## RESTful API

El Rent Manager Web API (WAPI) sigue los principios RESTful.

### Glosario corto

| Término | Descripción |
|---|---|
| **Resource** | Punto de acceso para WAPI. Puede representar interacción de datos o una forma de ejecutar trabajo. |
| **Entity** | Los objetos a los que los resources dan acceso. `/Tenants/` es un resource que da acceso a entidades `Tenant`. |
| **URL** | Define dónde está ubicado el resource. |
| **HTTP Method / Verb** | Indica el método deseado de interacción con un resource. |

### HTTP Methods

- **GET** — Recupera datos del resource especificado.
- **POST** — Envía datos al resource especificado. Dependiendo del resource, estos datos pueden guardarse o usarse para ejecutar trabajo.
- **DELETE** — Elimina datos en el resource especificado.

## Tipos de Resources

### 1. Collection Resource

Interactúa con una colección de entidades.

**GET**
```
GET https://sampleco.api.rentmanager.com/Tenants
```
Devuelve una colección de todas las entidades de un resource. Soporta parámetros de query string: `fields`, `embed`, `filters`, `pagenumber`, `pagesize`, `nocontent`, `orderby`. Todo GET sobre una colección debe devolver el header `X-Total-Results`. La paginación se activa automáticamente si el set supera 1000 registros.

**POST**
```
POST https://sampleco.api.rentmanager.com/Tenants
```
Crea o actualiza entidades. Debe incluir una lista de objetos DataModel del tipo correcto. La respuesta contiene las entidades actualizadas.

**DELETE**
```
DELETE https://sampleco.api.rentmanager.com/Tenants
```
Elimina entidades. Debe incluir una lista de Integers con los Id's de las entidades a eliminar.

### 2. Instance Resource

Interactúa con una única entidad.

**GET**
```
GET https://sampleco.api.rentmanager.com/Tenants/{id}
```
Soporta `fields`, `embed`, `nocontent`.

**POST**
```
POST https://sampleco.api.rentmanager.com/Tenants/{id}
```
Actualiza una entidad específica. Incluye un DataModel del tipo correcto. Devuelve la entidad actualizada.

**DELETE**
```
DELETE https://sampleco.api.rentmanager.com/Tenants/{id}
```
Elimina la entidad específica.

### 3. Action Resource

Trabajo relacionado con el resource, no con entidades específicas. **Solo permite POST.**

```
POST https://sampleco.api.rentmanager.com/RecurringCharges/PostRecurringCharges
```
El body puede incluir un DataModel con información para completar el trabajo. Devuelve un HTTP Status describiendo el resultado.

### Experimental Resources

No garantizados de permanecer en la WAPI; pueden cambiar en cualquier momento.

## Estructura de un Request

Un request consta de tres partes: **URL**, **header**, **body** (el body solo en POST). WAPI solo acepta requests sobre sesión SSL.

### Ejemplo de Request

```
GET https://sampleco.api.rentmanager.com/Tenants
```

| Parte | Descripción |
|---|---|
| GET | Método HTTP |
| https | SSL Encrypted |
| sampleco | Identificador de compañía |
| api | Identificador de producto |
| rentmanager.com | URL base de RentManager |
| Tenants | Identificador de resource |

### Ejemplo de Header

```
GET /Tenants HTTP/1.1
Accept: application/json
Content-Type: application/json
Host: sampleco.api.rentmanager.com
X-RM12Api-ApiToken: 2e995f2073a64bb1ac3e649a746e62fb
```

| Parte del Header | Descripción |
|---|---|
| GET | Método (GET, POST o DELETE) |
| /Tenants | Resource |
| HTTP/1.1 | Protocolo |
| Accept | Formato de la respuesta |
| Content-Type | Formato del body del request |
| Host | Información de host y dominio |
| X-RM12Api-ApiToken | Token de autenticación requerido en todos los requests |

## Data Models

Los GET devuelven Data Models; los POST pueden incluir un data model en el body.

### Ejemplo de campos

| Field | Data Type | Features |
|---|---|---|
| ApiUri | String | |
| Id | Int | |
| FirstName | String | |
| LastName | String | |
| ColorId | Int | |
| Color | Resource | Embed |
| DateCreated | DateTime | |
| CreateUserId | Int | |
| CreateUser | Resource | Embed |
| DateUpdated | DateTime | |
| UpdateUserId | Int | |
| UpdateUser | Resource | Embed |
| PrimaryContact | Resource | Embed, Atomic Create |
| Contacts | Resource | Embed |

Todos los Data Models incluyen el campo `ApiUri`, que especifica el resource usado para obtener el dato.

### Ejemplo JSON

```json
{
  "ApiUri": "https://sampleco.api.rentmanager.com/Tenants/1",
  "Id": 1,
  "FirstName": "Joe",
  "LastName": "Rocker",
  "ColorId": 1,
  "Color": {
    "ApiUri": "https://sampleco.api.rentmanager.com/Colors/1"
  },
  "DateCreated": "2014-01-01 14:32:25",
  "CreateUserId": 1,
  "CreateUser": {
    "ApiUri": "https://sampleco.api.rentmanager.com/Users/1"
  },
  "DateUpdated": "2014-01-01 14:32:25",
  "UpdateUserId": 1,
  "UpdateUser": {
    "ApiUri": "https://sampleco.api.rentmanager.com/Users/1"
  },
  "PrimaryContact": {
    "ApiUri": "https://sampleco.api.rentmanager.com/Tenants/1/PrimaryContact"
  },
  "Contacts": [
    { "ApiUri": "https://sampleco.api.rentmanager.com/Tenants/1/Contacts" }
  ]
}
```

> Nota: se usa `ApiUri` en lugar de `@id` (recomendación W3C para Linked Data) porque `@id` no se traduce bien a modelos de datos de .NET.

## Linked Data

- Los data models pueden incluir información de otros resources relacionados, con al menos el campo `ApiUri`.
- Se puede embeber el detalle in-line con el parámetro `embed`.
- Excepto los marcados como "atomic create", los campos de linked data se ignoran en POST y DELETE.
- `ApiUri` siempre es el link más directo para obtener la info enlazada.

## Atributos de campos

- **Primary Key**: identificador único del objeto.
- **Required**: obligatorio al crear un registro vía POST.
- **Required (create)**: puede pasarse al crear; el linked data se crea junto con el padre.
- **Read Only**: no se actualiza aunque se envíe en un POST.
- **Concurrency Key**: si la concurrencia está habilitada (default), el update debe incluir `UpdateDate`.
- **Calculated Field**: no se lee directo de la BD, se calcula combinando varios campos.
- **Requires Embed '{Embed Option}'**: el campo necesita un embed para completarse.

## Query String Parameters

| Parámetro | Uso |
|---|---|
| `fields` | Devuelve solo los campos listados. |
| `embed` | Incluye un resource relacionado in-line. |
| `filter` | Filtra resultados (variante de RQL). |
| `pagenumber` | Página de resultados (junto con `pagesize`). |
| `pagesize` | Cantidad de registros por página. |
| `nocontent` | No devuelve contenido (útil en creates/updates). |
| `orderby` | Ordena los resultados. |

### Ejemplo combinado

```
GET https://sampleco.api.rentmanager.com/Tenants?fields=Id,FirstName,LastName,Color&embeds=Color&filters=LastName,eq,Rockler&pagenumber=1&pagesize=10&orderby=FirstName
```

### Fields

```
GET https://sampleco.api.rentmanager.com/Tenants?fields=Id,FirstName,LastName
```
```json
{
  "Id": 1,
  "FirstName": "Joe",
  "LastName": "Rockler"
}
```

### Embed

```
GET https://sampleco.api.rentmanager.com/Tenants?embed=Color
```
Si se combina con `fields`, el campo a embeber también debe estar en `fields`, o no se incluye en la respuesta.

### Filter (RQL)

Formato: `field,operator,value`. Múltiples filtros separados por `;`.

| Operación | Operador RQL | Descripción | Ejemplo |
|---|---|---|---|
| Between | `bt` | Entre dos valores (inclusive) | `/Tenants?filters=TenantID,bt,(2,10)` |
| Contains | `ct` | El objeto contiene el valor | `/Tenants?filters=Name,ct,John` |
| Ends With | `ew` | Termina con el valor | `/Tenants?filters=Name,ew,Hudson` |
| Equal To | `eq` | = | `/Tenants?filters=TenantID,eq,10` |
| Greater Than | `gt` | > | `/Tenants?filters=TenantDisplayID,gt,8` |
| Greater Than Or Equal | `ge` | >= | `/Tenants?filters=TenantDisplayID,ge,8` |
| Greater Than Or Equal Or Null | `gen` | >= OR null | `/ServiceManagerIssues?filters=AssignedOpenDate,gen,2018-06-06` |
| Greater Than Or Null | `gtn` | > OR null | `/ServiceManagerIssues?filters=AssignedOpenDate,gtn,2018-06-06` |
| Has Value | `hv` | Tiene un valor | `/ServiceManagerIssues?filters=AssignedOpenDate,hv,2018-06-06` |
| In | `in` | Está en una lista | `/Tenants?filters=TenantID,in,(4,5,6)` |
| Less Than | `lt` | < | `/Tenants?filters=AccountGroupID,lt,10;` |
| Less Than Or Equal | `le` | <= | `/Tenants?filters=AccountGroupID,le,10;` |
| Less Than Or Equal Or Null | `len` | <= OR null | `/ServiceManagerIssues?filters=AssignedOpenDate,len,2018-06-06` |
| Less Than Or Null | `ltn` | < OR null | `/ServiceManagerIssues?filters=AssignedOpenDate,ltn,2018-06-06` |
| Not Equal To | `ne` | <> | `/Tenants?filters=TenantID,ne,10` |
| Not In | `ni` | No está en la lista | `/Tenants?filters=TenantID,ni,(4,5,6)` |
| Starts With | `sw` | Empieza con el valor | `/Tenants?filters=Name,sw,J` |

**Filtros deprecados**: pueden dejar de funcionar después de la fecha de deprecación (normalmente por renombramiento/reemplazo).

### SaveOptions

Pasa instrucciones sobre cómo guardar ciertos modelos.

```
POST https://sampleco.api.rentmanager.com/Tenants/{id}/Charges?saveOptions=IgnoreHardClose
```

### Pagination

- Se activa automáticamente si hay más de 1000 registros.
- Si se especifica `pagenumber` y `pagesize` > 1000 (o no se especifica), `pagesize` se fija en 1000.

**Headers de respuesta:**

| Header | Propósito |
|---|---|
| `Link` | Enlaces a First, Previous, Next y/o Last. |
| `X-Total-Results` | Total de resultados encontrados. |

**Ejemplo:**
```
GET https://sampleco.api.rentmanager.com/Tenants?pagenumber=4&pagesize=25
```
```
HTTP/1.1 200 OK
Content-Type: application/json
Link: <...pagenumber=1&pagesize=25>; rel="first",
      <...pagenumber=3&pagesize=25>; rel="previous",
      <...pagenumber=5&pagesize=25>; rel="next",
      <...pagenumber=12&pagesize=25>; rel="last"
X-Total-Results: 296
```

### NoContent

Útil en create/update cuando no interesa recibir el data de vuelta.

### OrderBy — **Soporte parcial**

- No se puede ordenar por linked data aunque esté embebida.
- Múltiples campos separados por coma, en el orden dado.
- Ascendente por default; `DESC` para descendente.

```
GET https://sampleco.api.rentmanager.com/Tenants?orderby=LastName
GET https://sampleco.api.rentmanager.com/Tenants?orderby=LastName%20DESC
GET https://sampleco.api.rentmanager.com/Tenants?orderby=LastName,FirstName
```

## Concurrency — **Soporte parcial**

- Habilitada por defecto durante la autenticación.
- Si está habilitada, el update debe incluir `UpdateDate`.
- **Actualmente no se puede deshabilitar.**

## Creating a Record — **Soporte parcial**

Al crear un registro, la respuesta HTTP debe incluir un header `Location` apuntando al nuevo registro (incluso con `nocontent`).

## Partial Updates

- WAPI soporta actualizaciones parciales: solo se envían los campos a cambiar.
- Internamente, la API primero hace un fetch del objeto y luego actualiza solo los campos provistos.
- Si la concurrencia está habilitada, el POST debe incluir `UpdateDate`.

**Ejemplo (solo actualiza FirstName/LastName):**
```json
{
  "ApiUri": "https://sampleco.api.rentmanager.com/Tenants/1",
  "Id": 1,
  "FirstName": "Joe",
  "LastName": "Rocker"
}
```

## Finite vs Dynamic Results

Algunos collection resources tienen un número finito de resultados posibles definidos por el resource (ej: Addresses de un Tenant, definido por los AddressTypes asociados). El resultado incluye un registro por cada AddressType aunque esté vacío/NULL.

- **Read/Update** disponibles directamente.
- **Create**: requiere primero asociar un AddressType a la entidad.
- **Delete**: requiere desasociar el type de la entidad (elimina las addresses de ese type para todas las entidades asociadas).

**Ejemplo:**
```
GET https://sampleco.api.rentmanager.com/Tenants/115/Addresses
```
```json
{
  "Addresses": [
    {
      "APIURI": "https://sampleco.api.rentmanager.com/Addresses/12",
      "AddressTypeId": 4,
      "AddressType": { "APIURI": "https://sampleco.api.rentmanager.com/AddressTypes/4" },
      "Street1": "123 South Street",
      "Street2": "",
      "City": "Milford",
      "State": "Ohio",
      "PostalCode": "45150",
      "IsPrimary": true,
      "IsBilling": true
    },
    {
      "APIURI": "https://sampleco.api.rentmanager.com/Addresses/16",
      "AddressTypeId": 5,
      "AddressType": { "APIURI": "https://sampleco.api.rentmanager.com/AddressTypes/5" },
      "Street1": "548 Commerce Place",
      "Street2": "Suite 4",
      "City": "Loveland",
      "State": "Ohio",
      "PostalCode": "45140",
      "IsPrimary": false,
      "IsBilling": false
    },
    {
      "APIURI": null,
      "AddressTypeId": 6,
      "AddressType": { "APIURI": "https://sampleco.api.rentmanager.com/AddressTypes/6" },
      "Street1": null, "Street2": null, "City": null,
      "State": null, "PostalCode": null,
      "IsPrimary": null, "IsBilling": null
    }
  ]
}
```

## Backwards Compatibility

**Se puede hacer sin romper compatibilidad:**
- Crear un nuevo resource.
- Crear un nuevo data model.
- Agregar un campo a un data model existente.

**No se puede hacer sin romper compatibilidad:**
- Cambiar el nombre de un resource (se podrían usar redirects si la funcionalidad sigue igual).
- Eliminar o renombrar un campo en un data model.

## Versioning

Si un resource va a eliminarse, la API responde con un header personalizado durante un período previo. Una vez eliminado, responde `410 - Gone`.

## Responding to Requests for Feedback

Ejemplos que pueden requerir feedback/confirmación: Hard Close Override, Owner Overdraft Override, sincronización de Primary Contact/Account Names (si la preferencia está en "Ask"), valores default de User Defined Fields.

- Si el usuario tiene privilegios suficientes pero no se envió el override: `412 - Precondition Failed` + ErrorModel.
- Para forzar la operación, se debe reenviar el POST con el Save Option correspondiente.

## Cross Join Tables — **Soporte parcial**

Ejemplos de relaciones cross-join en la BD de Rent Manager:
- Users ↔ Roles
- Users ↔ Privileges
- Roles ↔ Privileges
- Properties ↔ Floors
- Units ↔ Amenities
- Users ↔ Properties
- Users ↔ Bank/CC Accounts

**Dos formas de obtener elementos vinculados:**

| Método | Resultado |
|---|---|
| `GET /Users/{id}/Roles` | Lista de Roles del usuario |
| `GET /Users/{id}/RoleSelectList` | Lista de modelos `UserRoleSelectListItem` (respuesta más liviana, con seleccionados y disponibles) |

**Patrón de los Select List Item models:**

| Field | Type | Embed | Read Only | Atomic Create |
|---|---|---|---|---|
| Model1ID | Int | | X | |
| Model2ID | Int | | X | |
| Selected | Boolean | | | |
| Model1 | DataModel | X | X | |
| Model2 | DataModel | X | X | |

Se puede cambiar `Selected` y hacer POST para actualizar la asociación. Para interactuar directamente con Users o Roles, se usa el resource correspondiente.

## HTTP Response Codes

- **200 – OK**: Request exitoso.
- **201 – Created**: Registro(s) creado(s).
- **204 – No Content**: Exitoso sin contenido (típico en DELETE).
- **206 – Partial Content**: Paginación activada; incluye headers `Link` y `X-Total-Results`.
- **400 – Bad Request**: Datos de entrada malformados.
- **401 – Unauthorized**: Cliente no autenticado.
- **403 – Forbidden**: Cliente sin permisos suficientes.
- **404 – Not Found**: No se encontró el/los registro(s).
- **405 – Method Not Allowed**: Método no permitido en ese resource.
- **409 – Conflict**: Duplicado (create) o cambio de concurrencia (update).
- **412 – Precondition Failed**: No se cumplió una precondición (ej. hard close sin override).
- **429 – Too Many Requests**: Rate limiting.
- **500 – Internal Server Error**: Error inesperado del servidor.

## Resources and Permissions

**`GET /Resource`** (colección, según filtro):
- Acceso a al menos un registro → **200 – OK**
- Sin acceso a ningún registro (sin importar el filtro) → **403 – Unauthorized**
- Sin registros tras aplicar el filtro (incluso por permisos) → **404 – Not Found**

**`GET /Resource/{Id}`**:
- Con acceso al registro → **200 – OK**
- Sin encontrarlo o sin acceso → **404 – Not Found**

## Authentication Process

1. POST a `/Authentication/AuthorizeUser` con un `UserAuthorizationModel` (mínimo username y password).
2. WAPI busca la compañía según la URL y obtiene la info de conexión.
3. WAPI se conecta a la BD de la compañía.
4. WAPI realiza el login:
   1. Valida las credenciales.
   2. Verifica si el usuario ya tiene un login token.
   3. Devuelve o genera un login token según corresponda.
5. Crea un `APIToken`.
6. Devuelve el `APIToken`.
7. Los requests siguientes incluyen el `APIToken` para validar la autenticación.

> El token siempre se invalida si cambia información relevante del usuario (permisos o password).
