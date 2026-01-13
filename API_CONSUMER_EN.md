# Report Engine API Guide

Welcome to the Report Engine API. This service enables the generation of high-fidelity PDF reports using HTML templates. It provides capabilities for designing custom layouts, headers, and footers, and rendering them with dynamic data injection.

## Authentication

All API requests must include the API Token in the header to ensure authorized access:

```http
x-api-token: sk_your_token_here
```

## Template Management

Templates are the fundamental components of the reporting system. The API supports creating various template types that can be combined during the generation process.

### Template Types

| Type | Description |
| :--- | :--- |
| **LAYOUT** | A master wrapper for reports (e.g., defines `<html>`, `<head>`, global styles). It must include the `{{{ body }}}` placeholder. |
| **BODY** | The primary content of the report (e.g., tables, charts, text). |
| **HEADER** | Native PDF header rendered by the browser's printing engine. |
| **FOOTER** | Native PDF footer rendered by the browser's printing engine. |

### 1. Create a Template

**Endpoint:** `POST /templates`

#### Example: Master Layout
This layout embeds a custom header and footer using HTML structure.

```json
{
  "name": "Corporate Layout",
  "type": "LAYOUT",
  "content": "<html><head><style>{{{ styles }}}</style></head><body><header>My Header</header><main>{{{ body }}}</main><footer>Page {{ pageNumber }}</footer></body></html>",
  "styles": "body { font-family: Arial; }"
}
```

#### Example: Report Body
The core content of the document.

```json
{
  "name": "Invoice Body",
  "type": "BODY",
  "content": "<h1>Invoice #{{ invoiceNumber }}</h1><table><thead><tr><th>Item</th><th>Cost</th></tr></thead><tbody>{{#each items}}<tr><td>{{ this.name }}</td><td>{{ this.price }}</td></tr>{{/each}}</tbody></table>",
  "styles": "h1 { color: blue; }"
}
```

#### Example: Native Header
Header content rendered by the browser.

```json
{
  "name": "Native Header",
  "type": "HEADER",
  "content": "<div style='font-size: 10px; width: 100%; text-align: center; border-bottom: 1px solid #ddd;'>CONFIDENTIAL</div>",
  "styles": "div { color: red; }"
}
```

#### Example: Native Footer (with Page Counter)
For native PDF headers and footers, special class names are used to inject metadata such as page numbers.
*Note: Ensure valid HTML and explicit font sizing.*

```json
{
  "name": "Native Footer",
  "type": "FOOTER",
  "content": "<div style='font-size: 10px; width: 100%; text-align: right;'>Page <span class='pageNumber'></span> of <span class='totalPages'></span></div>",
  "styles": "div { color: #555; }"
}
```

### 2. List Templates
**Endpoint:** `GET /templates`

### 3. Update Template
**Endpoint:** `PATCH /templates/:id`

### 4. Delete Template
**Endpoint:** `DELETE /templates/:id`

---

## Rendering Reports

Reports can be generated either synchronously (blocking until the PDF is ready) or asynchronously (using a job queue).

### Scenario 1: Full Layout (Master + Body)
Utilize a Master Layout to wrap the Body content. This approach ensures consistent styling and structure across different reports.

**Endpoint:** `POST /render/pdf`

```json
{
  "fileName": "monthly_invoice",
  "layoutId": "uuid-of-master-layout",
  "bodyId": "uuid-of-invoice-body",
  "data": {
    "invoiceNumber": "INV-2024-001",
    "items": [
      { "name": "Consulting", "price": "$1000" },
      { "name": "Hosting", "price": "$50" }
    ]
  },
  "options": {
    "format": "A4",
    "margins": { "top": "10mm", "bottom": "10mm", "left": "10mm", "right": "10mm" }
  }
}
```

### Scenario 2: Simple Body (No Layout)
If the `layoutId` is omitted, the system renders the Body template directly within a default HTML wrapper.

```json
{
  "fileName": "quick_note",
  "bodyId": "uuid-of-simple-body",
  "data": { "message": "Hello World" }
}
```

### Scenario 3: Native Header & Footer
To leverage the browser's native header and footer rendering (ideal for precise page numbering), provide the `headerId` and `footerId`.

```json
{
  "fileName": "report_with_native_footer",
  "layoutId": "uuid-of-master-layout",
  "bodyId": "uuid-of-body",
  "headerId": "uuid-of-header-template",
  "footerId": "uuid-of-footer-template",
  "data": { ... },
  "options": {
    "repeatHeader": true,
    "repeatFooter": true,
    "margins": { "top": "20mm", "bottom": "20mm" }
  }
}
```

---

## Asynchronous Generation & Downloads

For complex reports or high-volume requests, it is recommended to use the asynchronous endpoint.

### 1. Request Generation
**Endpoint:** `POST /render/async`

**Response:**
```json
{
  "jobId": "39"
}
```

### 2. Check Status
**Endpoint:** `GET /report/status/:jobId`

**Response:**
```json
{
  "jobId": "39",
  "status": "completed",
  "url": "/report/download/generated-file-uuid.pdf",
  "createdAt": "2024-..."
}
```

### 3. Download PDF
**Endpoint:** `GET /report/download/:fileId`

Retrieves the generated PDF file.

> [!WARNING]  
> **Retention Policy**: Generated reports are retained for **24 hours**. Reports exceeding this period are automatically deleted from the server.
