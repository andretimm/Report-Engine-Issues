# Guia da API do Motor de Relatórios

Bem-vindo à API do Report Engine. Este serviço possibilita a geração de relatórios em PDF de alta fidelidade utilizando templates HTML. Oferece recursos para criação de layouts personalizados, cabeçalhos e rodapés, com renderização baseada em injeção dinâmica de dados.

## Autenticação

Todas as requisições à API devem incluir o Token de API no cabeçalho para garantir o acesso autorizado:

```http
x-api-token: sk_seu_token_aqui
```

## Gerenciamento de Templates

Os templates constituem a base do sistema de relatórios. A API suporta a criação de diversos tipos de templates que podem ser combinados durante o processo de geração.

### Tipos de Template

| Tipo | Descrição |
| :--- | :--- |
| **LAYOUT** | Um wrapper mestre para os relatórios (ex: define `<html>`, `<head>`, estilos globais). Deve conter o placeholder `{{{ body }}}`. |
| **BODY** | O conteúdo principal do relatório (ex: tabelas, gráficos, texto). |
| **HEADER** | Cabeçalho nativo do PDF, renderizado pelo motor de impressão do navegador. |
| **FOOTER** | Rodapé nativo do PDF, renderizado pelo motor de impressão do navegador. |

### 1. Criar um Template

**Endpoint:** `POST /templates`

#### Exemplo: Layout Mestre
Este layout incorpora um cabeçalho e rodapé personalizados utilizando estrutura HTML básica.

```json
{
  "name": "Layout Corporativo",
  "type": "LAYOUT",
  "content": "<html><head><style>{{{ styles }}}</style></head><body><header>Meu Cabeçalho</header><main>{{{ body }}}</main><footer>Página {{ pageNumber }}</footer></body></html>",
  "styles": "body { font-family: Arial; }"
}
```

#### Exemplo: Corpo do Relatório (Body)
O conteúdo central do documento.

```json
{
  "name": "Fatura de Venda",
  "type": "BODY",
  "content": "<h1>Fatura #{{ invoiceNumber }}</h1><table><thead><tr><th>Item</th><th>Valor</th></tr></thead><tbody>{{#each items}}<tr><td>{{ this.name }}</td><td>{{ this.price }}</td></tr>{{/each}}</tbody></table>",
  "styles": "h1 { color: blue; }"
}
```

#### Exemplo: Cabeçalho Nativo
Conteúdo do cabeçalho renderizado pelo navegador.

```json
{
  "name": "Cabeçalho Nativo",
  "type": "HEADER",
  "content": "<div style='font-size: 10px; width: 100%; text-align: center; border-bottom: 1px solid #ddd;'>CONFIDENCIAL</div>",
  "styles": "div { color: red; }"
}
```

#### Exemplo: Rodapé Nativo (Com Contador de Páginas)
Para cabeçalhos e rodapés nativos, classes específicas são utilizadas para injetar metadados, como número da página.
*Nota: Certifique-se de que o HTML seja válido e o tamanho da fonte esteja explícito.*

```json
{
  "name": "Rodapé Nativo",
  "type": "FOOTER",
  "content": "<div style='font-size: 10px; width: 100%; text-align: right;'>Página <span class='pageNumber'></span> de <span class='totalPages'></span></div>",
  "styles": "div { color: #555; }"
}
```

### 2. Listar Templates
**Endpoint:** `GET /templates`

### 3. Atualizar Template
**Endpoint:** `PATCH /templates/:id`

### 4. Deletar Template
**Endpoint:** `DELETE /templates/:id`

---

## Renderização de Relatórios

Os relatórios podem ser gerados de forma síncrona (aguardando a finalização do PDF) ou assíncrona (utilizando uma fila de processamento).

### Cenário 1: Layout Completo (Master + Body)
Utilize um Layout Mestre para envolver o conteúdo do Body. Esta abordagem assegura consistência de estilo e estrutura entre diferentes relatórios.

**Endpoint:** `POST /render/pdf`

```json
{
  "fileName": "fatura_mensal",
  "layoutId": "uuid-do-layout-mestre",
  "bodyId": "uuid-do-template-body",
  "data": {
    "invoiceNumber": "FAT-2024-001",
    "items": [
      { "name": "Consultoria", "price": "R$ 1000" },
      { "name": "Hospedagem", "price": "R$ 50" }
    ]
  },
  "options": {
    "format": "A4",
    "margins": { "top": "10mm", "bottom": "10mm", "left": "10mm", "right": "10mm" }
  }
}
```

### Cenário 2: Apenas Body (Sem Layout)
Caso o `layoutId` seja omitido, o sistema renderiza o template Body diretamente dentro de um wrapper HTML padrão.

```json
{
  "fileName": "nota_rapida",
  "bodyId": "uuid-do-template-body",
  "data": { "message": "Olá Mundo" }
}
```

### Cenário 3: Cabeçalho & Rodapé Nativos
Para aproveitar a renderização nativa de cabeçalho e rodapé do navegador (ideal para numeração precisa de páginas sobrepondo o conteúdo), forneça os parâmetros `headerId` e `footerId`.

```json
{
  "fileName": "relatorio_com_rodape",
  "layoutId": "uuid-do-layout-mestre",
  "bodyId": "uuid-do-body",
  "headerId": "uuid-do-template-header",
  "footerId": "uuid-do-template-footer",
  "data": { ... },
  "options": {
    "repeatHeader": true,
    "repeatFooter": true,
    "margins": { "top": "20mm", "bottom": "20mm" }
  }
}
```

---

## Geração Assíncrona & Downloads

Para relatórios complexos ou alto volume de requisições, recomenda-se o uso do endpoint assíncrono.

### 1. Solicitar Geração
**Endpoint:** `POST /render/async`

**Resposta:**
```json
{
  "jobId": "39"
}
```

### 2. Verificar Status
**Endpoint:** `GET /report/status/:jobId`

**Resposta:**
```json
{
  "jobId": "39",
  "status": "completed",
  "url": "/report/download/generated-file-uuid.pdf",
  "createdAt": "2024-..."
}
```

### 3. Baixar PDF
**Endpoint:** `GET /report/download/:fileId`

Realiza o download do arquivo PDF gerado.

> [!WARNING]  
> **Política de Retenção**: Os relatórios gerados são armazenados por **24 horas**. Arquivos que excederem este período são automaticamente excluídos do servidor.
