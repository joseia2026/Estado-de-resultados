# Informe Económico Mensual Automatizado (n8n)

Automatización que recibe datos contables por correo electrónico, calcula un
Estado de Resultados y un cuadro de Costo de Producción, genera un informe en
PDF con gráfico de distribución de gastos, y lo envía por correo — todo sin
intervención manual en el cálculo.

Proyecto desarrollado para la Diplomatura JCIA, alojado en **n8n (self-hosted
en Pikapods)**.

---

## 📋 Índice

- [Qué hace](#qué-hace)
- [Arquitectura del flujo](#arquitectura-del-flujo)
- [Descripción de cada nodo](#descripción-de-cada-nodo)
- [Lógica contable aplicada](#lógica-contable-aplicada)
- [Configuración necesaria](#configuración-necesaria)
- [Cómo probarlo](#cómo-probarlo)
- [Limitaciones actuales](#limitaciones-actuales)
- [Próximas mejoras](#próximas-mejoras)

---

## Qué hace

1. Un cliente envía un correo a la casilla del proyecto con un archivo
   adjunto (Excel/CSV/PDF) que contiene un plan de cuentas con columnas de
   **código, descripción e importe**.
2. El sistema extrae los datos del archivo.
3. Se calculan automáticamente:
   - El **Costo de Producción** (agrupando Mano de Obra, Gastos de
     Fabricación y Costo Directo de Fabricación, con exclusiones puntuales
     de códigos).
   - El **Estado de Resultados** completo (Ventas → Rechazo → Ventas Netas →
     Resultado Bruto → Resultado Neto).
4. Se genera un informe visual (HTML) con tablas y un gráfico de torta de
   distribución de gastos.
5. El informe se convierte a **PDF**.
6. El PDF se envía por correo a la casilla del proyecto para su revisión y
   reenvío al cliente correspondiente.

---

## Arquitectura del flujo

```
Gmail Trigger (recibe el correo del cliente)
        │
        ▼
Extraer del archivo (convierte el adjunto XLSX/CSV/PDF a filas JSON)
        │
        ▼
Código JS — Cálculo (agrupaciones, sumas, exclusiones, Estado de Resultados)
        │
        ▼
Código JS — HTML (arma el informe visual + gráfico de torta vía QuickChart)
        │
        ▼
Solicitud HTTP (convierte el HTML a PDF vía Renderly API)
        │
        ▼
Enviar mensaje (Gmail — envía el PDF adjunto a la casilla del proyecto)
```

---

## Descripción de cada nodo

| Nodo | Tipo | Función |
|---|---|---|
| **GmailTrigger** | Gmail Trigger | Escucha la casilla del proyecto y dispara el flujo cuando llega un correo con adjunto. |
| **Extraer del archivo** | Extract from File | Convierte el archivo adjunto (XLSX) en un array de objetos `{codigo, descripcion, importe}`. |
| **Código en JavaScript** | Code | Motor de cálculo: agrupa por rango de códigos, aplica exclusiones, calcula el Estado de Resultados completo. |
| **Código en JavaScript1** | Code | Arma el HTML del informe (tablas + gráfico de torta con QuickChart). |
| **Solicitud HTTP** | HTTP Request | Envía el HTML a la API de Renderly y recibe el PDF generado. |
| **Enviar mensaje** | Gmail | Envía el PDF por correo a la casilla del proyecto. |

---

## Lógica contable aplicada

**Costo de Producción** = A + B + C, donde:
- **A (Mano de Obra)** → suma de códigos `641009` a `641011`.
- **B (Gastos de Fabricación)** → suma de códigos `651001` a `659004`,
  excluyendo `654001`, `654002`, `654003`.
- **C (Costo Directo de Fabricación)** → detalle de códigos `641001` a
  `641008` + `654001` a `654003` (se conserva el ítem por ítem, no solo el
  total).

**Estado de Resultados**:

```
Ventas                              (código 511000)
− Rechazo (1% de Ventas)
= Ventas Netas
− Costo de Producción               (A + B + C)
= Resultado Bruto
− Gastos de Administración          (códigos 611001–618521)
− Gastos de Comercialización        (códigos 621001–624001)
− Gastos de Financiación            (códigos 631001–634002)
= Resultado Neto
```

---

## Archivo del workflow

El flujo completo, exportado desde n8n, se encuentra en
[`workflow-informe-economico.json`](./workflow-informe-economico.json).

Para importarlo en cualquier instancia de n8n:

1. Abrir n8n → menú de tres puntos (⋯) → **"Import from File"**.
2. Seleccionar `workflow-informe-economico.json`.
3. Configurar las credenciales (Gmail OAuth2 y la API Key de Renderly) según
   la sección [Configuración necesaria](#configuración-necesaria) — el
   archivo exportado **no incluye credenciales reales** por motivos de
   seguridad, hay que volver a conectarlas manualmente en cada instancia.

---

## Configuración necesaria

> ⚠️ **Ninguna credencial real está incluida en este repositorio.** Cada
> valor debe cargarse manualmente en n8n como credencial o variable de
> entorno.

### 1. Hosting (Pikapods)

- Variable de entorno `WEBHOOK_URL` configurada con el dominio público del
  pod (necesaria para que el flujo OAuth de Gmail funcione correctamente):
  ```
  WEBHOOK_URL=https://TU-DOMINIO.pikapod.net/
  ```

### 2. Credencial Gmail OAuth2

1. Crear un proyecto en [Google Cloud Console](https://console.cloud.google.com/).
2. Habilitar la Gmail API.
3. Crear un ID de cliente OAuth 2.0 (tipo "Aplicación web").
4. Agregar como URI de redirección la que muestra n8n al crear la
   credencial Gmail OAuth2.
5. Cargar `Client ID` y `Client Secret` en n8n y autenticar con la casilla
   del proyecto.

### 3. API de conversión HTML → PDF (Renderly)

1. Crear cuenta gratuita en [renderlyapi.com](https://renderlyapi.com)
   (no requiere tarjeta).
2. Copiar la API Key desde el dashboard.
3. Cargarla en el nodo **Solicitud HTTP** como header:
   ```
   Authorization: Bearer TU_API_KEY_AQUI
   ```

---

## Cómo probarlo

1. Enviar un correo a la casilla del proyecto con un archivo Excel adjunto,
   con columnas de código, descripción e importe (usando el plan de cuentas
   descripto arriba).
2. Esperar unos segundos.
3. Revisar la bandeja de la casilla del proyecto: debería llegar un nuevo
   correo con el informe en PDF adjunto (Estado de Resultados + cuadro de
   Costo de Producción + gráfico de distribución de gastos).

---

## Limitaciones actuales

- **El destinatario final del informe es fijo** (la propia casilla del
  proyecto), no el remitente original del Excel. Esto fue una decisión
  deliberada para esta versión: funciona como un paso de control de calidad
  antes del envío al cliente, que se realiza manualmente. La automatización
  completa del reenvío directo al cliente quedó pendiente por una
  inconsistencia puntual en cómo n8n expone el campo del remitente
  (`from`) a través de los nodos intermedios que transforman el archivo
  adjunto.
- La conversión a PDF depende de un servicio externo (Renderly, plan
  gratuito) — sujeto a límites de uso mensuales.
- Si el archivo adjunto es un PDF escaneado (sin texto seleccionable), la
  extracción de datos no funciona (no hay OCR incorporado).

---

## Próximas mejoras

- [ ] Automatizar el envío del informe directamente al cliente que mandó el
  Excel original (resolver la referencia dinámica al remitente).
- [ ] Agregar validación de formato del Excel antes de procesar.
- [ ] Soporte para múltiples meses / comparativo mensual.
- [ ] Mover la conversión a PDF a una solución autoalojada, para no
  depender de un servicio externo con límites de uso.
