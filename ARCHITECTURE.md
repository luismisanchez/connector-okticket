# Arquitectura del Conector OkTicket para Odoo

## Índice

1. [Visión General](#visión-general)
2. [Estructura del Proyecto](#estructura-del-proyecto)
3. [Módulos](#módulos)
4. [Conexión con la API de OkTicket](#conexión-con-la-api-de-okticket)
5. [Patrón de Arquitectura: Connector Framework](#patrón-de-arquitectura-connector-framework)
6. [Flujo de Datos](#flujo-de-datos)
7. [Modelos de Datos](#modelos-de-datos)
8. [Seguridad](#seguridad)
9. [Trabajos Programados (Cron Jobs)](#trabajos-programados-cron-jobs)
10. [Dependencias](#dependencias)

---

## Visión General

Este proyecto es un **conector de Odoo 15** que integra la plataforma de gestión de gastos **OkTicket** con el ERP Odoo. Permite la sincronización bidireccional de gastos, empleados, centros de coste, productos y hojas de gastos entre ambos sistemas.

- **Autor:** Alia Technologies, S.L.
- **Licencia:** AGPL-3.0
- **Versión de Odoo:** 15.0
- **Lenguaje:** Python

---

## Estructura del Proyecto

```
connector-okticket/
├── okticket_connector/                              # Módulo principal del conector
│   ├── okticket/                                    # Capa de comunicación HTTP con la API
│   │   ├── base_connector.py                        # Cliente HTTP base (peticiones, paginación, errores)
│   │   ├── ticket_connector.py                      # Conector OAuth 2.0 específico de OkTicket
│   │   └── exceptions.py                            # Jerarquía de excepciones del conector
│   ├── components/                                  # Componentes del framework Connector
│   │   ├── core.py                                  # Componente base del conector
│   │   ├── backend_adapter.py                       # Adaptador para comunicación con la API
│   │   ├── binder.py                                # Enlazador de registros Odoo ↔ OkTicket
│   │   └── mapper.py                                # Mapeadores de datos (import/export)
│   ├── models/                                      # Modelos de Odoo
│   │   ├── okticket_backend.py                      # Configuración del backend (credenciales, URLs)
│   │   ├── okticket_binding.py                      # Modelo abstracto de binding
│   │   ├── okticket_hr_expense.py                   # Binding de gastos
│   │   ├── okticket_hr_expense_importer.py          # Importador y mapeador de gastos
│   │   ├── okticket_hr_employee.py                  # Binding de empleados
│   │   ├── okticket_analytic.py                     # Binding de cuentas analíticas
│   │   ├── okticket_product.py                      # Extensión de productos
│   │   ├── okticket_company.py                      # Extensión de compañía
│   │   ├── hr_expense.py                            # Extensión del modelo hr.expense
│   │   ├── log_event.py                             # Modelo de log de eventos
│   │   └── res_users.py                             # Extensión de usuarios
│   ├── views/                                       # Vistas XML de Odoo
│   ├── data/                                        # Datos iniciales (crons, productos)
│   ├── security/                                    # Control de acceso
│   └── wizard/                                      # Asistentes
│
├── okticket_connector_cost_center/                  # Sincronización de centros de coste
├── okticket_connector_hr_expense_sheet/              # Hojas de gastos
├── okticket_connector_hr_expense_sheet_grouping/     # Agrupación de gastos
├── okticket_connector_product_synchronization/       # Sincronización de productos
├── okticket_connector_user_synchronization/          # Sincronización de usuarios
├── okticket_hr_expense_reporting/                   # Informes de gastos
├── okticket_hr_timesheet_cost_center/               # Partes de horas y centros de coste
│
├── requirements.txt                                 # Dependencias Python (cachetools==3.1.1)
├── oca_dependencies.txt                             # Dependencias OCA
└── README.md                                        # Documentación de usuario
```

---

## Módulos

### 1. `okticket_connector` — Módulo Principal

El núcleo del conector. Contiene toda la lógica base de comunicación con la API, los modelos de binding, el importador de gastos y la configuración del backend.

**Responsabilidades:**
- Cliente HTTP y autenticación OAuth 2.0
- Modelo de configuración del backend (`okticket.backend`)
- Importación de gastos desde OkTicket
- Modelos de binding (enlace entre IDs de OkTicket e IDs de Odoo)
- Sistema de logging de eventos (`log.event`)
- Extensiones de modelos de Odoo (`hr.expense`, `product.template`, `res.company`)

### 2. `okticket_connector_cost_center` — Centros de Coste

Sincroniza cuentas analíticas de Odoo como centros de coste en OkTicket.

**Depende de:** `okticket_connector`, `project`
**Componentes:** Binder, Exporter
**Flujo:** Odoo → OkTicket (exportación)

### 3. `okticket_connector_hr_expense_sheet` — Hojas de Gastos

Gestiona la sincronización bidireccional de hojas de gastos (reports) entre Odoo y OkTicket.

**Depende de:** `okticket_connector`, `sale_timesheet`, `hr_expense`
**Componentes:** Binder, Exporter, Listener
**Flujo:** Bidireccional (import/export)

### 4. `okticket_connector_hr_expense_sheet_grouping` — Agrupación de Gastos

Extiende la funcionalidad de hojas de gastos con estrategias configurables de agrupación: estándar, gasto único y sin agrupación.

**Depende de:** `okticket_connector`, `okticket_connector_hr_expense_sheet`

### 5. `okticket_connector_product_synchronization` — Sincronización de Productos

Importa categorías de productos desde OkTicket a Odoo mediante cron job programado.

**Depende de:** `okticket_connector`
**Componentes:** Binder, Importer
**Flujo:** OkTicket → Odoo (importación)

### 6. `okticket_connector_user_synchronization` — Sincronización de Usuarios

Importa usuarios desde OkTicket y los vincula con empleados de Odoo por email.

**Depende de:** `okticket_connector`, `hr`
**Componentes:** Binder, Importer, Listener
**Flujo:** OkTicket → Odoo (importación)

### 7. `okticket_hr_expense_reporting` — Informes

Proporciona plantillas de informes para gastos y hojas de gastos.

**Depende de:** `okticket_connector_hr_expense_sheet`, `web`

### 8. `okticket_hr_timesheet_cost_center` — Partes de Horas

Sincroniza automáticamente centros de coste en OkTicket cuando se crean proyectos desde partes de horas.

**Depende de:** `hr_timesheet`, `okticket_connector_cost_center`

---

## Conexión con la API de OkTicket

### Autenticación: OAuth 2.0

La conexión con la API de OkTicket utiliza el protocolo **OAuth 2.0** con credenciales de tipo `password grant`.

```
┌──────────┐                              ┌──────────────┐
│   Odoo   │  POST /api/public/oauth/token│  OkTicket    │
│ (Client) │ ─────────────────────────────>│    API       │
│          │  {grant_type, client_id,      │              │
│          │   client_secret, username,    │              │
│          │   password, scope}            │              │
│          │ <─────────────────────────────│              │
│          │  {access_token, token_type}   │              │
│          │                              │              │
│          │  GET /expenses               │              │
│          │  Authorization: Bearer <tok> │              │
│          │ ─────────────────────────────>│              │
│          │ <─────────────────────────────│              │
│          │  {data: [...], links: {...}} │              │
└──────────┘                              └──────────────┘
```

**Parámetros de autenticación** (configurados en `okticket.backend`):

| Parámetro | Descripción |
|-----------|-------------|
| `http_client_conn_url` | URL del servidor HTTP (ej: `dev.okticket.es`) |
| `base_url` | URL base de la API (ej: `/api/public`) |
| `auth_uri` | Ruta de autenticación OAuth (ej: `/oauth/token`) |
| `uri_op_path` | Ruta de operaciones de la API |
| `api_login` | Nombre de usuario de la API |
| `api_password` | Contraseña de la API |
| `grant_type` | Tipo de concesión OAuth (ej: `password`) |
| `oauth_client_id` | ID del cliente OAuth |
| `oauth_secret` | Secreto del cliente OAuth |
| `scope` | Alcance de permisos OAuth |
| `okticket_company_id` | ID de la compañía en OkTicket |
| `https` | Usar protocolo HTTPS (boolean) |

### Flujo de Autenticación

1. `OkTicketOpenConnector` se instancia con los parámetros del backend
2. Se llama a `login()` que envía un POST a `base_url + auth_uri`
3. La respuesta contiene `token_type` y `access_token`
4. Estos tokens se usan en las cabeceras `Authorization` de las siguientes peticiones
5. Si una petición recibe un error 401 (AuthError), se reautentica automáticamente

### Endpoints de la API

| Endpoint | Método | Descripción | Módulo que lo usa |
|----------|--------|-------------|-------------------|
| `/api/public/oauth/token` | POST | Autenticación OAuth 2.0 | `okticket_connector` |
| `/expenses` | GET | Lista de gastos (con paginación) | `okticket_connector` |
| `/expenses/{id}` | GET | Detalle de un gasto específico | `okticket_connector` |
| `/reports` | GET | Hojas de gastos/informes | `okticket_connector_hr_expense_sheet` |
| `/reports/{id}` | GET | Detalle de un informe | `okticket_connector` |
| `/cost-centers` | GET | Lista de centros de coste | `okticket_connector_cost_center` |
| `/users` | GET | Lista de usuarios | `okticket_connector_user_synchronization` |
| `/categories` | GET | Categorías de productos | `okticket_connector_product_synchronization` |

### Gestión de Paginación

La API de OkTicket devuelve respuestas paginadas con un campo `links.next`. El método `process_request()` en `BaseConnector` maneja automáticamente la paginación:

```python
# Recorre todas las páginas automáticamente
if response['result'].get('links'):
    while response['result']['links'].get('next'):
        conn.close()
        conn = self.get_http_connection(https=https)
        next_url = response['result']['links']['next']
        response = self.request_base(next_url, ...)
        result = result + response['result'].get('data')
```

### Gestión de Errores

La jerarquía de excepciones maneja los códigos HTTP de la API:

| Código HTTP | Excepción | Acción |
|-------------|-----------|--------|
| 200, 201 | — | Respuesta exitosa |
| 204 | — | Eliminación exitosa |
| 401 | `AuthError` | Reautenticación automática |
| 403 | `ForbiddenError` | Recurso prohibido |
| 404 | `ResourceNotFoundError` | Recurso no encontrado |
| 409 | `ConflictError` | Conflicto de versión |
| 413 | `RequestEntityTooLargeError` | Petición demasiado grande |
| 422 | `ValidationError` | Errores de validación |
| 500 | `ServerError` | Error interno del servidor |

---

## Patrón de Arquitectura: Connector Framework

El proyecto sigue el patrón del **Odoo Connector Framework** (OCA), que define componentes reutilizables para integrar sistemas externos.

```
┌────────────────────────────────────────────────────────────┐
│                    okticket.backend                         │
│              (Configuración y credenciales)                 │
└─────────────────────┬──────────────────────────────────────┘
                      │ work_on()
                      ▼
┌────────────────────────────────────────────────────────────┐
│              Componentes del Conector                       │
│                                                            │
│  ┌─────────────┐  ┌──────────┐  ┌────────┐  ┌──────────┐ │
│  │  Adapter    │  │  Mapper  │  │ Binder │  │ Importer │ │
│  │ (HTTP API)  │  │ (Trans-  │  │ (ID    │  │/Exporter │ │
│  │             │  │  form)   │  │  link) │  │ (Orches- │ │
│  │             │  │          │  │        │  │  trator)  │ │
│  └──────┬──────┘  └────┬─────┘  └───┬────┘  └────┬─────┘ │
│         │              │            │             │        │
└─────────┼──────────────┼────────────┼─────────────┼────────┘
          │              │            │             │
          ▼              ▼            ▼             ▼
   ┌──────────┐   ┌──────────┐ ┌──────────┐ ┌──────────┐
   │ OkTicket │   │  Datos   │ │ Binding  │ │  Modelo  │
   │   API    │   │ mapeados │ │  Odoo ↔  │ │   Odoo   │
   │          │   │          │ │ OkTicket │ │          │
   └──────────┘   └──────────┘ └──────────┘ └──────────┘
```

### Componentes Principales

#### 1. Backend Adapter (`components/backend_adapter.py`)
Responsable de la comunicación directa con la API de OkTicket. Gestiona la autenticación y las peticiones HTTP.

```python
class OkticketAdapter(Component):
    _name = 'okticket.adapter'
    _usage = 'backend.adapter'

    def _auth(self):
        # Lee credenciales del backend → crea OkTicketOpenConnector → login()
        okticket_api = OkTicketOpenConnector(params=auth_data)
        okticket_api.login(https=self.backend_record.https)
```

#### 2. Binder (`components/binder.py`)
Enlaza los IDs externos de OkTicket con los IDs internos de Odoo. Cada modelo de binding (ej: `okticket.hr.expense`) almacena esta relación.

```python
class OkticketExpenseBinder(Component):
    _name = 'okticket.expense.binder'
    _apply_on = ['okticket.hr.expense']
    _usage = 'binder'
```

#### 3. Mapper (`components/mapper.py`)
Transforma los datos entre el formato de la API de OkTicket y el formato de los modelos de Odoo usando decoradores `@mapping`.

```python
class OkticketImportMapper(AbstractComponent):
    _name = 'okticket.import.mapper'
    _usage = 'import.mapper'
```

#### 4. Importer (`models/okticket_hr_expense_importer.py`)
Orquesta el proceso de importación: llama al adapter para obtener datos, los mapea y los crea/actualiza en Odoo.

```python
class HrExpenseBatchImporter(Component):
    _name = 'okticket.expenses.batch.importer'
    _usage = 'importer'

    def run(self, filters=None, options=None):
        adapter = self.component(usage='backend.adapter')
        mapper = self.component(usage='importer')
        binder = self.component(usage='binder')
        # ...
```

#### 5. Componente Base (`components/core.py`)
Define el componente base del cual heredan todos los componentes del conector, vinculándolos a la colección `okticket.backend`.

```python
class BaseOkticketConnectorComponent(AbstractComponent):
    _name = 'base.okticket.connector'
    _inherit = 'base.connector'
    _collection = 'okticket.backend'
```

---

## Flujo de Datos

### Importación de Gastos (OkTicket → Odoo)

```
1. Cron job ejecuta _scheduler_import_expenses() cada 2 horas
         │
2. Para cada backend → import_expenses()
         │
3. okticket.hr.expense.import_batch(backend)
         │
4. backend.work_on('okticket.hr.expense')
         │
5. Importer.run()
   ├── Adapter._auth() → Login OAuth 2.0
   ├── Adapter.search() → GET /expenses (con filtros de fecha)
   │
   ├── Para cada gasto recibido:
   │   ├── Binder.to_internal() → ¿Existe ya en Odoo?
   │   ├── Mapper.map_record() → Transformar datos OkTicket → Odoo
   │   │   ├── @mapping name → nombre del gasto
   │   │   ├── @mapping product_id → producto según type_id y category_id
   │   │   ├── @mapping employee_id → empleado por okticket_user_id
   │   │   ├── @mapping amount → importe total
   │   │   ├── @mapping date → fecha del gasto
   │   │   ├── @mapping analytic_account_id → centro de coste
   │   │   ├── @mapping okticket_img → imagen del ticket (base64)
   │   │   └── ... (15+ campos mapeados)
   │   │
   │   ├── Si existe → binding.write(internal_data)
   │   ├── Si no existe → model.create(internal_data)
   │   └── Binder.bind() → Vincular IDs
   │
   └── Actualizar import_expenses_since
```

### Exportación de Centros de Coste (Odoo → OkTicket)

```
1. Listener detecta creación/modificación de cuenta analítica
         │
2. Exporter prepara datos con Mapper
         │
3. Adapter envía POST/PUT a la API de OkTicket
```

### Mapeo de Campos de Gastos (OkTicket → Odoo)

| Campo OkTicket | Campo Odoo | Lógica |
|----------------|------------|--------|
| `_id` | `external_id` | ID externo directo |
| `name` / `ticket_num` | `name` | Nombre del gasto |
| `type_id` + `category_id` | `product_id` | Búsqueda de producto por tipo y categoría |
| `amount` | `total_amount` | Importe del gasto |
| `date` | `date` | Conversión de formato datetime |
| `comments` | `description` | Descripción del gasto |
| `user_id` | `employee_id` | Búsqueda de empleado por `okticket_user_id` |
| `company_id` | `company_id` | Búsqueda de compañía por `okticket_company_id` |
| `status_id` | `okticket_status` | 1 = confirmed, otro = pending |
| `cif` | `okticket_vat` | NIF/CIF del proveedor |
| `remote_uri` | `okticket_img` | Descarga de imagen como base64 |
| `cost_center_id` | `analytic_account_id` | Búsqueda por binding de centro de coste |
| `payment_method` | `payment_mode` | efectivo → own_account, otro → company_account |
| `custom_fields.refacturable` | producto rebillable | Selección de versión refacturable del producto |
| `ticket_num` | `reference` | Referencia del ticket |

---

## Modelos de Datos

### Modelo de Binding Abstracto (`okticket.binding`)

Todos los modelos de binding heredan de `external.binding` e incluyen:
- `backend_id`: Referencia al backend de OkTicket
- `external_id`: ID del registro en OkTicket
- Restricción SQL de unicidad: `(backend_id, external_id)`

### Modelos de Binding Concretos

| Modelo de Binding | Modelo Odoo | Descripción |
|-------------------|-------------|-------------|
| `okticket.hr.expense` | `hr.expense` | Gastos profesionales |
| `okticket.hr.employee` | `hr.employee` | Empleados/usuarios |
| `okticket.account.analytic.account` | `account.analytic.account` | Centros de coste |
| `okticket.product.template` | `product.template` | Productos/categorías |
| `okticket.hr.expense.sheet` | `hr.expense.sheet` | Hojas de gastos |

### Modelo de Backend (`okticket.backend`)

Almacena toda la configuración de conexión con OkTicket:
- Credenciales OAuth 2.0
- URLs de la API
- Configuración de importación (fecha desde, solo revisados, etc.)
- Log de eventos

### Modelo de Log de Eventos (`log.event`)

Registra todas las operaciones realizadas con la API:
- Tipo: info, success, warning, error
- Tag: AUTH, GET, POST, PUT, PATCH, DELETE, OP
- Contenido del mensaje
- Fecha/hora del evento

### Extensiones de Productos

Los productos en Odoo se extienden con campos específicos de OkTicket:
- `okticket_categ_prod_id`: ID de categoría en OkTicket
- `okticket_type_prod_id`: Tipo (0=Gasto, 1=Factura, 2=Kilómetros)
- `rebillable_prod_id`: Referencia al producto refacturable
- `invoiceable_prod_id`: Referencia al producto facturable

---

## Seguridad

- Los campos sensibles (credenciales, tokens OAuth) están restringidos al grupo `connector.group_connector_manager`
- Se define un control de acceso por roles en `security/ir.model.access.csv`
- Se define un grupo de seguridad específico en `security/okticket_connector_security.xml`

---

## Trabajos Programados (Cron Jobs)

| Cron Job | Intervalo | Descripción |
|----------|-----------|-------------|
| Importar gastos | Cada 2 horas | `_scheduler_import_expenses()` |
| Importar productos | Configurable | `_scheduler_import_products()` |
| Importar empleados | Configurable | `_scheduler_import_employees()` |

---

## Dependencias

### Módulos Odoo Requeridos
- `hr_expense`, `hr`, `hr_timesheet` — Gestión de RRHH y gastos
- `sale_expense`, `sale_timesheet` — Ventas y partes de horas
- `product`, `analytic`, `uom` — Productos y contabilidad analítica
- `connector`, `component`, `component_event`, `queue_job` — Framework OCA Connector
- `project` — Gestión de proyectos (para centros de coste)
- `web` — Interfaz web (para informes)

### Dependencias Python
- `cachetools==3.1.1` — Librería de caché
- `requests` — Cliente HTTP (para descarga de imágenes de tickets)

### Dependencias OCA
Definidas en `oca_dependencies.txt`:
- `cachetools==3.1.1`
