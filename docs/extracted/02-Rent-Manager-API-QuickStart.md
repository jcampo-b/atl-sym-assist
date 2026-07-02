# Rent Manager API — Quick Start

> Fuente: `QuickStart-12_2605_9670_23445.pdf`

## Introducción

El Rent Manager Web API (API) ofrece una solución RESTful para interactuar con Rent Manager Online (RMO), con acceso de **lectura** y **escritura** a la base de datos.

**Ejemplos de uso:**
- Un sitio web que se actualiza dinámicamente en tiempo real con cambios de Rent Manager.
- Un dashboard web que muestra métricas de rendimiento de los leasing agents.
- Una app de kiosco donde la gente ingresa sus datos y se crean prospects automáticamente en Rent Manager.
- Que otro software cargue información en tiempo real a Rent Manager.

## Requisitos

- Comprar acceso a la API (contactar al Sales Representative para pricing).
- Tener una cuenta de usuario RMO activa:
  - Con los permisos adecuados para la tarea (ej. "add prospects" para crear prospects vía kiosco).
  - Si se usan "Locations", el usuario debe tener acceso a la Location correspondiente.

## RESTful API

Basada en **Resources** y **Verbs**.

### Resources — ejemplos

| Resource | Acceso a entidad |
|---|---|
| Tenants | Cuentas de tenants |
| Units | Información de unidades |
| Vendors | Cuentas de vendors |
| RecurringCharges | Posteo de cargos recurrentes |

### Tipos de Resources

| Tipo | Descripción | Ejemplos |
|---|---|---|
| **Collection Resource** | Grupo de registros dentro de un resource; puede filtrarse | Todas las properties, todos los tenants, todas las active properties |
| **Instance Resource** | Un único registro | Una property, una unidad, un vendor |
| **Action Resource** | Dispara "trabajo" en Rent Manager | Post recurring charges, post late fees, merge vendors |

### Verbs

| Verb | Sobre Collection Resource | Sobre Instance Resource |
|---|---|---|
| **Get** | Recupera la colección de registros | Recupera el registro específico |
| **Post** | Crea uno o más registros nuevos | Actualiza el registro específico |
| **Delete** | Elimina los registros | Elimina el registro específico |

> GET, POST y DELETE se usan en Collection y Instance Resources. **Solo POST** se usa en Action Resources.

## Requests

Un Request = **Verb + Resource**: hacer algo con/al resource especificado en Rent Manager.

> Para la estructura formal completa (headers, body de POST, Data Models), ver la **RM API Overview Guide**.

### Componentes de la URL

| Componente | ¿Requerido? | Descripción |
|---|---|---|
| **Protocol** | Sí | Asegura conexión SSL. Único tipo soportado. Ej: `https://` |
| **Hostname** | Sí | Identificador corporativo de la empresa. Ej: `corpid` (se reemplaza por el identificador único asignado por LCS) |
| **Resource** | Sí | Tipo/clase de dato sobre el que actúa el Verb. Define si es Collection o Instance Resource |
| **Parameters** | No | Refinan/filtran lo que el Request retorna, actualiza o elimina |

**Ejemplos de Resource:**
- `/properties` → Collection Resource completa de Properties
- `/tenants` → Collection Resource completa de Tenants
- `/tenants/1` → Instance Resource, el tenant con ID 1

**Ejemplos de Parameters:**
- `?fields=City` → obtiene/actualiza/elimina solo el campo "City"
- `?filters=PropertyID,eq,1&fields=Addresses` → obtiene/actualiza/elimina las Addresses de los registros asociados a la property con ID 1

### Ejemplo 1

```
GET https://corpid.api.rentmanager.com/tenants
```

| Componente | Elemento URL | Descripción |
|---|---|---|
| `https://corpid.api.rentmanager.com` | Protocol + Hostname | Conexión SSL, dirige a la BD corporativa |
| `/tenants` | Resource | Collection Resource: todos los tenants |

**Traducción:** Recuperar todos los tenants de la base de datos de Rent Manager.

### Ejemplo 2

```
DELETE https://corpid.api.rentmanager.com/tenants/35
```

| Componente | Elemento URL | Descripción |
|---|---|---|
| `https://corpid.api.rentmanager.com` | Protocol + Hostname | Conexión SSL, dirige a la BD corporativa |
| `/tenants/35` | Resource | Instance Resource: tenant con ID 35 |

**Traducción:** Eliminar el tenant con ID 35 de la base de datos.

### Ejemplo 3

```
POST https://corpid.api.rentmanager.com/tenants/35?fields=LastName
```

| Componente | Elemento URL | Descripción |
|---|---|---|
| `https://corpid.api.rentmanager.com` | Protocol + Hostname | Conexión SSL |
| `/tenants/35` | Resource | Instance Resource: tenant con ID 35 |
| `?fields=LastName` | Parameter | Actualiza el campo LastName |

**Traducción:** Actualizar el tenant con ID 35 en la base de datos.

## Enviando Requests a la API

Una vez autenticado, se pueden hacer requests a la base de datos. Ejemplo: recuperar un solo tenant (C#).

> Descargar y revisar la **Rent Manager API Sample Application** para más ejemplos (ejecución de requests, transformación de datos a modelos usables).

```csharp
using System.Net.Http;
using System.Net.Http.Headers;

static async Task GetObject(string apiToken)
{
    HttpClient client = new HttpClient();

    client.BaseAddress = new Uri("https://corpid.api.rentmanager.com/");
    client.DefaultRequestHeaders.Accept.Add(new
        MediaTypeWithQualityHeaderValue("application/json"));
    client.DefaultRequestHeaders.Add("X-RM12Api-ApiToken", apiToken);

    HttpResponseMessage response = await client.GetAsync("/tenants/1");
    response.EnsureSuccessStatusCode();
    TenantModel tenant = response.Content.ReadAsAsync<TenantModel>().Result;

    return tenant;
}

public class TenantModel
{
    public int TenantID { get; set; }
    public string Name { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public bool IsCompany { get; set; }
    public int ColorID { get; set; }
    public string Comment { get; set; }
    public int RentDueDay { get; set; }
    public string LeasePeriod { get; set; }
    public bool DoNotChargeLateFees { get; set; }
    public bool DoNotPrintStatements { get; set; }
    public bool DoNotAcceptChecks { get; set; }
    public bool IsProspect { get; set; }
    public int LeasingAgentID { get; set; }
    public int AccountGroupID { get; set; }
    // ...
    public int UpdateUserID { get; set; }
}
```

> **Serialización JSON → Modelo**: el JSON devuelto por la API se serializa automáticamente al modelo provisto. Ver la Sample Application para ejemplos usando JSON sin deserializar.

## Autenticación con la API

Antes de cualquier request hay que "autenticarse" para verificar permisos. La autenticación crea un token time-sensitive que se incluye en cada request futuro. Si el token se invalida o expira, hay que re-autenticarse.

### Datos para autenticar

| Campo | Valor |
|---|---|
| **url** | `https://corpid/Authentication/AuthorizeUser` (`corpid` = identificador único asignado por LCS) |
| **method** | POST |
| **data type** | JSON |

### Ejemplo de código (C#)

```csharp
using System.Net.Http;
using System.Net.Http.Headers;

string apiToken;
HttpClient client = new HttpClient();
client.BaseAddress = new Uri("https://corpid.api.rentmanager.com/");
client.DefaultRequestHeaders.Accept.Add(new
    MediaTypeWithQualityHeaderValue("application/json"));

// Reemplazar Username, Password y Location ID con los provistos por LCS.
UserAuthorizationModel uam = new UserAuthorizationModel
{
    Username = "QsUser",
    Password = "123456",
    LocationID = 1
};

HttpResponseMessage response = await
    client.PostAsJsonAsync("/Authentication/AuthorizeUser", uam);
response.EnsureSuccessStatusCode();

apiToken = response.Content.ReadAsStringAsync().Result.Trim('"');

// Incluir el apiToken en el header de cada request siguiente:
client.DefaultRequestHeaders.Add("X-RM12Api-ApiToken", apiToken);

public class UserAuthorizationModel
{
    public string Username { get; set; }
    public string Password { get; set; }
    public int LocationID { get; set; }
}
```

> **Expiración del token**: el `apiToken` se invalida a las **24 horas**, o tras **15 minutos de inactividad**. Una vez expirado, se requiere re-autenticación.

## Rent Manager API Sample Application

App de consola en C# que ilustra autenticación, distintos métodos de resources, y generación/descarga de reportes. Aunque no se use C#, las técnicas son transferibles a otros entornos.

**Requisitos para correr la Sample Application:**
- Acceso a internet.
- Una cuenta de usuario válida y autorizada para usar la API.
- Visual Studio 2013 o superior para compilar y correr los ejemplos.

**Descarga:**
```
https://corpid.api.rentmanger.com/content/quickstartapp/LCS.UI.API.QuickStart.Examples.zip
```
(`corpid` = identificador único asignado por LCS)

## Copyright

Copyright © 2015 by London Computer Systems, Inc.
Versión 12.2605.9670.23445. Todos los derechos reservados. Esta guía y el software descrito se distribuyen bajo licencia y solo pueden usarse o copiarse conforme a los términos de dicha licencia. La información es solo para uso informativo, sujeta a cambios sin previo aviso, y no debe interpretarse como un compromiso de London Computer Systems, Inc.

London Computer Systems, Inc. no asume responsabilidad por errores o inexactitudes en esta guía. Los datos usados en los ejemplos son ficticios salvo que se indique lo contrario.
