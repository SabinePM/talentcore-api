# TalentCore API

Microservicio REST para gestionar empleados y solicitudes de vacaciones, hecho con **FastAPI** y **SQLite**. Calcula los días de vacaciones según la antigüedad, permite solicitar días libres y exige que el **jefe directo** apruebe o rechace cada solicitud.

Es un proyecto académico: el código fue generado por una cadena de 6 agentes de IA (Product Owner, Arquitecto, Desarrollador, QA, Revisor de seguridad y DevOps) y después probado, auditado y publicado paso a paso.

## Qué hace

- Registra empleados con un rol y, opcionalmente, un jefe directo.
- Calcula el saldo de vacaciones según los años de antigüedad.
- Permite solicitar días y rechaza las solicitudes que superan el saldo.
- Permite que solo el jefe directo apruebe o rechace una solicitud.

## Reglas de negocio

**Roles**

| Rol | Qué puede hacer |
|---|---|
| `EMPLEADO` | Consultar su saldo y solicitar vacaciones |
| `MANAGER` | Aprobar o rechazar solicitudes de sus subordinados directos |
| `ADMIN_RRHH` | Crear empleados |

**Días de vacaciones por antigüedad**

```
dias_totales = min(12 + 2 * (anios_antiguedad - 1), 20)     # desde el 1er año
dias_totales = 12                                           # si tiene menos de 1 año
saldo_disponible = dias_totales - dias_aprobados
```

Solo las solicitudes `APPROVED` descuentan saldo.

**Estados de una solicitud:** `PENDING` (al crearla), `APPROVED` y `REJECTED`. Solo se puede revisar una solicitud que siga en `PENDING`.

**Criterios de aceptación**

| Código | Escenario | Respuesta |
|---|---|---|
| AC-01 | Solicitud válida con saldo suficiente | `201 Created`, estado `PENDING` |
| AC-02 | Pide más días que su saldo | `400 Bad Request`, `Insufficient vacation balance` |
| AC-03 | Un manager revisa a alguien que no es su subordinado | `403 Forbidden`, `Unauthorized: Not the direct manager` |

## Tecnologías

- Python, FastAPI y Uvicorn
- SQLAlchemy y SQLite
- Pydantic v2
- Pytest y httpx
- Docker y Docker Compose
- GitHub Actions (integración continua)

## Estructura del proyecto

```
.
├── .github/workflows/ci.yml   # Pruebas automáticas en cada push
├── app/
│   ├── __init__.py
│   ├── main.py                # Endpoints y reglas de negocio
│   └── models.py              # Base de datos, modelos y esquemas
├── tests/
│   ├── __init__.py
│   └── test_api.py            # Pruebas con pytest
├── Dockerfile
├── docker-compose.yml
├── probar_api.py              # Script que prueba la API en vivo
├── requirements.txt
└── README.md
```

## Cómo ejecutarlo en tu computadora

1. Crea y activa un entorno virtual (en Windows PowerShell):

```powershell
python -m venv venv
.\venv\Scripts\Activate.ps1
```

2. Instala las dependencias:

```powershell
pip install -r requirements.txt
```

3. Enciende la API:

```powershell
uvicorn app.main:app --reload
```

4. Abre la documentación interactiva (Swagger) en: <http://127.0.0.1:8000/docs>

## Endpoints

| Método | Ruta | Descripción |
|---|---|---|
| `GET` | `/health` | Comprueba que el servicio está vivo (público) |
| `POST` | `/api/v1/empleados` | Crea un empleado |
| `GET` | `/api/v1/vacaciones/saldo/{empleado_id}` | Consulta el saldo de vacaciones |
| `POST` | `/api/v1/vacaciones/solicitar` | Crea una solicitud en estado `PENDING` |
| `POST` | `/api/v1/vacaciones/{solicitud_id}/revisar` | Aprueba (`APPROVE`) o rechaza (`REJECT`) |

**Identificación:** quien hace la petición se indica con el header `X-User-Id` (el id del empleado).

**Cómo crear al primer administrador:** si la tabla de empleados está vacía, `POST /api/v1/empleados` permite crear el primer `ADMIN_RRHH` sin header. Después, solo un `ADMIN_RRHH` puede crear empleados.

## Pruebas

Pruebas automáticas (4 pruebas con pytest):

```powershell
pytest -v
```

Prueba en vivo, con el servidor encendido en otra terminal. Empieza con una base de datos vacía (borra `talentcore.db` antes de usarlo):

```powershell
python probar_api.py
```

## Con Docker (opcional)

Requiere tener Docker Desktop instalado:

```powershell
docker compose up --build
```

La API quedará disponible en <http://127.0.0.1:8000/docs>.

## Integración continua

El archivo `.github/workflows/ci.yml` ejecuta `pytest -v` en GitHub Actions con cada `push` o `pull_request` a la rama `main`.

## Despliegue

La API se publicó en **Render** (plan gratuito) con estos comandos:

| Campo | Valor |
|---|---|
| Build Command | `pip install --no-cache-dir -r requirements.txt` |
| Start Command | `uvicorn app.main:app --host 0.0.0.0 --port $PORT` |

> En el plan gratuito el servicio se duerme tras un rato sin visitas (la primera petición puede tardar cerca de un minuto) y la base SQLite se reinicia vacía al despertar.

## Limitaciones conocidas

Este proyecto es de práctica y **no está listo para producción**. La auditoría de seguridad señaló, entre otros puntos:

- **Identidad por header:** `X-User-Id` lo escribe quien hace la petición, así que cualquiera podría hacerse pasar por otro usuario. Para producción haría falta autenticación con tokens firmados (JWT).
- **Solicitudes a nombre de otros:** en `POST /api/v1/vacaciones/solicitar` el `empleado_id` viaja en el cuerpo y no se compara con quien hace la petición (vulnerabilidad BOLA).
- **Base de datos:** SQLite sin migraciones. Para producción convendría usar una base de datos persistente y migraciones con Alembic.

## Autoría

Proyecto académico de práctica de ingeniería de software con agentes de IA. GitHub: [@SabinePM](https://github.com/SabinePM).
