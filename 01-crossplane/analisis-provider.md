# =====================================================================
# Análisis del Provider PostgreSQL
# =====================================================================

## Provider: tages/provider-postgresql v0.1.0

### 1. Managed Resources disponibles


El Provider registró 17 tipos en el clúster, de los cuales **14 son Managed Resources**.
Los tres restantes —`ProviderConfig`, `ProviderConfigUsage` y `StoreConfig`— pertenecen al
grupo `postgresql.upbound.io` y no representan objetos de PostgreSQL: son la maquinaria
interna con la que Crossplane gestiona la conexión al sistema externo.

### Objetos estructurales

| Kind | Grupo | Propósito |
|---|---|---|
| `Database` | `postgresql.postgresql.upbound.io` | Crea una base de datos en el servidor |
| `Schema` | `postgresql.postgresql.upbound.io` | Crea un esquema dentro de una base de datos |
| `Extension` | `postgresql.postgresql.upbound.io` | Habilita extensiones (pgcrypto, postgis, uuid-ossp) |
| `Function` | `postgresql.postgresql.upbound.io` | Define funciones almacenadas |

### Identidad y control de acceso

| Kind | Grupo | Propósito |
|---|---|---|
| `Role` | `postgresql.postgresql.upbound.io` | Crea un rol o usuario de PostgreSQL |
| `Role` | `grant.postgresql.upbound.io` | Otorga la pertenencia de un rol dentro de otro rol |
| `Grant` | `postgresql.postgresql.upbound.io` | Concede privilegios sobre objetos existentes |
| `Privileges` | `default.postgresql.upbound.io` | Define privilegios por defecto para objetos futuros |

> **Nota de diseño:** existen dos kinds llamados `Role` en grupos distintos. El primero
> crea la identidad; el segundo gestiona membresías entre roles. La distinción solo es
> visible por el grupo de API, lo que constituye una fuente de error al escribir
> Compositions.

### Replicación

| Kind | Grupo | Propósito |
|---|---|---|
| `Publication` | `postgresql.postgresql.upbound.io` | Publica un conjunto de tablas para replicación lógica |
| `Subscription` | `postgresql.postgresql.upbound.io` | Se suscribe a una publicación de otro servidor |
| `ReplicationSlot` | `physical.postgresql.upbound.io` | Slot de replicación física |
| `Slot` | `replication.postgresql.upbound.io` | Slot de replicación lógica |

### Acceso a datos externos (Foreign Data Wrappers)

| Kind | Grupo | Propósito |
|---|---|---|
| `Server` | `postgresql.postgresql.upbound.io` | Declara un servidor externo accesible vía FDW |
| `Mapping` | `user.postgresql.upbound.io` | Mapea un usuario local a credenciales del servidor externo |

---


### 2. Campos requeridos del recurso Database
### Campos requeridos por el schema

**Ninguno.** El array `required` del OpenAPI schema está vacío:

```bash
kubectl get crd databases.postgresql.postgresql.upbound.io \
  -o jsonpath="{.spec.versions[0].schema.openAPIV3Schema.properties.spec.properties.forProvider.required}"
# (sin salida)
```

Esto ocurre porque el Provider fue generado a partir del provider de Terraform para
PostgreSQL, donde todos los atributos tienen valor por defecto. En la práctica, sin
embargo, el campo `name` sí es funcionalmente necesario: si se omite, Crossplane deriva
el nombre externo de la anotación `crossplane.io/external-name`, que a su vez toma por
defecto el `metadata.name` del recurso.

### Campos opcionales

| Campo | Tipo | Descripción | Inmutable |
|---|---|---|---|
| `name` | string | Nombre de la base de datos, único en el servidor | No |
| `owner` | string | Rol propietario de la base de datos | No |
| `template` | string | Base de datos plantilla de origen (por defecto `template0`) | Sí |
| `encoding` | string | Codificación de caracteres (por defecto `UTF8`) | Sí |
| `lcCollate` | string | Orden de colación `LC_COLLATE` (por defecto `C`) | Sí |
| `lcCtype` | string | Clasificación de caracteres `LC_CTYPE` (por defecto `C`) | Sí |
| `tablespaceName` | string | Tablespace asociado a la base de datos | No |
| `connectionLimit` | number | Conexiones concurrentes máximas (`-1` = sin límite) | No |
| `allowConnections` | boolean | Permite o bloquea conexiones (por defecto `true`) | No |
| `isTemplate` | boolean | Permite clonar la base de datos con privilegio CREATEDB | No |

Los campos marcados como inmutables solo pueden fijarse en el momento de creación:
modificarlos fuerza la destrucción y recreación del recurso. Es un detalle relevante para
GitOps, donde un cambio aparentemente menor en Git puede desencadenar una operación
destructiva durante la reconciliación.

---

### 3. Información requerida por el ProviderConfig


El `ProviderConfig` no almacena credenciales: **declara de dónde obtenerlas**.

```yaml
apiVersion: postgresql.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: postgresql-config
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: postgresql-credentials
      key: connection
```

| Campo | Valor | Función |
|---|---|---|
| `source` | `Secret` | Indica que las credenciales provienen de un Secret de Kubernetes. Es sensible a mayúsculas. |
| `secretRef.namespace` | `crossplane-system` | Namespace donde vive el Secret |
| `secretRef.name` | `postgresql-credentials` | Nombre del Secret |
| `secretRef.key` | `connection` | Clave dentro del Secret que contiene el JSON de conexión |

El Secret referenciado contiene un único valor JSON con seis parámetros:

| Parámetro | Valor en esta PoC | Función |
|---|---|---|
| `host` | `postgresql.postgresql.svc.cluster.local` | DNS interno del servicio de PostgreSQL |
| `port` | `5432` | Puerto de escucha |
| `username` | `postgres` | Usuario administrador, con privilegios para crear bases de datos |
| `password` | *(en el Secret)* | Contraseña del usuario administrador |
| `database` | `postgres` | Base de datos inicial de conexión |
| `sslmode` | `disable` | Sin TLS, aceptable únicamente en entorno local |
