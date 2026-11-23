# Especificação - Document Management System

## 1. Objetivo

Entregar uma aplicação web simples para usuários enviarem, listarem e baixarem documentos armazenados localmente, por meio de uma API Express integrada a uma interface React.

## 2. Escopo

### Dentro do escopo

- Upload de um documento por meio de multipart/form-data.
- Listagem dos metadados dos documentos enviados.
- Download de um documento pelo seu identificador.
- Associação simples do documento a um proprietário identificado pela configuração `DMS_OWNER`.
- Interface web para selecionar, enviar, listar e baixar arquivos.
- Comunicação frontend/backend por meio do prefixo `/api`.
- Persistência local dos arquivos no filesystem da aplicação.
- Persistência dos metadados em memória durante a execução do processo backend.
- Respostas HTTP claras para entradas inválidas e recursos inexistentes.

### Fora do escopo

- Armazenamento externo, em nuvem ou em provedores de terceiros.
- Autenticação, autorização e controle completo de usuários.
- Banco de dados persistente.
- Versionamento de documentos.
- Exclusão ou substituição de documentos.
- Busca avançada, categorização, compartilhamento ou auditoria.
- Encriptação gerenciada pelo sistema.

## 3. Requisitos funcionais

| ID | Requisito |
| --- | --- |
| RF-01 | O usuário pode selecionar e enviar um único documento por meio da interface. |
| RF-02 | O backend deve rejeitar uma requisição de upload sem campo `file`. |
| RF-03 | O sistema deve salvar o conteúdo recebido no diretório local configurado usando Multer com `diskStorage`. |
| RF-04 | O sistema deve gerar um identificador único para cada documento enviado. |
| RF-05 | O sistema deve registrar metadados do documento após um upload bem-sucedido. |
| RF-06 | O usuário pode listar todos os documentos enviados na sessão atual. |
| RF-07 | A listagem deve apresentar identificador, nome original, tamanho, data de upload e proprietário. |
| RF-08 | O usuário pode baixar um documento pelo identificador recebido na listagem ou no upload. |
| RF-09 | O sistema deve preservar o nome original como nome sugerido no download. |
| RF-10 | O sistema deve retornar erro quando o identificador não existir ou o arquivo não estiver disponível localmente. |
| RF-11 | A interface deve informar erros de upload e carregamento sem interromper a aplicação. |
| RF-12 | Após um upload bem-sucedido, a interface deve atualizar automaticamente a lista de documentos. |

## 4. Requisitos não funcionais

| ID | Requisito |
| --- | --- |
| RNF-01 | Arquivos enviados devem ser gravados no filesystem local por Multer com `diskStorage`. |
| RNF-02 | O diretório de armazenamento padrão deve ser `backend/storage`. |
| RNF-03 | O diretório local pode ser sobrescrito pela variável de ambiente `DMS_STORAGE_DIR`. |
| RNF-04 | O identificador do proprietário pode ser definido pela variável `DMS_OWNER`; o valor padrão é `local-user`. |
| RNF-05 | A porta HTTP do backend pode ser configurada pela variável `PORT`, com valor padrão `3000`. |
| RNF-06 | Os metadados devem permanecer em memória nesta fase e podem ser perdidos quando o processo reiniciar. |
| RNF-07 | O backend deve seguir Node.js, Express, JavaScript CommonJS e módulos por responsabilidade. |
| RNF-08 | O frontend deve seguir React 19, Vite, JavaScript ESM e componentes funcionais com Hooks. |
| RNF-09 | O backend deve seguir Clean Architecture simples: `routes -> controllers -> services -> repositories`. |
| RNF-10 | No desenvolvimento, o frontend deve acessar `/api/*` e o Vite deve encaminhar esse prefixo ao backend na porta `3000`. |
| RNF-11 | Mensagens destinadas ao usuário devem estar em português; nomes de código e símbolos devem estar em inglês. |
| RNF-12 | A solução deve evitar abstrações desnecessárias e dependências além das já declaradas no projeto. |

## 5. Modelo de dados

### Documento

| Campo | Tipo | Obrigatório | Descrição |
| --- | --- | --- | --- |
| `id` | string | Sim | Identificador único gerado pelo backend para o arquivo armazenado. |
| `originalName` | string | Sim | Nome original enviado pelo cliente. |
| `size` | number | Sim | Tamanho em bytes informado pelo Multer. |
| `uploadedAt` | string | Sim | Data/hora do upload em formato ISO 8601. |
| `owner` | string | Sim | Identificador simples do proprietário. |
| `storageName` | string | Interno | Nome físico do arquivo dentro de `backend/storage`; não deve ser exposto pela API. |

O `storageName` é derivado de um UUID com a extensão original preservada. A coleção de metadados é mantida em uma `Map` em memória pelo repositório.

### Estados do documento

- Criado somente após o arquivo ter sido recebido pelo Multer.
- Disponível na listagem após o repositório registrar os metadados.
- Disponível para download enquanto o identificador existir em memória e o arquivo existir no diretório local.
- Reinicialização do backend remove os metadados da memória, mesmo que os arquivos permaneçam em disco.

## 6. Contratos de API

O backend expõe os endpoints diretamente na raiz. Durante o desenvolvimento, o frontend utiliza o prefixo `/api`, reescrito pelo proxy do Vite.

### GET /health

Verificação básica da disponibilidade do processo.

**Resposta 200**

```json
{
  "status": "ok"
}
```

### POST /upload

Recebe um único arquivo no campo multipart `file`.

**Entrada**

```text
Content-Type: multipart/form-data
Campo: file
Tipo: arquivo
```

**Resposta 201**

```json
{
  "id": "5d2f6d9e-f4c5-4f3f-a2d7-6bd71fdac1a2.txt",
  "originalName": "documento.txt",
  "size": 29,
  "uploadedAt": "2026-11-23T12:00:00.000Z",
  "owner": "local-user"
}
```

**Erros**

| Status | Condição | Corpo |
| --- | --- | --- |
| 400 | Campo `file` ausente | `{ "error": "Nenhum arquivo foi enviado" }` |
| 500 | Falha ao registrar os metadados | `{ "error": "Não foi possível salvar o documento" }` |

### GET /documents

Lista os metadados dos documentos registrados em memória.

**Resposta 200**

```json
[
  {
    "id": "5d2f6d9e-f4c5-4f3f-a2d7-6bd71fdac1a2.txt",
    "originalName": "documento.txt",
    "size": 29,
    "uploadedAt": "2026-11-23T12:00:00.000Z",
    "owner": "local-user"
  }
]
```

A lista deve ser ordenada do upload mais recente para o mais antigo.

### GET /documents/:id/download

Envia o conteúdo binário do documento correspondente ao identificador.

**Resposta 200**

- `Content-Type`: tipo ou aplicação adequada ao arquivo.
- `Content-Disposition`: nome original do arquivo como nome de download.
- Corpo: conteúdo binário do arquivo.

**Erros**

| Status | Condição | Corpo |
| --- | --- | --- |
| 404 | Documento não está registrado | `{ "error": "Documento não encontrado" }` |
| 404 | Arquivo não existe no armazenamento local | `{ "error": "Arquivo não encontrado no armazenamento local" }` |

## 7. Decisões arquiteturais

### Backend

- `routes/documentRoutes.js`: configura Multer com `diskStorage`, registra os endpoints e delega ao controller.
- `controllers/documentController.js`: interpreta entrada HTTP, responde aos clientes e realiza validações básicas.
- `services/documentService.js`: aplica regras de upload, listagem e consulta de metadados.
- `repositories/documentRepository.js`: encapsula o mapa em memória dos metadados.
- `app.js`: cria a aplicação Express, configura JSON e registra o router.
- Os arquivos físicos permanecem em `backend/storage` e não são armazenados em serviços externos.

O fluxo de uma operação é: `routes -> controllers -> services -> repositories`. O frontend não acessa repositórios nem o filesystem diretamente.

### Frontend

- `App.jsx` mantém o estado da página e coordena carregamento e atualização.
- `components/UploadComponent.jsx` coleta o arquivo e envia o formulário.
- `components/DocumentList.jsx` apresenta a lista e seus estados.
- `components/DownloadButton.jsx` gera o link de download.
- `services/documentService.js` centraliza chamadas `fetch` ao backend.
- O proxy de Vite encaminha `/api` para `http://localhost:3000`.

### Configuração

- `PORT`: porta HTTP do backend.
- `DMS_OWNER`: proprietário atribuído aos documentos nesta fase.
- `DMS_STORAGE_DIR`: diretório local alternativo para os arquivos enviados.

## 8. Plano de execução

1. **Estruturar a persistência local**
   - Criar o repositório em memória com operações de criação, busca por identificador e listagem.
   - Criar o diretório `backend/storage` como destino padrão dos uploads.

2. **Implementar as regras de negócio**
   - Criar o serviço responsável por validar o arquivo recebido e montar os metadados.
   - Expor operações de registro, listagem e consulta por identificador.

3. **Implementar os limites HTTP**
   - Criar o controller para upload, listagem e download.
   - Adicionar respostas HTTP e tratamento de erros nos limites da API.

4. **Configurar Multer e as rotas**
   - Usar `diskStorage` com identificador UUID e extensão original.
   - Registrar `POST /upload`, `GET /documents` e `GET /documents/:id/download`.
   - Conectar o router ao app Express sem alterar o endpoint `/health`.

5. **Cobrir o backend com testes**
   - Validar exportação do app.
   - Validar listagem vazia.
   - Validar upload, metadados, listagem e conteúdo de download.
   - Validar resposta 404 para identificadores inexistentes.

6. **Implementar o cliente frontend**
   - Criar o serviço `fetch` para upload e listagem.
   - Criar a URL de download usando o identificador codificado.

7. **Construir os componentes React**
   - Criar o componente de upload com estados de seleção e envio.
   - Criar a lista de documentos e o botão de download.
   - Integrar os componentes na página raiz e atualizar a lista após o upload.

8. **Integrar frontend e backend**
   - Manter o proxy `/api` apontando para o backend local.
   - Validar os estados de carregamento, sucesso, vazio e erro.

9. **Executar validações finais**
   - Executar os testes nativos do backend.
   - Gerar o build de produção do frontend.
   - Iniciar backend e frontend para uso local.
