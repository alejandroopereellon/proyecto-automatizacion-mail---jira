# Automatización de incidencias por correo con IA + Jira (n8n)

Repositorio con **2 workflows de n8n** orientados a un entorno real de soporte técnico:

1. **Workflow principal**: procesa correos entrantes, filtra mensajes automáticos, clasifica con IA y crea/actualiza incidencias en Jira.  
2. **Workflow auxiliar**: lista proyectos de Jira para obtener `id`, `key` y `name` (útil para configurar el mapeo dominio → proyecto).

Diseñado para ser **modular, reutilizable y fácil de adaptar** a otras organizaciones.

---

## Workflows incluidos

### 1) Workflow principal — Gestión automática de incidencias

**Objetivo:** automatizar el flujo desde correo → clasificación IA → Jira

**Qué hace (alto nivel):**

- Lee correos desde un buzón (IMAP).
- Filtra correos:
  - solo si el buzón está en **To** (destinatario directo),
  - y si el remitente **no** es un correo tipo **no-reply**.
- Extrae el **dominio** del remitente (después del `@`).
- Consulta en **Postgres** el destino (dominio → proyecto Jira).
- Llama a IA para:
  - determinar si es incidencia,
  - detectar reenvío/respuesta,
  - generar un resumen y descripción para uso interno.
- Si es **incidencia**: crea ticket en Jira.
- Si es **respuesta** y no hay ID claro: solicita ID al técnico y añade comentario.

---

### 2) Workflow auxiliar — Obtener proyectos de Jira (ID / KEY / NAME)

**Objetivo:** obtener rápidamente la lista de proyectos disponibles en Jira para identificar el `key` (y opcionalmente `id`).

Realiza:
- `GET /rest/api/3/project`

Devuelve campos útiles (`id`, `key`, `name`) agregados para revisión rápida.

---

## Requisitos

- n8n (self-hosted o cloud)
- Buzón IMAP (lectura)
- SMTP (envío de notificaciones / formularios)
- Jira Cloud (credenciales)
- Postgres (mapeo dominio → proyecto)
- API key de OpenAI (clasificación y resumen)

---

## Instalación / Importación en n8n

1. Descarga los archivos `.json` de este repositorio.
2. En n8n: **Workflows → Import from File**.
3. Crea tus **credenciales** (IMAP, SMTP, Jira, Postgres, OpenAI).
4. Ajusta los nodos de configuración (ver apartado “Configuración”).

---

## Configuración

### A) Correos de control (buzón y no-reply)

En el workflow principal existe un nodo tipo **Set** con:

- `email_buzon`: correo del buzón que debe estar en **To**
- `email_no-reply`: remitente automático que quieres excluir

**Ejemplo:**

- `email_buzon = support@empresa.com`
- `email_no-reply = no-reply@empresa.com`

---

### B) Base de datos (Postgres): dominio → proyecto Jira

Tabla recomendada:

```
CREATE TABLE jira_destino (
  dominio TEXT PRIMARY KEY,
  jira_project_key TEXT NOT NULL
);
```

Ejemplo de inserción:

```
INSERT INTO jira_destino (dominio, jira_project_key)
VALUES ('empresa.com', 'KAN');
```

Nota: normalmente basta con `jira_project_key` (por ejemplo `KAN`) para crear incidencias en Jira.

---

### C) Jira (credenciales)

Configura credenciales de Jira Cloud en n8n (usuario + API Token o credencial nativa).

Asegúrate de que el nodo de creación de incidencias está apuntando al proyecto correcto (por `key`).

---

### D) OpenAI (clasificación)

La IA se utiliza para:

- clasificar si el correo es incidencia,
- detectar reenvío o respuesta,
- generar resumen y descripción estructurados.

La IA **no decide** prioridades, SLA, estados ni asignaciones.

---

## Workflow auxiliar — URL correcta del HTTP Request

En el nodo **HTTP Request** usa:

https://TU_DOMINIO.atlassian.net/rest/api/3/project

Si lo construyes por variable (recomendado):

- Variable base: `https://TU_DOMINIO.atlassian.net/`

URL final:

{{ $json.url_atlassian }}rest/api/3/project

Importante: evita espacios al final de la URL.

---

## Limitaciones conocidas

- No gestiona adjuntos.
- No implementa deduplicación de incidencias por sí solo.
- No calcula SLA, prioridad ni asignaciones automáticas (se deja a Jira o al técnico).

---

## Seguridad y publicación en GitHub

- Los workflows exportados por n8n **no incluyen contraseñas ni tokens**.
- Los IDs de credenciales en el JSON son internos de tu instancia y **no son reutilizables**.

**Recomendación antes de publicar:**

- Eliminar `pinData` del JSON.
- Reemplazar valores reales por placeholders (`TU_DOMINIO`, `support@empresa.com`, etc.).

---

## Autor

Proyecto desarrollado por **Alejandro Perellón**

- LinkedIn: https://www.linkedin.com/in/alejandro-perellón-lópez-180746278  
- Web: https://alejandroperellon.es
