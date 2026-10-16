# PixelPix

Gerador e decodificador de payload **PIX estático** — o padrão EMV®/BR Code definido pelo Banco Central para o "PIX copia e cola" — escrito em TypeScript, com uma interface web em React para gerar o QR Code em tempo real.

[![CI](https://github.com/MaatheusPolli/pixel-pix/actions/workflows/ci.yml/badge.svg)](https://github.com/MaatheusPolli/pixel-pix/actions)
[![Licença: MIT pessoal + comercial](https://img.shields.io/badge/licen%C3%A7a-MIT%20pessoal%20%2B%20comercial-blue.svg)](LICENSE)
![TypeScript](https://img.shields.io/badge/TypeScript-strict-blue)
![Cobertura](https://img.shields.io/badge/cobertura-%E2%89%A596%25-brightgreen)
![Node](https://img.shields.io/badge/node-%3E%3D20-brightgreen)

> Projeto de portfólio com licença dupla: livre para uso pessoal e educacional (MIT), licença comercial para uso comercial. Ver [Licença](#licença).

## Sobre o projeto

Todo desenvolvedor brasileiro, em algum momento, precisa gerar um código PIX "copia e cola" — e descobre que a especificação do Bacen (baseada no padrão internacional EMV® QRCPS-MPM) não é trivial: os dados são organizados em uma árvore de campos **TLV** (Tag-Length-Value) aninhados, o valor precisa ser formatado com exatidão, textos precisam ser normalizados para um charset ASCII restrito, e o payload inteiro é fechado com uma checksum **CRC16-CCITT** calculada byte a byte.

O **PixelPix** resolve isso de duas formas:

1. **`@pixelpix/core`** — uma biblioteca TypeScript pura que monta, valida e decodifica esse payload, sem nenhuma dependência de runtime além do [Zod](https://zod.dev/).
2. **`apps/web`** — uma interface React que usa a lib para transformar um formulário simples (chave Pix, nome, cidade, valor opcional) em um QR Code exibido instantaneamente, tudo rodando no navegador — nenhum dado é enviado a um servidor.

Foi construído como peça de portfólio para demonstrar TypeScript avançado (tipos discriminados, validação com Zod, `strict` sem `any`), organização de monorepo, e uma suíte de testes séria (unitários, golden tests com payloads reais e testes baseados em propriedades) sobre uma lógica de domínio genuinamente não-trivial.

## O que ele faz

- **Gera** o payload PIX estático a partir de: chave Pix, nome do recebedor, cidade, valor (opcional), identificador da transação/`txid` (opcional) e descrição (opcional).
- **Detecta automaticamente o tipo da chave** informada como texto livre — CPF e CNPJ (com validação real dos dígitos verificadores, módulo 11), e-mail, telefone (com ou sem `+55`) ou chave aleatória (EVP/UUID).
- **Calcula e anexa o CRC16** exigido pela especificação (polinômio `0x1021`, valor inicial `0xFFFF`).
- **Normaliza texto livre** — remove acentos e caracteres fora do charset ASCII permitido, já que o payload inteiro precisa ser ASCII puro (o comprimento declarado em cada campo TLV é contado em caracteres, mas o CRC é calculado sobre os bytes UTF-8 — os dois só coincidem se o texto for ASCII).
- **Decodifica** um payload existente de volta a um objeto tipado, verificando também se o CRC embutido confere com o calculado.
- **Renderiza o QR Code** ao vivo, com validação de erros em tempo real e um botão para copiar o código.

## Como usar

```bash
pnpm install
pnpm build        # builda @pixelpix/core — necessário antes do app web
pnpm dev          # abre em http://localhost:5173
```

1. Preencha **Chave Pix** (CPF, CNPJ, e-mail, telefone ou chave aleatória — o tipo é detectado automaticamente), **Nome do recebedor** e **Cidade**.
2. Opcionalmente, informe **Valor**, **Identificador (txid)** e **Descrição**.
3. O QR Code é gerado instantaneamente conforme você digita, com validação de erros em tempo real.
4. Clique em **Copiar código Pix** para copiar o "copia e cola", ou escaneie o QR Code direto com o app do seu banco.

Nenhum dado sai do seu navegador — não há backend, não há coleta de informação.

### Fora do escopo (por decisão de projeto)

- PIX dinâmico (o tipo que depende de integração com um PSP/banco).
- Leitura de QR Code por câmera.
- Qualquer backend ou persistência — é 100% client-side.

## Como o payload é montado

```
entrada → valida (Zod) → normaliza (remove acentos, ajusta charset)
        → monta árvore TLV → serializa → anexa cabeçalho do CRC ("6304")
        → calcula CRC16 → payload final
```

Os principais campos do padrão EMV usados pelo PIX:

| ID   | Campo                    | Conteúdo                                                                |
| ---- | ------------------------ | ----------------------------------------------------------------------- |
| `00` | Payload Format Indicator | `01` (fixo)                                                             |
| `26` | Merchant Account Info    | sub-campos: `00` GUI (`br.gov.bcb.pix`), `01` chave Pix, `02` descrição |
| `52` | Merchant Category Code   | `0000`                                                                  |
| `53` | Moeda                    | `986` (BRL)                                                             |
| `54` | Valor                    | opcional                                                                |
| `58` | País                     | `BR`                                                                    |
| `59` | Nome do recebedor        | até 25 caracteres                                                       |
| `60` | Cidade do recebedor      | até 15 caracteres                                                       |
| `62` | Dados adicionais         | sub-campo `05`: `txid`                                                  |
| `63` | CRC16                    | 4 dígitos hexadecimais                                                  |

## Estrutura do projeto

Monorepo gerenciado com **pnpm workspaces**:

```
pixel-pix/
├── packages/
│   └── core/                    # @pixelpix/core — lib pura, publicável, sem UI
│       ├── src/
│       │   ├── domain/          # tipos e constantes do padrão EMV/Pix
│       │   ├── validation/      # schemas Zod + detecção/validação de chave
│       │   ├── encoder/         # normalização, CRC16, TLV, montagem do payload
│       │   └── decoder/         # parse do TLV e reconstrução tipada
│       └── tests/               # unitários, golden tests, property-based
└── apps/
    └── web/                     # app React + Vite — formulário, QR Code, copiar
        └── src/
            ├── components/      # PixForm, QrPreview, CopyButton
            └── hooks/           # usePixPayload — liga o formulário à lib
```

A lib segue uma pequena arquitetura em camadas, com o domínio isolado de I/O — `core` não depende de DOM, `Date`, `Math.random` ou qualquer efeito colateral, o que o torna 100% testável de forma determinística.

## Stack técnica

| Camada                 | Tecnologia                                      | Por quê                                                   |
| ---------------------- | ----------------------------------------------- | --------------------------------------------------------- |
| Linguagem              | TypeScript 5 (`strict`, sem `any`)              | segurança de tipos ponta a ponta                          |
| Validação              | Zod 3                                           | schemas tipados, mensagens de erro claras e agregadas     |
| Build da lib           | tsup                                            | gera ESM + CJS + `.d.ts` com configuração mínima          |
| Testes                 | Vitest 4 + `@vitest/coverage-v8` + `fast-check` | rápido, nativo em TS, com testes baseados em propriedades |
| UI                     | React 18 + Vite 6                               | padrão de mercado                                         |
| QR Code                | `qrcode.react`                                  | renderização em SVG, direto no cliente                    |
| Lint/Format            | ESLint (typescript-eslint) + Prettier           | consistência de código                                    |
| CI                     | GitHub Actions                                  | lint, typecheck, testes com cobertura e build a cada push |
| Gerenciador de pacotes | pnpm                                            | workspaces nativos, instalação rápida                     |

## Uso da biblioteca

```ts
import { buildPixPayload, parsePixPayload } from '@pixelpix/core';

const payload = buildPixPayload({
  key: 'loja@exemplo.com',
  merchantName: 'Loja Exemplo',
  merchantCity: 'SAO PAULO',
  amount: 99.9,
  txid: 'PEDIDO123',
});
// → '00020126380014br.gov.bcb.pix0116loja@exemplo.com...6304F849'

const decoded = parsePixPayload(payload);
// → { key: { kind: 'email', value: 'loja@exemplo.com' }, merchantName: 'Loja Exemplo', amount: 99.9, crcValid: true, ... }
```

Entradas inválidas lançam `PixValidationError`, que agrega **todos** os problemas encontrados (não só o primeiro), cada um com um código estável (`INVALID_KEY`, `INVALID_AMOUNT`, etc.) para tratamento programático. Há também `safeValidatePixInput`, uma versão sem exceção pensada para uso em formulários.

## Desenvolvimento

```bash
pnpm install        # instala dependências
pnpm build          # builda @pixelpix/core — necessário antes do app web
pnpm dev            # roda o app web (http://localhost:5173)
pnpm test           # roda os testes de todos os pacotes
pnpm test:coverage  # cobertura do core (meta: ≥ 90%)
pnpm lint           # verifica padrão de código
pnpm typecheck      # verifica tipos TypeScript
pnpm format         # aplica o Prettier
```

## Estratégia de testes

- **Unitários**: cada módulo do `core` testado isoladamente — o CRC16, por exemplo, é validado contra o vetor de teste canônico (`"123456789"` → `29B1`).
- **Golden tests**: payloads fixos e verificados em `packages/core/tests/fixtures/payloads.json` — o encoder precisa reproduzi-los byte a byte, e o decoder precisa fazer o caminho inverso corretamente.
- **Property-based** (`fast-check`): para um universo de entradas válidas geradas aleatoriamente, `decode(encode(x))` sempre preserva a chave, o valor e produz um CRC válido.
- **Invariantes de segurança do formato**: um payload gerado é sempre ASCII puro — o teste verifica que o comprimento em caracteres é sempre igual ao número de bytes UTF-8, condição necessária para o TLV não corromper.

A cobertura atual do `core` está acima de 96% (linhas e statements), com o CI configurado para falhar abaixo de 90%.

## Status

Implementado (core + interface web React).

Pendências conhecidas:

- testar o payload gerado no app de pelo menos 2 bancos
- publicar a demo
- CI verde e cobertura ≥ 90% no core

## Licença

Este projeto usa **licença dupla**. O texto completo está em [LICENSE](LICENSE).

- **Uso pessoal, educacional e não comercial:** livre, nos termos da licença MIT (mantenha o aviso de copyright).
- **Uso comercial:** exige licença comercial. Isso inclui usar em produto ou serviço comercial, vender acesso ou derivados, usar em atividades com fins lucrativos e implantar em empresas.

Contato para licença comercial: ma.polli@hotmail.com
