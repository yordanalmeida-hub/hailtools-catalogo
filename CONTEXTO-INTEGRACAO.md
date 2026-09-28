# Contexto de integração — página Sandvik Coromant no site Hailtools (Astro + Wix headless)

Cole este arquivo no projeto do site. Ele descreve a fonte de dados pronta, as regras de exibição e o que a rota `/sandvik` precisa fazer. Uma implementação de referência (HTML puro, funcionando) está em `index.html` deste repositório e publicada em https://yordanalmeida-hub.github.io/hailtools-catalogo/ — use como espelho de comportamento, não como código a copiar.

## 1. Fonte de dados (pronta, somente leitura)

- Supabase, projeto `fthgudqfiiwyqoolcjgl` (São Paulo). Base REST: `https://fthgudqfiiwyqoolcjgl.supabase.co/rest/v1/`
- Chave pública (só leitura, pode ir no frontend): `sb_publishable_yVk3RKhFHLHQCj4drZ8-sg_csn7hNz4`
- Cabeçalhos em toda chamada: `apikey: <chave>`, `Authorization: Bearer <chave>`, `Content-Type: application/json`
- Catálogo completo Sandvik Coromant (Master Data File CoroPak 26.1): 50.183 produtos, 248 atributos ISO 13399, 249.540 links de mídia, 75.308 itens de BOM.
- **Não há preço no acervo** e não deve haver. Preço continua na Wix Stores.

### Funções (chamar via `POST /rest/v1/rpc/<nome>` com JSON no corpo)

`sandvik_search` — busca e listagem
```json
{"q": "CNMG 12 04 08-PM 4425", "p_family": null, "p_tpc": null, "p_attrs": {}, "p_highlighted": false, "p_limit": 24, "p_offset": 0}
```
- `q`: código com ou sem espaços/hífens, prefixo, erro de digitação ou texto em português. `null` = listar tudo no recorte.
- `p_family`: valor de `family_l1` (ver §3). `p_attrs`: filtros por atributo, ex. `{"GRADE":"4425","CTPT":"finishing"}` (valores como string). `p_highlighted`: só o sortimento recomendado pela Sandvik (14.860 itens).
- Retorna linhas com: `material_id, ordering_code, ordering_code_packed, description_pt, family, tpc_name, family_l1, family_l2, lcs_code, stock_code, highlighted, has_price, wix_slug, pict_url, score, total` (`total` = contagem do recorte, repetida em toda linha).

`sandvik_facet` — contagem de valores de um atributo no recorte atual (para os filtros)
```json
{"p_key": "GRADE", "q": null, "p_family": "Pastilha", "p_tpc": null, "p_attrs": {}, "p_highlighted": false, "p_limit": 8}
```
Retorna `[{value, n}]`. Chaves úteis: `GRADE` (classe), `CTPT` (tipo de operação), `TMC1_PRIMARY` (material ISO principal), `HAND` (sentido), `CUTINT_SIZESHAPE` (tamanho/forma da pastilha), `IC_MM`, `DC_MM`, `APMX_1_MM`.

`sandvik_product_full` — tudo de um produto para a página de detalhe
```json
{"p_material_id": "5724557"}
```
Retorna um JSON: `product` (todos os campos, inclusive `attrs` JSONB), `media` (`{pict_3d, draw_char, draw_char2, draw_detail, expl_view, funct, stp_basic, stp_detail, dxf_2d}` → `{url, file_name}`), `bom` (`[{code, description, included, qty}]`), `attributes` (`[{key, iso_code, label_pt, label_en, unit, value}]`), `replacement` (`{material_id, ordering_code, description_pt}` ou null).

Tabelas também expostas por REST (`GET /rest/v1/sandvik_product?select=...`), úteis para `getStaticPaths`/sitemap: `sandvik_product`, `sandvik_media`, `sandvik_bom`, `sandvik_attribute`.

Latência: 0,4–1,2 s por busca; até 2,6 s com erro de digitação. Limite do PostgREST: 8 s por consulta. Em SSR, chamar as facetas em paralelo com a busca.

## 2. Regra de preço e ação (não negociável)

- `has_price = true` (384 itens, os que existem na loja Wix atual) → mostrar chip "Preço na loja" e botão **Comprar** → `https://www.hailtools.com.br/product-page/{wix_slug}` (V1). No headless, substituir pelo produto Wix correspondente (localizar pelo nome = `ordering_code` ou pelo slug).
- caso contrário → botão **Solicitar cotação** (formulário: código pré-preenchido, quantidade com `min_order_qty` como padrão, empresa, cidade/UF, e-mail, telefone, aplicação). O envio deve virar lead no Y-CRM (Base44) com `material_id`, `ordering_code` e URL de origem — enquanto a integração não existe, enviar por e-mail para hailtools@hailtools.com.br.
- Um único botão amarelo (`#FFCC1B`) por tela: na lista nenhum botão, só chips; na página do produto, o botão Comprar ou Solicitar cotação.
- `lcs_code = 30` → chip "Em descontinuação" + bloco com o substituto (`replacement`). `stock_code = 2` → "Sob encomenda na Sandvik". `highlighted` → chip "Recomendado".

## 3. Estrutura da rota `/sandvik`

- `/sandvik` — busca + 7 famílias com contagem (`sandvik_search` com `q: null, p_family: X, p_limit: 1` → `total`). Famílias (`family_l1`, valores exatos do acervo → rótulo no site): `Pastilha`→Pastilhas · `Item da ferramenta`→Ferramentas · `Item de adaptação`→Adaptação · `Item de montagem`→Montagem · `Item acessório`→Acessórios · `Kit`→Kits · `Blank`→Blanks.
- `/sandvik/busca?q=&familia=&attrs=` — lista (24 por página) com facetas laterais (5 chaves acima), chips de filtro ativos, paginação.
- `/sandvik/p/{ordering_code_packed}` — URL canônica por produto (ex.: `/sandvik/p/CNMG120408PM4425`); resolver para `material_id` via `sandvik_product?ordering_code_packed=eq.X`. Cabeçalho: família · tipo; `ordering_code` como H1; `description_pt`; chips; ação; galeria (foto 3D, desenhos); "Dados principais" (código, código compactado, Material ID, família, tipo, classe = `attrs.GRADE`, revestimento, substrato, situação, disponibilidade Sandvik, qtde mínima, embalagem, peso, EAN, NCM = `hs_code`, lançamento, origem); "Arquivos CAD" (STP básico, STP detalhado, DXF — links diretos Sandvik, abrir em nova aba); "Ficha técnica ISO 13399" (tabela parâmetro · código · valor + unidade); "Itens na embalagem" (BOM); substituto quando houver.
- SEO: SSR com `<title>` = `{ordering_code} — {description_pt} | Hailtools`, meta description com família e classe, `og:image` = foto 3D, JSON-LD `Product` (sem preço quando não há), sitemap com os 50.183 produtos (gerar em build a partir de `sandvik_product?select=ordering_code_packed,updated_at`).

## 4. Mídia (links públicos da Sandvik, sem hospedar nada)

- Domínio `https://productinformation.sandvik.coromant.com/s3/documents/...`, cache de 1 ano, sem autenticação. Fotos jpg 1500×1500 (`pict_url` / `media.pict_3d.url`); desenhos jpg; STP e DXF como download direto.
- ~4% dos itens não têm foto: mostrar caixa neutra com o código.
- AR: fora de escopo por enquanto (não existe glTF/USDZ; depende de conversão própria e aval da Sandvik). Deixar apenas uma linha "Ver em realidade aumentada: em breve" na seção CAD.

## 5. Visual (mesmos tokens do site)

`--page #F7F7F5` · `--card #FFFFFF` · `--line #E4E4E0` · `--ink #1E1E1C` · `--ink-2 #6B6B66` · `--grafite #3C3C3B` · `--amarelo #FFCC1B` (só botão primário, um por tela) · `--on-amarelo #3D2E00`. Roboto 400/500. Raio 12 px em cards, 8 px em botões. Sem sombra, sem gradiente. Códigos em `font-variant-numeric: tabular-nums`. Mobile-first: filtros acima da lista abaixo de 820 px.

## 6. Rótulos e textos

- `attributes[].label_pt` vem em português de Portugal (ex.: "Código de Encomenda", sufixo "métrica"). Na exibição: remover o sufixo " métrica"/" Métrica" e capitalizar; tabela de correções pt-BR fica em `sandvik_attribute.label_ptbr` (hoje vazia — quando preenchida, usar `coalesce(label_ptbr, label_pt, label_en)`, que `sandvik_product_full` já faz).
- Valores a traduzir na tela: `finishing`→Acabamento, `medium`→Médio, `roughing`→Desbaste, `Neutral`→Neutro, `Right`→Direita, `Left`→Esquerda, `true/false`→Sim/Não.
- Palavras proibidas no site: "barato", "caro", "premium". Rodapé de crédito: "Dados técnicos, imagens e modelos CAD: Sandvik Coromant."

## 7. O que ainda não existe (não simular)

- Estoque Sandvik em tempo real (Stock availability) e atualização automática do catálogo (Product Data API): dependem de credenciais da Sandvik. Não mostrar estoque numérico.
- Sincronização automática `has_price`/`wix_slug` com a Wix: hoje é carga manual.
- Lead de cotação no Y-CRM: endpoint ainda não definido.
