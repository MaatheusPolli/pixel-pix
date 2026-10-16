# PixelPix

Gerador e decodificador de payload **PIX estÃ¡tico** â€” o padrÃ£o EMVÂ®/BR Code definido pelo Banco Central para o "PIX copia e cola" â€” escrito em TypeScript, com uma interface web em React para gerar o QR Code em tempo real.

[![CI](https://github.com/MaatheusPolli/pixel-pix/actions/workflows/ci.yml/badge.svg)](https://github.com/MaatheusPolli/pixel-pix/actions)
[![LicenÃ§a](https://img.shields.io/badge/licenÃ§a-todos%20os%20direitos%20reservados-red)](./LICENSE)
![TypeScript](https://img.shields.io/badge/TypeScript-strict-blue)
![Cobertura](https://img.shields.io/badge/cobertura-%E2%89%A596%25-brightgreen)
![Node](https://img.shields.io/badge/node-%3E%3D20-brightgreen)

> âš ï¸ Projeto de portfÃ³lio, sob licenÃ§a proprietÃ¡ria. Ver [LicenÃ§a](#licenÃ§a) antes de reutilizar qualquer parte do cÃ³digo â€” leitura para avaliaÃ§Ã£o Ã© bem-vinda, uso e cÃ³pia nÃ£o sÃ£o permitidos sem autorizaÃ§Ã£o.

## Sobre o projeto

Todo desenvolvedor brasileiro, em algum momento, precisa gerar um cÃ³digo PIX "copia e cola" â€” e descobre que a especificaÃ§Ã£o do Bacen (baseada no padrÃ£o internacional EMVÂ® QRCPS-MPM) nÃ£o Ã© trivial: os dados sÃ£o organizados em uma Ã¡rvore de campos **TLV** (Tag-Length-Value) aninhados, o valor precisa ser formatado com exatidÃ£o, textos precisam ser normalizados para um charset ASCII restrito, e o payload inteiro Ã© fechado com uma checksum **CRC16-CCITT** calculada byte a byte.

O **PixelPix** resolve isso de duas formas:

1. **`@pixelpix/core`** â€” uma biblioteca TypeScript pura que monta, valida e decodifica esse payload, sem nenhuma dependÃªncia de runtime alÃ©m do [Zod](https://zod.dev/).
2. **`apps/web`** â€” uma interface React que usa a lib para transformar um formulÃ¡rio simples (chave Pix, nome, cidade, valor opcional) em um QR Code exibido instantaneamente, tudo rodando no navegador â€” nenhum dado Ã© enviado a um servidor.

Foi construÃ­do como peÃ§a de portfÃ³lio para demonstrar TypeScript avanÃ§ado (tipos discriminados, validaÃ§Ã£o com Zod, `strict` sem `any`), organizaÃ§Ã£o de monorepo, e uma suÃ­te de testes sÃ©ria (unitÃ¡rios, golden tests com payloads reais e testes baseados em propriedades) sobre uma lÃ³gica de domÃ­nio genuinamente nÃ£o-trivial.

## O que ele faz

- **Gera** o payload PIX estÃ¡tico a partir de: chave Pix, nome do recebedor, cidade, valor (opcional), identificador da transaÃ§Ã£o/`txid` (opcional) e descriÃ§Ã£o (opcional).
- **Detecta automaticamente o tipo da chave** informada como texto livre â€” CPF e CNPJ (com validaÃ§Ã£o real dos dÃ­gitos verificadores, mÃ³dulo 11), e-mail, telefone (com ou sem `+55`) ou chave aleatÃ³ria (EVP/UUID).
- **Calcula e anexa o CRC16** exigido pela especificaÃ§Ã£o (polinÃ´mio `0x1021`, valor inicial `0xFFFF`).
- **Normaliza texto livre** â€” remove acentos e caracteres fora do charset ASCII permitido, jÃ¡ que o payload inteiro precisa ser ASCII puro (o comprimento declarado em cada campo TLV Ã© contado em caracteres, mas o CRC Ã© calculado sobre os bytes UTF-8 â€” os dois sÃ³ coincidem se o texto for ASCII).
- **Decodifica** um payload existente de volta a um objeto tipado, verificando tambÃ©m se o CRC embutido confere com o calculado.
- **Renderiza o QR Code** ao vivo, com validaÃ§Ã£o de erros em tempo real e um botÃ£o para copiar o cÃ³digo.

## Como usar

```bash
pnpm install
pnpm build        # builda @pixelpix/core â€” necessÃ¡rio antes do app web
pnpm dev          # abre em http://localhost:5173
```

1. Preencha **Chave Pix** (CPF, CNPJ, e-mail, telefone ou chave aleatÃ³ria â€” o tipo Ã© detectado automaticamente), **Nome do recebedor** e **Cidade**.
2. Opcionalmente, informe **Valor**, **Identificador (txid)** e **DescriÃ§Ã£o**.
3. O QR Code Ã© gerado instantaneamente conforme vocÃª digita, com validaÃ§Ã£o de erros em tempo real.
4. Clique em **Copiar cÃ³digo Pix** para copiar o "copia e cola", ou escaneie o QR Code direto com o app do seu banco.

Nenhum dado sai do seu navegador â€” nÃ£o hÃ¡ backend, nÃ£o hÃ¡ coleta de informaÃ§Ã£o.

### Fora do escopo (por decisÃ£o de projeto)

- PIX dinÃ¢mico (o tipo que depende de integraÃ§Ã£o com um PSP/banco).
- Leitura de QR Code por cÃ¢mera.
- Qualquer backend ou persistÃªncia â€” Ã© 100% client-side.

## Como o payload Ã© montado

```
entrada â†’ valida (Zod) â†’ normaliza (remove acentos, ajusta charset)
        â†’ monta Ã¡rvore TLV â†’ serializa â†’ anexa cabeÃ§alho do CRC ("6304")
        â†’ calcula CRC16 â†’ payload final
```

Os principais campos do padrÃ£o EMV usados pelo PIX:

| ID   | Campo                    | ConteÃºdo                                                                |
| ---- | ------------------------ | ----------------------------------------------------------------------- |
| `00` | Payload Format Indicator | `01` (fixo)                                                             |
| `26` | Merchant Account Info    | sub-campos: `00` GUI (`br.gov.bcb.pix`), `01` chave Pix, `02` descriÃ§Ã£o |
| `52` | Merchant Category Code   | `0000`                                                                  |
| `53` | Moeda                    | `986` (BRL)                                                             |
| `54` | Valor                    | opcional                                                                |
| `58` | PaÃ­s                     | `BR`                                                                    |
| `59` | Nome do recebedor        | atÃ© 25 caracteres                                                       |
| `60` | Cidade do recebedor      | atÃ© 15 caracteres                                                       |
| `62` | Dados adicionais         | sub-campo `05`: `txid`                                                  |
| `63` | CRC16                    | 4 dÃ­gitos hexadecimais                                                  |

## Estrutura do projeto

Monorepo gerenciado com **pnpm workspaces**:

```
pixel-pix/
â”œâ”€â”€ packages/
â”‚   â””â”€â”€ core/                    # @pixelpix/core â€” lib pura, publicÃ¡vel, sem UI
â”‚       â”œâ”€â”€ src/
â”‚       â”‚   â”œâ”€â”€ domain/          # tipos e constantes do padrÃ£o EMV/Pix
â”‚       â”‚   â”œâ”€â”€ validation/      # schemas Zod + detecÃ§Ã£o/validaÃ§Ã£o de chave
â”‚       â”‚   â”œâ”€â”€ encoder/         # normalizaÃ§Ã£o, CRC16, TLV, montagem do payload
â”‚       â”‚   â””â”€â”€ decoder/         # parse do TLV e reconstruÃ§Ã£o tipada
â”‚       â””â”€â”€ tests/               # unitÃ¡rios, golden tests, property-based
â””â”€â”€ apps/
    â””â”€â”€ web/                     # app React + Vite â€” formulÃ¡rio, QR Code, copiar
        â””â”€â”€ src/
            â”œâ”€â”€ components/      # PixForm, QrPreview, CopyButton
            â””â”€â”€ hooks/           # usePixPayload â€” liga o formulÃ¡rio Ã  lib
```

A lib segue uma pequena arquitetura em camadas, com o domÃ­nio isolado de I/O â€” `core` nÃ£o depende de DOM, `Date`, `Math.random` ou qualquer efeito colateral, o que o torna 100% testÃ¡vel de forma determinÃ­stica.

## Stack tÃ©cnica

| Camada                 | Tecnologia                                      | Por quÃª                                                   |
| ---------------------- | ----------------------------------------------- | --------------------------------------------------------- |
| Linguagem              | TypeScript 5 (`strict`, sem `any`)              | seguranÃ§a de tipos ponta a ponta                          |
| ValidaÃ§Ã£o              | Zod 3                                           | schemas tipados, mensagens de erro claras e agregadas     |
| Build da lib           | tsup                                            | gera ESM + CJS + `.d.ts` com configuraÃ§Ã£o mÃ­nima          |
| Testes                 | Vitest 4 + `@vitest/coverage-v8` + `fast-check` | rÃ¡pido, nativo em TS, com testes baseados em propriedades |
| UI                     | React 18 + Vite 6                               | padrÃ£o de mercado                                         |
| QR Code                | `qrcode.react`                                  | renderizaÃ§Ã£o em SVG, direto no cliente                    |
| Lint/Format            | ESLint (typescript-eslint) + Prettier           | consistÃªncia de cÃ³digo                                    |
| CI                     | GitHub Actions                                  | lint, typecheck, testes com cobertura e build a cada push |
| Gerenciador de pacotes | pnpm                                            | workspaces nativos, instalaÃ§Ã£o rÃ¡pida                     |

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
// â†’ '00020126380014br.gov.bcb.pix0116loja@exemplo.com...6304F849'

const decoded = parsePixPayload(payload);
// â†’ { key: { kind: 'email', value: 'loja@exemplo.com' }, merchantName: 'Loja Exemplo', amount: 99.9, crcValid: true, ... }
```

Entradas invÃ¡lidas lanÃ§am `PixValidationError`, que agrega **todos** os problemas encontrados (nÃ£o sÃ³ o primeiro), cada um com um cÃ³digo estÃ¡vel (`INVALID_KEY`, `INVALID_AMOUNT`, etc.) para tratamento programÃ¡tico. HÃ¡ tambÃ©m `safeValidatePixInput`, uma versÃ£o sem exceÃ§Ã£o pensada para uso em formulÃ¡rios.

## Desenvolvimento

```bash
pnpm install        # instala dependÃªncias
pnpm build          # builda @pixelpix/core â€” necessÃ¡rio antes do app web
pnpm dev            # roda o app web (http://localhost:5173)
pnpm test           # roda os testes de todos os pacotes
pnpm test:coverage  # cobertura do core (meta: â‰¥ 90%)
pnpm lint           # verifica padrÃ£o de cÃ³digo
pnpm typecheck      # verifica tipos TypeScript
pnpm format         # aplica o Prettier
```

## EstratÃ©gia de testes

- **UnitÃ¡rios**: cada mÃ³dulo do `core` testado isoladamente â€” o CRC16, por exemplo, Ã© validado contra o vetor de teste canÃ´nico (`"123456789"` â†’ `29B1`).
- **Golden tests**: payloads fixos e verificados em `packages/core/tests/fixtures/payloads.json` â€” o encoder precisa reproduzi-los byte a byte, e o decoder precisa fazer o caminho inverso corretamente.
- **Property-based** (`fast-check`): para um universo de entradas vÃ¡lidas geradas aleatoriamente, `decode(encode(x))` sempre preserva a chave, o valor e produz um CRC vÃ¡lido.
- **Invariantes de seguranÃ§a do formato**: um payload gerado Ã© sempre ASCII puro â€” o teste verifica que o comprimento em caracteres Ã© sempre igual ao nÃºmero de bytes UTF-8, condiÃ§Ã£o necessÃ¡ria para o TLV nÃ£o corromper.

A cobertura atual do `core` estÃ¡ acima de 96% (linhas e statements), com o CI configurado para falhar abaixo de 90%.

## LicenÃ§a

**Todos os direitos reservados.** Este repositÃ³rio Ã© pÃºblico apenas para fins de portfÃ³lio â€” leitura para avaliaÃ§Ã£o tÃ©cnica Ã© bem-vinda, mas **cÃ³pia, redistribuiÃ§Ã£o e qualquer forma de uso do cÃ³digo sÃ£o proibidos** sem autorizaÃ§Ã£o expressa. Ver o arquivo [LICENSE](./LICENSE) para os termos completos.


## Licença

### Uso Pessoal
Este projeto é licenciado sob a **Licença MIT** para uso pessoal, educacional e não comercial.

### Uso Comercial
Se você deseja usar este projeto comercialmente, deve obter uma licença comercial do autor.

**Uso comercial inclui:**
- Incorporar em um produto comercial
- Oferecer como serviço
- Usar em atividades com fins lucrativos
- Implantação empresarial

**Para obter uma licença comercial:**
Email: ma.polli@hotmail.com
