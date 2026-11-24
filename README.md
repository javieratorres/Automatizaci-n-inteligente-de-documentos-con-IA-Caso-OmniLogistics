# Automatización inteligente de documentos: OmniLogistics (Make) y laboratorio n8n

Proyecto del **Bootcamp Construcción de Agentes IA y Automatización de Tareas** (Skillnest y Banco de Chile).

**Autores:** Javiera Torres, Michille Taffo, Matías Orgeta y Neil Meza

## Descripción

Sistema que lee los correos de una bandeja de operaciones, extrae con IA los datos clave de los documentos adjuntos (facturas, órdenes de compra y reclamos), los valida y los enruta según reglas de negocio. Reemplaza el registro manual del caso **OmniLogistics Corp** (450 correos diarios, más de 72 horas de retraso y 12% de error de digitación).

## Contenido del repositorio

| Archivo | Descripción |
| --- | --- |
| `corpomnilogistics.blueprint.json` | Blueprint exportado del escenario de Make |
| `compose.yaml` | Docker Compose del laboratorio de n8n (PostgreSQL, n8n y API de laboratorio) |
| `.env.ejemplo` | Plantilla de variables de entorno (el `.env` real no se sube al repositorio) |
| `lab-calbuco.json` | Workflow de n8n que clasifica documentos de prueba |

## Parte 1: Escenario en Make (OmniLogistics)

### Flujo

1. **Gmail – Watch Emails:** detecta correos nuevos con adjunto.
2. **Gmail – List Email Attachments:** obtiene los adjuntos del correo.
3. **Google Gemini – Upload a file:** sube el documento a Gemini.
4. **Google Gemini – Generate a response:** extrae `Tipo_Documento`, `Identificador_Emisor` y `Monto_Total` con un Prompt Maestro (temperatura 0).
5. **JSON – Parse JSON:** convierte la respuesta en variables.
6. **Router:** dirige cada documento por una de tres rutas excluyentes.

### Rutas del Router

| Ruta | Condición | Acciones |
| --- | --- | --- |
| Falta de datos | Falta el identificador o el monto, o el monto no es numérico | Correo automático al remitente pidiendo reenviar el documento |
| Aprobación | Datos completos y monto menor o igual a 10000 | Fila en Google Sheets con estado «Aprobado» y notificación al remitente |
| Escalamiento | Datos completos y monto mayor a 10000 | Fila en Google Sheets con estado «Pendiente» y alerta a Gerencia |

### Calidad de datos y resiliencia

- **Temperatura 0** en el módulo de IA, para una extracción determinista; si un dato no existe, la IA devuelve `null` y no inventa valores.
- **Break** en el módulo de IA: reintenta ante caídas o saturación del servicio.
- **Error Handler** en los módulos de Google Sheets: ante una falla de escritura, envía un correo de alerta a TI con los datos extraídos y el mensaje de error, y el flujo continúa con los demás correos.

### Cómo importar el Blueprint

1. En Make, crea un escenario nuevo.
2. Menú de tres puntos → **Import Blueprint** y selecciona `corpomnilogistics.blueprint.json`.
3. Reconecta tus propias cuentas (Gmail, Google Gemini AI y Google Sheets), porque el Blueprint no incluye credenciales.
4. Crea una hoja de Google Sheets con las columnas: Tipo_Documento, Identificador_Emisor, Monto_Total, Estado y Fecha_Proceso, y selecciónala en los módulos de Sheets.

### Pruebas realizadas

| Caso | Resultado esperado |
| --- | --- |
| Orden de compra válida menor a 10000 | Fila «Aprobado» |
| Factura válida mayor a 10000 | Fila «Pendiente» y alerta a Gerencia |
| Documento sin RUT ni monto legibles | Correo de reenvío y sin fila en Sheets |

**Video de demostración:** [pegar aquí el enlace]

## Parte 2: Laboratorio n8n en Docker

Instancia local de n8n con PostgreSQL y almacenamiento persistente, y un workflow que clasifica cuatro documentos de prueba en **Aprobado**, **Pendiente** o **Contingencia** según el monto (umbral: mayor que 500000).

### Cómo levantarlo

1. Copia `.env.ejemplo` como `.env` (`cp .env.ejemplo .env` en Linux o Mac, `Copy-Item .env.ejemplo .env` en PowerShell).
2. Ejecuta `docker compose up -d` y comprueba con `docker compose ps`.
3. Abre `http://localhost:5690` (n8n) y `http://localhost:5690/healthz` (verificación de salud).
4. En n8n, importa `lab-calbuco.json` desde **Import from File** y presiona **Execute**.

### Resultados esperados

| Documento | Monto | Estado |
| --- | --- | --- |
| OC-9102 | 320000 | Aprobado |
| Factura 4521 | 1250000 | Pendiente |
| Corrupto | sin monto | Contingencia |
| Borde | 500000 | Aprobado (el umbral es «mayor que») |

## Seguridad

- No se suben contraseñas, API Keys ni el archivo `.env`.
- Las credenciales de Make y de n8n se gestionan fuera de los archivos exportados.
