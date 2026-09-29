# Instrução para a sessão do site — página SCHUNK (`/schunk/p/{id}`), nível 1

Objetivo: dar à página de produto SCHUNK as mesmas ações da página Sandvik (ficha técnica, 3D/CAD, cotação) usando só o que a SCHUNK já disponibiliza publicamente. Sem novos dados, sem hospedar arquivos.

## 1. Links que já existem por produto (usar o código SCHUNK numérico, ex.: `1514246`)

| Ação | URL | O que abre |
|---|---|---|
| Ficha técnica | `https://schunk.app/p/{codigo}` | Página oficial do produto no schunk.com (dados técnicos, downloads, manuais) |
| 3D e CAD | `https://schunk.app/3d/{codigo}` | Portal CAD SCHUNK (CADENAS PARTcommunity): visualizador 3D e download em STEP, DXF, DWG e outros formatos; exige cadastro gratuito |

Regras: abrir sempre em nova aba (`target="_blank" rel="noopener"`). **Não incorporar em iframe** — o portal bloqueia (`X-Frame-Options: SAMEORIGIN`). Se o produto não tiver as URLs na descrição, montar pelo código SCHUNK mesmo assim (o padrão é fixo); se o código não for numérico, ocultar os botões.

## 2. O que muda na página

1. Remover da descrição as linhas "Ficha Técnica: https://…" e "Arquivo 3D: https://…" (hoje aparecem como texto solto).
2. Criar o bloco **Documentação e CAD** logo abaixo da ação principal, com dois botões secundários (borda grafite, sem preenchimento):
   - `Abrir ficha técnica` → schunk.app/p
   - `Ver 3D e baixar CAD` → schunk.app/3d, com a linha de apoio: "Portal SCHUNK · STEP, DXF, DWG e outros formatos · cadastro gratuito"
   - Linha de rodapé do bloco: "Realidade aumentada: em breve" (não implementar agora).
3. Manter **um único botão amarelo** na tela: `Solicitar cotação` (SCHUNK não tem preço no site). WhatsApp continua como está.
4. Transformar os "Dados técnicos" da descrição em tabela: interpretar linhas no formato `Rótulo: valor` que vierem depois de "Dados técnicos:" e antes de "Ficha Técnica:", e renderizar como lista chave/valor (Dimensões, Força de fixação máxima, Torque máximo, Peso etc.). O que estiver antes de "Dados técnicos:" continua como descrição/características. Se a interpretação falhar, mostrar o texto como hoje — nunca esconder informação.
5. Cabeçalho: `Código SCHUNK {codigo}` em `tabular-nums` abaixo do nome, como na página Sandvik.

## 3. Textos (pt-BR)

- Título do bloco: **Documentação e CAD**
- Apoio do botão 3D: "Abre o portal CAD da SCHUNK. Visualize em 3D e baixe STEP, DXF ou DWG. Cadastro gratuito."
- Apoio da ficha: "Página oficial SCHUNK com dados técnicos, manuais e downloads."
- Crédito no rodapé da página: "Dados técnicos e arquivos CAD: SCHUNK."

## 4. Fora de escopo agora (depende do pacote de dados SCHUNK para distribuidor)

- Download direto de STEP/DXF com um clique dentro do site.
- Ficha técnica em PDF exibida na própria página.
- Foto oficial do produto (as URLs do CDN SCHUNK não são deriváveis do código).
- Realidade aumentada (precisa de GLB/USDZ gerados a partir do STEP, com autorização da SCHUNK).
- Atributos técnicos filtráveis por faceta, como na Sandvik.

## 5. Referência visual

Mesmos tokens do site: `--card #FFFFFF`, `--line #E4E4E0`, `--grafite #3C3C3B`, `--amarelo #FFCC1B` (só na cotação), Roboto 400/500, raio 12 px em cards e 8 px em botões, sem sombra. Layout de referência dos blocos "Arquivos CAD" e "Dados principais": https://yordanalmeida-hub.github.io/hailtools-catalogo/#/p/5724557
