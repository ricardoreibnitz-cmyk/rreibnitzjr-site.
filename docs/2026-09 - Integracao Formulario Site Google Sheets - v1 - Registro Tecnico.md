# Integração do formulário do site com Google Sheets

Data: 15 de setembro de 2026
Status: Produção validada

## 1. Objetivo

O formulário de contato de rreibnitzjr.com.br envia leads para uma planilha Google Sheets por meio de um Google Apps Script Web App.

## 2. Arquitetura

Fluxo:

```
Visitante
→ rreibnitzjr.com.br
→ formulário HTML
→ JavaScript fetch
→ Google Apps Script Web App
→ Google Sheets
→ resposta JSON ao navegador
```

Não existe backend adicional hospedado no GitHub Pages. Toda a lógica de recebimento e gravação dos leads roda no Google Apps Script.

## 3. Formulário

Campos:

- Nome — obrigatório
- E-mail — obrigatório
- WhatsApp — obrigatório
- Empresa — obrigatório
- Empresa já tem produto e clientes — obrigatório
- Fundador ainda participa das vendas mais importantes — obrigatório
- Mensagem — opcional

O campo WhatsApp:
- usa `type="tel"`;
- não possui máscara;
- não possui `pattern`;
- é enviado preservando exatamente o valor digitado pelo visitante.

## 4. Payload enviado

Campos do JSON enviado:

- `nome`
- `email`
- `whatsapp`
- `empresa`
- `produto_clientes`
- `fundador_vendas`
- `mensagem`
- `origem`

`origem` é sempre `Formulário do site`.

Requisição:
- `Content-Type: text/plain;charset=utf-8`
- Body: `JSON.stringify(data)`

## 5. Endpoint

Endpoint atualmente utilizado:

```
https://script.google.com/macros/s/AKfycbzS60hguLnUboUvOsLzNAOFGLScl5jeBUBsg2PWzIHWgHJzzH-VzxbIsoR662t1HGUp/exec
```

É uma implantação pública de Google Apps Script Web App. A URL já está publicada no HTML do site, servido publicamente em produção.

## 6. Google Sheets

Arquivo: `CRM Site Ricardo Reibnitz Jr`
Aba: `Leads`
Timezone: `America/Sao_Paulo`
Locale: `pt_BR`

Colunas:

| Coluna | Conteúdo |
|---|---|
| A | Data e hora |
| B | Nome |
| C | E-mail |
| D | WhatsApp |
| E | Empresa |
| F | Empresa já tem produto e clientes |
| G | Fundador ainda participa das vendas mais importantes |
| H | Mensagem |
| I | Origem |
| J | Status |

Status inicial: `Novo`

O ID interno da planilha não é documentado aqui.

## 7. Apps Script

Descrição funcional (sem reprodução do código-fonte completo):

- recebe POST;
- interpreta o corpo como JSON;
- valida os campos obrigatórios;
- abre a planilha explicitamente;
- utiliza a aba `Leads`;
- usa `LockService` para evitar colisões em gravações simultâneas;
- grava data e hora da submissão;
- formata data e hora como `dd/MM/yyyy HH:mm:ss`;
- força o campo WhatsApp como texto, preservando o valor recebido (sem normalização, sem remoção de caracteres);
- define `Origem` com o valor recebido;
- define `Status` como `Novo`;
- retorna JSON com `status: "ok"` em caso de sucesso, ou um erro em caso de falha.

Campos obrigatórios validados no servidor:

- `nome`
- `email`
- `whatsapp`
- `empresa`
- `produto_clientes`
- `fundador_vendas`

## 8. Comportamento no navegador

- Durante o envio: `Enviando...`
- Em sucesso: `Recebido. Retorno em breve.`
- Em erro: `Não foi possível enviar agora. Tente pelo WhatsApp.`

Em sucesso:
- o formulário é limpo (`reset()`);
- a resposta precisa conter `status: "ok"` para ser considerada sucesso.

## 9. Validações realizadas

### Endpoint

Testes POST diretos via `curl` retornaram `{"status":"ok"}`.

### WhatsApp

Foi validada a preservação do formato digitado, incluindo `+`, espaços, parênteses e hífens, usando como exemplo:

```
+55 48 98806-8733
```

### Data e hora

Foi validada a exibição da data e horário completos na planilha.

### Navegador local

O fluxo completo foi testado usando um servidor HTTP local (`python3 -m http.server`, bind em `127.0.0.1`) e navegador real.

Resultado:
- POST executado;
- resposta JSON processada;
- formulário limpo;
- mensagem de sucesso apresentada;
- lead confirmado na planilha.

### Produção

Foi baixado e comparado o HTML servido por `https://rreibnitzjr.com.br` com o `index.html` do `main`, e os dois arquivos estavam idênticos byte a byte.

Também foi realizada uma submissão real em produção.

Resultado:
- mensagem "Recebido. Retorno em breve.";
- lead confirmado na planilha;
- `Origem = Formulário do site`;
- `Status = Novo`.

As 5 linhas de teste geradas durante a validação foram removidas manualmente da planilha após a validação, restando apenas o cabeçalho.

### Responsividade

Formulário validado visualmente em desktop e mobile.

## 10. Histórico Git relevante

- `632a4f7` — Remove unauthorized ACATE testimonial quote
- `d441c1e` — Adiciona formulário de captação de leads na seção de contato
- `1ad1adc` — fix: ativar formulário de leads com WhatsApp
- `2f9f855` — Merge pull request #1 from ricardoreibnitz-cmyk/feat/formulario-leads-google-sheets-v2

PR: #1

Durante a implementação houve uma divergência de branches: uma implementação local do formulário foi feita em paralelo a uma implementação já publicada diretamente em `origin/main`. Para evitar reintroduzir conteúdo já removido do repositório (a citação da ACATE) e preservar o histórico remoto como fonte da verdade, a correção definitiva (adição do campo WhatsApp e do endpoint real) foi reconstruída sobre `origin/main`, em vez de mesclar as duas implementações divergentes.

## 11. Operação e manutenção

Se o formulário parar de funcionar, verificar nesta ordem:

1. o site publicado contém o endpoint correto;
2. a implantação (deployment) do Apps Script continua ativa;
3. o acesso do Web App continua permitindo submissão pública;
4. a aba `Leads` continua existindo;
5. os nomes das colunas e dos campos continuam coerentes;
6. a resposta do endpoint via `curl`;
7. o console e a aba Network do navegador;
8. a entrada efetiva na planilha.

## 12. Regra para futuras alterações

Qualquer mudança no formulário deve seguir:

```
origin/main
→ nova branch
→ alteração mínima
→ git diff --check
→ teste local
→ teste de envio
→ commit
→ push da branch
→ Pull Request
→ revisão
→ merge commit
→ validação em produção
→ sincronização do main local
→ limpeza das branches
```

Não trabalhar diretamente em `main`.

## 13. Estado final

- Site em produção.
- Formulário funcional.
- Google Apps Script ativo.
- Google Sheets recebendo leads.
- WhatsApp obrigatório.
- Testes removidos da planilha.
- Cloudflare Web Analytics preservado.
- Citação da ACATE permanece removida.
