# Compra360 - Automatización del proceso de solicitudes de compra con n8n

## Descripción

**Compra360** es un proyecto desarrollado para la asignatura **Electiva IV** de la EAM.

La solución busca automatizar la recepción, normalización, clasificación y registro de solicitudes de compra que, en un proceso manual, requieren revisión humana para determinar si continúan por un proceso normal o necesitan una aprobación especial según su valor y categoría.

La solución utiliza varios workflows de n8n con responsabilidades separadas:

- Un workflow principal que coordina el proceso.
- Un subworkflow encargado de normalizar los datos.
- Un workflow de manejo de errores.
- Integración con Supabase mediante HTTP Request.
- Integración prevista con Google Sheets para registrar incidentes técnicos.

---

# Integrantes

## Participación y responsabilidades

| Integrante | Asistió | Rol de la clase | Responsabilidad asumida | Aporte verificable |
|---|---|---|---|---|
| Juan Pablo Bedoya Payán | Sí | Líder de proyecto | Coordinar la revisión del workflow y validar que la lógica de decisiones, las rutas de aprobación y la integración final correspondieran con el proceso de compras definido. | Revisión del flujo principal y validación de reglas de negocio. |
| Andrea Giraldo Vásquez | Sí | Documentador / analista | Actualizar la documentación del proyecto, registrar los cambios del flujo y organizar la información necesaria para la bitácora y la entrega del segundo corte. | Bitácora actualizada y documentación del funcionamiento del workflow. |
| Sebastián Agudelo Mejía | Sí | Tester / QA | Ejecutar pruebas del Webhook desde Postman, validar las respuestas HTTP y verificar el comportamiento del flujo ante solicitudes de prueba. | Evidencias de pruebas POST, respuestas JSON y validación de ejecuciones. |
| Jerónimo Duque Rincón | Sí | Soporte técnico / integrador | Configurar y ajustar el Webhook, la normalización de datos, las referencias entre nodos y la integración mediante HTTP Request con Supabase. | Workflow actualizado, conexión con Supabase y pruebas de registro en la base de datos. |

---

# Problema que resuelve

En un proceso manual de compras, una solicitud puede ser diligenciada y enviada al área correspondiente para que un analista revise:

- Identificación de la solicitud.
- Datos del solicitante.
- Categoría de la compra.
- Valor solicitado.
- Descripción de la compra.
- Centro de costo.
- Necesidad de aprobación.
- Área responsable de la aprobación.

Este proceso manual puede generar:

- Demoras en la revisión.
- Errores al copiar o transcribir información.
- Inconsistencias en los datos.
- Dependencia del criterio del analista.
- Falta de trazabilidad.
- Dificultad para consultar solicitudes anteriores.
- Dificultad para identificar y registrar fallos técnicos.

Compra360 busca reducir estos problemas mediante reglas consistentes y una estructura automatizada en n8n.

---

# Alcance actual

## Implementado

Actualmente la solución incluye:

- Webhook HTTP POST como punto de entrada.
- Recepción de solicitudes en formato JSON.
- Subworkflow independiente para normalización.
- Limpieza y transformación de datos.
- Evaluación del valor mediante un nodo `If`.
- Clasificación mediante un nodo `Switch`.
- Rutas diferenciadas para:
  - Tecnología.
  - Infraestructura.
  - Mobiliario.
  - Otras categorías.
- Ruta de proceso normal.
- Preparación de decisión, nivel y motivo.
- HTTP Request hacia Supabase.
- Registro de solicitudes en Supabase.
- Respuesta mediante `Respond to Webhook`.
- Pruebas de solicitudes mediante Postman.
- Workflow separado para manejo de errores.
- Preparación estructurada del registro de incidentes.

## Implementado parcialmente

Los siguientes componentes ya existen, pero todavía requieren configuración o validación final:

- Asociación definitiva entre el workflow principal y el subworkflow después de importar los JSON.
- Asociación definitiva entre el workflow principal y el manejador de errores.
- Registro de incidentes mediante Google Sheets.
- Configuración del documento, hoja y columnas de Google Sheets.
- Validación completa del comportamiento del manejador de errores.

## Pendiente

Todavía se encuentra pendiente:

- Operación de lectura desde un servicio externo.
- Prevención de registros duplicados.
- Reintentos automáticos.
- Recuperación documentada después de fallos.
- Prueba final de fallo técnico controlado.
- Configuración y validación completa del segundo servicio externo.

---

# Arquitectura general

La solución está organizada actualmente de la siguiente manera:

```text
Sistema emisor / Postman
        |
        v
Webhook POST
        |
        v
Ejecutar Subworkflow
        |
        v
Subworkflow_Normalizar_Solicitud
        |
        v
If: ¿valor > $2.000.000?
       / \
      /   \
    No     Sí
    |       |
Proceso   Switch por categoría
normal    /    |      |      \
        Tec. Infra. Mob.    Otras
          \     |     |       /
                 |
                 v
            HTTP Request
                 |
                 v
              Supabase
                 |
                 v
        Respond to Webhook
```

De forma independiente existe:

```text
Manejador_Errores_Compra360
        |
        v
Error Trigger
        |
        v
Preparar Registro de Incidente
        |
        v
Google Sheets
[EN CONFIGURACIÓN]
```

---

# Organización de workflows

Los archivos exportados de n8n se almacenan dentro de:

```text
/workflows
```

Actualmente existen tres workflows principales.

---

## 1. Workflow principal

Archivo:

```text
Principal_Compra360_Modificado.json
```

Responsabilidades:

- Recibir solicitudes mediante Webhook.
- Llamar al subworkflow de normalización.
- Evaluar el valor de la compra.
- Determinar si la solicitud requiere aprobación.
- Clasificar por categoría.
- Preparar la decisión correspondiente.
- Registrar la solicitud en Supabase.
- Devolver una respuesta al sistema emisor.

### Regla principal

El valor definido como umbral es:

```text
$2.000.000 COP
```

Si:

```text
valor > 2.000.000
```

la solicitud pasa a clasificación por categoría.

Si no supera el umbral:

```text
Proceso normal
```

---

## 2. Subworkflow de normalización

Archivo:

```text
Subworkflow_Normalizar_Solicitud.json
```

Este workflow es ejecutado desde el workflow principal.

Su responsabilidad es recibir los datos originales y convertirlos a un formato consistente antes de aplicar las reglas del negocio.

### Datos procesados

#### solicitud_id

Ejemplo de transformación:

```text
" c010 "
```

se transforma en:

```text
"C010"
```

Se utiliza:

- `trim()`
- conversión a mayúsculas.

---

#### solicitante

Entrada:

```text
"   Pablo     Duque   "
```

Salida:

```text
"PABLO DUQUE"
```

Se utiliza:

- `trim()`
- reducción de espacios repetidos.
- conversión a mayúsculas.

---

#### categoria

Entrada:

```text
"Tecnología"
```

Salida:

```text
"tecnología"
```

Se utiliza:

- `trim()`
- conversión a minúsculas.

---

#### valor

El valor recibido se convierte a tipo numérico.

Ejemplo:

```text
3000000
```

---

#### descripcion

Se eliminan espacios innecesarios para mantener un formato consistente.

---

#### centro_costo

Actualmente el subworkflow contiene un valor configurado para el campo:

```text
CC-105
```

Este comportamiento debe revisarse si en la versión final el centro de costo debe llegar dinámicamente desde la solicitud.

---

#### fecha_registro

Si la solicitud incluye fecha, se utiliza esa información.

Si no la incluye, n8n genera automáticamente la fecha actual.

---

# Reglas de decisión

Después de normalizar la información, el workflow principal evalúa:

```text
¿valor > $2.000.000 COP?
```

## Si la respuesta es NO

La solicitud continúa por:

```text
Proceso normal
```

Resultado configurado:

```text
decision: PROCESO NORMAL
nivel: Automático – Analista de Compras
motivo: La compra no supera el umbral de aprobación especial
```

---

## Si la respuesta es SÍ

La solicitud pasa por:

```text
Switch por categoría
```

Las categorías definidas son:

- Tecnología.
- Infraestructura.
- Mobiliario.
- Otras categorías.

---

# Rutas de aprobación

## Tecnología

Resultado:

```text
decision: REQUIERE APROBACIÓN
nivel: Nivel 2 – Jefe de Compras
motivo: Compra de tecnología de alto valor
```

---

## Infraestructura

Resultado:

```text
decision: REQUIERE APROBACIÓN
nivel: Nivel 2 – Jefe de Compras
motivo: Compra de infraestructura de alto valor
```

---

## Mobiliario

Resultado:

```text
decision: REQUIERE APROBACIÓN
nivel: Nivel 2 – Jefe de Compras
motivo: Compra de mobiliaria de alto valor
```

---

## Otras categorías

Resultado:

```text
decision: REQUIERE APROBACIÓN
nivel: Nivel 2 – Jefe de Área
motivo: Compra de alto valor sin nivel específico
```

---

# Integraciones externas

## Supabase

Supabase es actualmente la integración externa implementada y utilizada para almacenar las solicitudes procesadas.

### Operación

```text
POST
```

### Endpoint general

```text
https://<PROJECT_REF>.supabase.co/rest/v1/solicitudes_compra
```

`<PROJECT_REF>` debe ser reemplazado por el proyecto configurado por cada usuario.

---

## Tabla

```text
solicitudes_compra
```

Campos utilizados:

```text
id
solicitud_id
solicitante
categoria
valor
descripcion
centro_costo
estado
area_aprobadora
requiere_aprobacion
asunto
fecha_registro
```

El campo `id` es generado por la base de datos y no debe ser enviado desde n8n.

---

# Google Sheets

Google Sheets está contemplado como segundo servicio externo para almacenar incidentes técnicos capturados por el manejador de errores.

Actualmente existe el nodo:

```text
Guardar en Google Sheets (COMPLETAR)
```

Sin embargo, todavía falta configurar:

- Documento.
- Hoja.
- Columnas.
- Mapeo de campos.
- Credenciales.
- Prueba real de escritura.

Por lo tanto:

```text
Estado: EN CONFIGURACIÓN
```

No debe presentarse todavía como una integración terminada.

---

# Manejador de errores

Archivo:

```text
Manejador_Errores_Compra360.json
```

Este workflow utiliza:

```text
Error Trigger
```

para activarse cuando falla una ejecución asociada.

Después ejecuta:

```text
Preparar Registro de Incidente
```

y genera información como:

- Fecha y hora.
- Nombre del workflow.
- ID de ejecución.
- Último nodo ejecutado.
- Mensaje de error.
- Stack del error.
- URL de la ejecución.

La finalidad es permitir identificar qué ocurrió y facilitar la recuperación del proceso.

Actualmente el almacenamiento definitivo del incidente en Google Sheets está pendiente de configuración.

---

# Webhook

El workflow principal utiliza un Webhook como disparador.

## Método

```text
POST
```

## Path

```text
solicitud-compra
```

---

## Ejemplo de entrada

```json
{
  "solicitud_id": "C010",
  "solicitante": "Pablo Duque",
  "categoria": "Tecnología",
  "valor": 3000000,
  "descripcion": "Compra de computador",
  "centro_costo": "TI"
}
```

---

# URL de prueba

Durante desarrollo puede utilizarse una URL similar a:

```text
http://localhost:5679/webhook-test/solicitud-compra
```

La dirección exacta depende de la instalación local de n8n.

La Test URL requiere que el workflow esté esperando el evento de prueba.

---

# URL de producción

Cuando el workflow esté publicado debe utilizarse la URL de producción generada por n8n.

No debe utilizarse:

```text
/webhook-test/
```

como endpoint permanente.

---

# Respuesta del Webhook

Después del procesamiento se utiliza:

```text
Respond to Webhook
```

Ejemplo de respuesta:

```json
{
  "recibido": true,
  "mensaje": "Solicitud recibida y procesada correctamente",
  "solicitud_id": "C010"
}
```

Código HTTP esperado:

```text
200 OK
```

---

# Caso de prueba principal

## Entrada

```json
{
  "solicitud_id": "C010",
  "solicitante": "Pablo Duque",
  "categoria": "Tecnología",
  "valor": 3000000,
  "descripcion": "Compra de computador",
  "centro_costo": "TI"
}
```

---

## Recorrido esperado

```text
Webhook
↓
Subworkflow_Normalizar_Solicitud
↓
valor = 3.000.000
↓
3.000.000 > 2.000.000
↓
TRUE
↓
Switch
↓
Tecnología
↓
Aprobación Tecnología
↓
HTTP Request
↓
Supabase
↓
Respond to Webhook
```

---

## Transformación esperada

```text
Pablo Duque
```

se transforma en:

```text
PABLO DUQUE
```

La categoría:

```text
Tecnología
```

se transforma en:

```text
tecnología
```

---

## Resultado de negocio

```text
REQUIERE APROBACIÓN
```

Nivel:

```text
Nivel 2 – Jefe de Compras
```

Motivo:

```text
Compra de tecnología de alto valor
```

---

# Comprobación del registro

Después de una ejecución exitosa:

1. Entrar a Supabase.
2. Abrir `Table Editor`.
3. Seleccionar:

```text
solicitudes_compra
```

4. Buscar:

```text
solicitud_id = C010
```

5. Verificar que la fila exista y contenga los datos esperados.

Una respuesta HTTP `200 OK` no sustituye la comprobación del registro en el sistema externo.

---

# Requisitos

Para ejecutar el proyecto se necesita:

- n8n.
- Cuenta de Supabase.
- Proyecto de Supabase configurado.
- Data API disponible.
- Tabla `solicitudes_compra`.
- API key propia.
- Postman o herramienta equivalente.
- Cuenta de Google si se completa la integración con Google Sheets.
- Los archivos JSON de los workflows.

---

# Configuración de Supabase

## Crear tabla

Crear:

```text
solicitudes_compra
```

con las columnas documentadas anteriormente.

---

## Data API

La tabla debe encontrarse disponible mediante la Data API.

---

## Row Level Security

Si RLS está habilitado, deben existir permisos compatibles con las operaciones utilizadas.

Ejemplo para permitir INSERT durante pruebas:

```sql
grant insert on table public.solicitudes_compra to anon;

alter table public.solicitudes_compra enable row level security;

create policy "permitir_insert_solicitudes"
on public.solicitudes_compra
as permissive
for insert
to anon
with check (true);
```

Esta configuración debe ajustarse a las necesidades y políticas de seguridad del entorno real.

---

# Configuración de n8n

Después de importar los workflows:

1. Importar los JSON.
2. Seleccionar correctamente el subworkflow.
3. Seleccionar correctamente el manejador de errores.
4. Configurar credenciales propias.
5. Reemplazar referencias de Supabase.
6. Revisar endpoints.
7. Configurar headers.
8. Revisar conexiones.
9. Ejecutar pruebas.
10. Publicar únicamente cuando todas las integraciones hayan sido verificadas.

---

# Importación de workflows

Los JSON están ubicados en:

```text
/workflows
```

Orden recomendado:

```text
1. Subworkflow_Normalizar_Solicitud.json
2. Manejador_Errores_Compra360.json
3. Principal_Compra360_Modificado.json
```

Después de importar:

- Seleccionar manualmente el subworkflow en el nodo `Execute Workflow`.
- Seleccionar manualmente el manejador de errores.
- Configurar las credenciales.
- Revisar las referencias entre workflows.
- Revisar los servicios externos.

---

# Seguridad

## No subir secretos a GitHub

Los archivos JSON no deben contener:

- API keys reales.
- Secret keys.
- Tokens.
- Contraseñas.
- Headers de autenticación reales.
- Archivos `.env` con credenciales.
- Datos personales reales.

---

## Supabase

Antes de exportar el workflow para GitHub:

1. Abrir el nodo `HTTP Request`.
2. Retirar la API key real del header.
3. Sustituirla temporalmente por:

```text
TU_SUPABASE_API_KEY
```

4. Sustituir la URL real por:

```text
https://TU_PROJECT_REF.supabase.co/rest/v1/solicitudes_compra
```

5. Exportar nuevamente el JSON.

---

## Revisión manual del JSON

Antes de subirlo buscar expresiones como:

```text
sb_publishable_
sb_secret_
apikey
Authorization
Bearer
token
secret
password
```

No debe quedar ningún valor real.

---

# Ejecución de prueba

## En n8n

1. Abrir el workflow principal.
2. Ejecutar desde el Webhook.
3. Esperar la petición.

---

## En Postman

Método:

```text
POST
```

Body:

```text
raw → JSON
```

Ejemplo:

```json
{
  "solicitud_id": "C010",
  "solicitante": "Pablo Duque",
  "categoria": "Tecnología",
  "valor": 3000000,
  "descripcion": "Compra de computador",
  "centro_costo": "TI"
}
```

---

# Pruebas requeridas para la entrega

Las evidencias deben almacenarse en:

```text
documentacion/evidencias/
```

La versión final debe incluir al menos:

```text
Caso_Normal.png
Entrada_No_Valida.png
Fallo_Controlado.png
```

Cada prueba debe registrar:

- Datos de entrada.
- Pasos para reproducirla.
- Resultado esperado.
- Resultado observado.
- Evidencia.
- Conclusión.

---

# Prueba normal

Debe demostrar:

```text
Entrada válida
→ procesamiento correcto
→ decisión
→ registro real
→ respuesta
```

---

# Entrada no válida

Debe demostrar qué sucede cuando falta información obligatoria o la entrada no cumple las condiciones esperadas.

Esta validación debe quedar documentada según el comportamiento final implementado.

---

# Fallo técnico controlado

Debe comprobar el comportamiento del manejador de errores.

Debe evidenciar:

- qué servicio falló;
- qué nodo falló;
- qué información capturó Error Trigger;
- cómo se registró o informó el incidente;
- qué procedimiento permite recuperar la operación.

Google Sheets solo debe mostrarse como evidencia cuando su escritura haya sido configurada y comprobada.

---

# Diagramas

Los diagramas deben almacenarse en:

```text
documentacion/diagramas/
```

Archivos previstos:

```text
Proceso_Manual.png
Proceso_Automatizado.png
```

El diagrama automatizado debe representar:

```text
Webhook
→ Execute Subworkflow
→ Normalización
→ If
→ Switch
→ rutas
→ Supabase
→ respuesta
```

y mostrar de forma separada el manejador de errores.

---

# Documentación

La documentación principal se encuentra en:

```text
documentacion/
```

Estructura:

```text
documentacion/
├── Bitacora_Unificada_Proyecto.md
├── Presentacion_Corte_2.pdf
├── diagramas/
└── evidencias/
```

---

# Presentación del Corte 2

Archivo:

```text
documentacion/Presentacion_Corte_2.pdf
```

La presentación explica:

- Proyecto e integrantes.
- Problema.
- Proceso manual.
- Objetivo y alcance.
- Arquitectura automatizada.
- Caso completo.
- Normalización.
- Reglas de decisión.
- Integraciones.
- Organización de workflows.
- Manejo de errores.
- Pruebas.
- Limitaciones.
- Estado final.

---

# Repositorio

Estructura recomendada:

```text
Compra360-Automatizacion-n8n/
│
├── README.md
│
├── workflows/
│   ├── Principal_Compra360_Modificado.json
│   ├── Subworkflow_Normalizar_Solicitud.json
│   └── Manejador_Errores_Compra360.json
│
└── documentacion/
    ├── Bitacora_Unificada_Proyecto.md
    ├── Presentacion_Corte_2.pdf
    │
    ├── diagramas/
    │   ├── Proceso_Manual.png
    │   └── Proceso_Automatizado.png
    │
    └── evidencias/
        ├── Caso_Normal.png
        ├── Entrada_No_Valida.png
        └── Fallo_Controlado.png
```

---

# Estado actual del proyecto

## Implementado

- Webhook HTTP POST.
- Recepción JSON.
- Subworkflow de normalización.
- Limpieza de datos.
- Conversión de tipos.
- Umbral de $2.000.000.
- Switch por categoría.
- Rutas de aprobación.
- Proceso normal.
- HTTP Request.
- Supabase.
- Persistencia externa.
- Respond to Webhook.
- Pruebas con Postman.

---

## Implementado parcialmente

- Organización modular en workflows.
- Asociación del subworkflow.
- Manejador de errores.
- Preparación del registro de incidentes.
- Segundo servicio externo mediante Google Sheets.

---

## Pendiente

- Finalizar Google Sheets.
- Operación de lectura.
- Prevención de duplicados.
- Reintentos.
- Recuperación después de errores.
- Prueba técnica controlada.
- Validación final de todos los workflows importados juntos.

---

# Limitaciones actuales

La versión actual todavía presenta algunas limitaciones:

- El segundo servicio externo no se encuentra completamente configurado.
- El centro de costo debe revisarse para determinar si será dinámico o fijo.
- No se ha evidenciado todavía una consulta de lectura.
- No existe todavía evidencia final de prevención de duplicados.
- El manejo de errores requiere prueba completa.
- Las referencias entre workflows deben verificarse después de importar.

---

# Evolución desde el primer corte

Desde el primer corte, Compra360 evolucionó de un flujo básico de recepción y clasificación hacia una solución más modular.

Los principales cambios incluyen:

- Sustitución del formulario como entrada principal por Webhook.
- Recepción de solicitudes externas mediante HTTP POST.
- Normalización separada en un subworkflow.
- Integración real con Supabase.
- Persistencia de solicitudes.
- Respuesta HTTP al sistema emisor.
- Separación de manejo de errores.
- Preparación de una segunda integración mediante Google Sheets.
- Organización del proyecto para publicación en GitHub.
- Documentación reproducible del sistema.

---

# Documentación y evidencias

- [Bitácora unificada](documentacion/Bitacora_Unificada_Proyecto.md)
- [Presentación Corte 2](documentacion/Presentacion_Corte_2.pdf)
- [Diagramas](documentacion/diagramas/)
- [Evidencias](documentacion/evidencias/)
- [Workflows](workflows/)

---

# Uso académico

Proyecto académico desarrollado para:

**Automatización Inteligente de Procesos con n8n**  
**EAM**  
**Periodo 2026-II**

El proyecto tiene fines académicos y demostrativos.

---

# Nota final

La documentación, los diagramas, las evidencias y los JSON deben describir siempre la misma versión de Compra360.

No debe afirmarse como implementada una funcionalidad que todavía no haya sido ejecutada y comprobada mediante evidencia.