# Rent Manager — Partner Network Resource Guide (2024)

> Fuente: `ResourceGuide2024.pdf`

## Índice

| Sección | Página |
|---|---|
| Technical Resources | 2, 3, 4 |
| Learning Rent Manager | 5 |
| Rent Manager Features & Customization | 6, 7 |
| Activating & Managing Your Integration | 8 |
| Supporting Rent Manager Customers | 9 |
| Marketing to Rent Manager Customers | 10 |
| Frequently Asked Questions | 11 |

---

## Technical Resources

### Plataformas para acceder a Rent Manager

Los clientes de Rent Manager acceden a sus datos mediante dos soluciones cloud-based, ambas con acceso a los mismos datos:

- **Rent Manager Express**: se accede vía navegador web.
- **Rent Manager 12**: se accede vía Remote Desktop Connection.

### Demo Database

Entorno Rent Manager completamente funcional, pre-poblado con datos que imitan el uso real, usado para:
- Aprender el software.
- Probar la integración.
- Dar soporte a clientes.

### Cómo acceder al Sandbox

**Rent Manager Express**
1. Deberías haber recibido un email de `Ar@RentManager.com` con tu company code y detalles para programar la llamada de onboarding.
2. Deberías haber recibido un segundo email de `Sales@RentManager.com` con tus credenciales de Rent Manager (username: `Admin` + password fuerte único). Se puede usar "forgot password".
3. Ir a `companycode.rmx.rentmanager.com` (el company code está en el email del paso 1).
4. Ingresar las credenciales en la pantalla de login.

**Rent Manager 12**
1. Abrir la aplicación Remote Desktop Connection e ingresar credenciales.
2. Computer name: `companycode.rmo.rentmanager.com`; username: `companycode-username` (ej: `companycode-admin`).
3. Ir a `RentManager.com/Install`, elegir el sistema operativo y seguir las instrucciones.
4. Al ingresar, se pedirán las credenciales de server seat y las de Rent Manager (habrá que actualizar ambas passwords).

> Nota: las credenciales de server seat **no** son necesarias para Rent Manager Express.

### API & API Portal Access

**Overview de la API:**
- API REST basada en web.
- Formato de datos JSON.
- Soporta GET y POST.
- Soporta Webhooks.

**Acceso al API Portal:**
1. Ir a `companycode.api.rentmanager.com`.
2. Loguearse con las mismas credenciales del Rent Manager Sandbox.

**Métodos de autenticación — Partner Token:**
- Token estático que permite hacer llamadas a la API sin necesidad de username/password.
- Debe pasarse en el header de cada request: `X-RM12API-PartnerToken`.
- El request debe hacerse desde la **IP whitelisteada** designada en Partner Web Access (PWA).

Se recomienda usar el API Portal propio o una herramienta como **Postman** para aprender y familiarizarse con la API.

**Agregar privilegio de acceso API a un usuario:**
1. Ir a `companycode.rmx.rentmanager.com` con las credenciales de Rent Manager.
2. En el Command Launch, escribir "Users".
3. Seleccionar el usuario a editar.
4. Buscar el tile "Options" a la derecha.
5. Marcar el checkbox "API Access" y guardar.

### API & Database Structure — Locations

- Las **Locations** son bases de datos separadas para organizar y actualizar datos de negocio. Un cliente puede tener múltiples Locations, cada una con sus propios registros.
- Es importante entender qué Location(s) tiene creado el cliente.
- Durante el setup inicial, este request devuelve el/los Location ID(s) a los que la integración tiene acceso:
  ```
  companycode.api.rentmanager.com/Current/Locations
  ```
- Se puede especificar un Location ID en el header del request para obtener datos de esa Location. **Se recomienda** incluir el Location ID en el header para verificar dónde se están leyendo/escribiendo los datos.

> Dudas sobre cómo hacer un request: `APISupport@RentManager.com`.

---

## Learning Rent Manager

Distintas formas de aprender sobre Rent Manager:

- **Rent Manager University (RMU)**: plataforma de entrenamiento online con videos, actividades de eLearning, simulaciones y evaluaciones.
- **Rent Manager Help File**: documentación detallada con instrucciones paso a paso y privilegios de usuario requeridos por funcionalidad. Se accede con el ícono de "?" (Rent Manager Express) o con **F1** (Rent Manager 12).
- **Integrations Bootcamp**: serie de webinars educativos en vivo para optimizar la partnership. Sesiones disponibles:
  - PWA Core Session
  - Custom Layouts/Tabs Session
  - Service Issues Session
  - Unit Availability Session

---

## Rent Manager Features & Customization

### Custom Layouts, Tabs & UDFs

**User Defined Fields (UDFs):** campos personalizados para trackear información que Rent Manager no trackea por defecto. Cada entidad (tenant, prospect, property, unit, etc.) puede tener su propio set de UDFs; los valores son específicos por cuenta.

Tipos de UDF disponibles:
- Text
- Encrypted Text
- Dropdown List
- Selection List
- Yes/No
- Date
- Numeric
- File
- Image
- Hyperlink

**Ejemplo de uso:** trackear si un tenant debe pasar a cobranza mediante un dropdown Yes/No, disparando el proceso de cobranza si el valor es "Yes".

**Custom Tabs:** organizan los UDFs en una tab dentro de Rent Manager 12, además de sumar branding al producto. Se pueden subir a la Online Template Library para que los clientes los descarguen e instalen.

**Custom Layouts:** permiten personalizar cómo se muestra la página de detalle en ciertas áreas de Rent Manager Express. Los tiles pueden agregarse, eliminarse o reposicionarse; se pueden crear tiles en blanco. Los layouts pueden configurarse como default general o asignarse a roles/tipos de property específicos.

### Highlights

- **Entity Names**: por defecto Rent Manager usa nombres estándar para las entidades (ej. "tenants"), pero los clientes pueden renombrarlas (ej. "residents").
- **Reports/Custom Reports**: reportes built-in + Report Writer para crear reportes propios, con campos insertables y scripting.
- **Dashboard/Workplace**: el Dashboard muestra tiles configurables por rol/usuario (Dashboard Designer). "My Workspace" es lo primero que ve el usuario en Rent Manager Express, personalizado por usuario.
- **Menu Ribbon**: navegación de Rent Manager 12, personalizable agregando/quitando tabs de menú.
- **Scoreboards**: la mayoría de las ventanas de Rent Manager 12 tienen un Scoreboard con campos relacionados a la ventana, totalmente personalizable.

---

## Activating & Managing Your Integration

### Partner Web Access (PWA)

- Para hacer visible la integración en el marketplace de Rent Manager, hay que configurar el portal **PWA**.
- El **Available Integrations Marketplace** es la herramienta para que los clientes descubran la integración — es la **única** forma de que los clientes la activen.

### Qué se necesita para empezar

**Ítems técnicos:**
- IPs whitelisteadas
- Privilegios de usuario

**Ítems de marketing:**
- Descripción del producto y logo
- Teléfonos y email de Sales y Support

### Acceder a PWA

1. Ir a `PWA.RentManager.com`.
2. La primera vez, hacer clic en "Forgot Password?" para crear una contraseña.
3. El username es el email del contacto principal de la compañía.
4. Una vez logueado, el contacto principal puede agregar usuarios adicionales.

Recursos en video relacionados:
- Setting up & logging into PWA
- Applying user privileges to your integration
- Requesting/Activating & deactivating customers

---

## Supporting Rent Manager Customers

El equipo debe estar preparado no solo para desarrollar, sino también para vender, implementar y dar soporte a la integración.

> **Importante:** no incluir al cliente en los mensajes al equipo de Rent Manager, para reducir comunicación/confusión innecesaria.

### Contacto según el tipo de consulta

| Tipo de consulta | Contacto |
|---|---|
| Funcionalidad de Rent Manager | `Integrations@RentManager.com` (o formulario) |
| Preguntas de API (endpoints, detalles técnicos) | `APISupport@RentManager.com` (o formulario). Para errores en llamadas API, usar el link específico de soporte de errores. |

### Flujo de soporte

1. El cliente contacta a tu compañía.
2. Tu equipo hace el primer intento de resolver el problema.
3. Si no se puede resolver, tu equipo contacta al equipo correspondiente de Rent Manager por los medios indicados arriba.
4. Tu equipo trabaja directamente con Rent Manager en representación del cliente para resolver el problema.
5. Tu equipo se comunica con el cliente con la resolución.

---

## Marketing to Rent Manager Customers

Canales y oportunidades disponibles para promocionar la integración:

- **Newsletter/Periodical**: aparecer en la newsletter trimestral de Integrations (6,000+ clientes) vía guest blog, webinar Integrations Spotlight, case studies conjuntos, etc.
- **Beyond Rent Podcast**: podcast sobre tendencias y tecnología en property management.
- **Social Media**: cross-promoción de contenido relevante de la industria en las redes de Rent Manager.
- **Integrations Expo**: evento anual en las oficinas centrales de Rent Manager para exposición ante sus empleados.
- **Submit a Guest Blog**: artículos de blog enfocados en la industria, publicados en el sitio de Rent Manager (posiblemente destacados en la newsletter).
- **Integrations Spotlight**: webinar semanal para presentar tu solución a una audiencia en vivo de usuarios de Rent Manager.
- **Website Landing Page**: página de aterrizaje dedicada + formulario de contacto en la tab de Integrations de RentManager.com.
- **Integration Insights**: educar a los equipos de Sales y Customer Success de Rent Manager sobre tu compañía y el valor de tu integración.
- **Available Integration Demo**: crear una demo alojada en Rent Manager University (RMU).
- **Branding Guidelines**: guía de branding disponible.
- **Rent Manager User Conference**: evento presencial.

---

## Frequently Asked Questions

- ¿Dónde puedo obtener información sobre **Endpoints**?
- ¿Dónde puedo obtener información sobre **Locations**?
- ¿Dónde puedo obtener información sobre **Integrated Databases**?
- ¿Cómo gestiono clientes en el portal **PWA**?
- ¿Cómo elijo los privilegios relativos para la integración?
- ¿Cómo accedo a **Partner Web Access (PWA)** y creo nuevos usuarios?
- Tengo problemas para loguearme en mi cuenta **Sandbox**. ¿Cómo lo soluciono?
- ¿Cuál es la diferencia entre **Rent Manager Express** y **Rent Manager 12**?

Recursos en video (Integrations Bootcamp):
- Unit Availability
- Service Issues
- Custom Tabs & Custom Layouts
- PWA Core Session
