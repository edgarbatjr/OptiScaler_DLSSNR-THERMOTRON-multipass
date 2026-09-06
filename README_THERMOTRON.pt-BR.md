# OptiScaler DLSS-NR — multi-pass com controle de halo (fork THERMOTRON / Blue Rattler)

Um fork do [OptiScaler_DLSSNR](https://github.com/Dagherbou/OptiScaler_DLSSNR) que roda o modelo de
Neural Rendering do DLSS 5 **mais de uma vez por frame** — até 4 passadas de verdade, cada uma na
sua própria feature do modelo — e acrescenta os controles que a gente precisou pra manter a imagem
limpa fazendo isso:

- **Passadas 1–4.** Cada passada é uma feature NGX separada, alimentada com a resposta da anterior
  (sem histórico compartilhado, logo sem smear), construída um frame antes pra troca de contagem
  nunca travar o frame.
- **Força por passada** (`PassDecay2/3/4`): as passadas seguintes rodam com Intensity e Local
  structure reduzidas. O brilho na silhueta ("halo") é contraste de borda que o modelo puxa, e N
  passadas puxam N vezes; passadas mais fracas puxam menos, e o modelo continua vendo o frame inteiro.
- **Resolução por passada** (`PassScale2/3/4`): cada passada seguinte na sua fração do tamanho de
  trabalho. Ela vê a imagem atual encolhida, e o que ela acrescenta volta como resíduo por cima da
  imagem cheia — a textura fina da passada 1 fica inteira; as seguintes rodam menores, mais baratas e
  mais suaves. Escada sempre **descendo** (100 / 85 / 70), nunca subindo.
- **Edge guard** (`EdgeGuardMode`): banda na silhueta pela profundidade (soften / sem clarear /
  luma lock) e os modos "detail only", que dividem de volta a mudança de brilho em larga escala pra
  sobrar só textura. Debug view mostra a banda.
- Tudo isso convive com os modos reversível / replace, resolução do modelo, supersampling e as
  ferramentas de exposição que o fork já tinha.

Testado em RE Engine (Onimusha: Way of the Sword), REDkit (The Blood of Dawnwalker) e Creation
Engine 2 (Starfield), RTX 5090 em 4K, driver 616.64. Vídeos: [Blue Rattler no YouTube](https://www.youtube.com/@BlueRattler).

> **Você precisa fornecer o `nvngx_dlssnr.dll`** (o modelo da NVIDIA, ~165 MB). Ele não está, e
> nunca vai estar, neste repositório nem nas releases. Tudo aqui é não documentado e sem suporte
> da NVIDIA; use por sua conta.

## Instalação

1. Compile (Visual Studio 2022, Release x64) ou pegue o `OptiScaler.dll` de uma release, renomeie
   pra `dxgi.dll` e coloque junto do executável do jogo, com o `nvngx.dll_dlssnr.dll` (o forwarder,
   vem na release) e o seu `nvngx_dlssnr.dll`.
2. Copie o `OptiScaler.ini` correspondente da pasta `presets/` (um por jogo) pra lá também.
3. No jogo, abra o menu do OptiScaler (Insert) → *DLSS Neural Rendering*. Tudo abaixo é ao vivo.

## Os botões (nome no menu → chave no ini, seção `[DlssNr]`)

| Menu | Ini | O que faz | Nosso valor |
|---|---|---|---|
| Passes | `Passes` | quantas vezes o modelo roda por frame, 1–4. Custo linear (~7 ms por passada em 4K numa 5090) | 3 |
| Pass 2/3/4 strength | `PassDecay2/3/4` | multiplicador de Intensity + Local structure daquela passada | RE Engine 0.93 / 0.50 · REDkit 1.00 / 0.89 |
| Pass 2/3/4 resolution | `PassScale2/3/4` | tamanho daquela passada como fração do tamanho de trabalho | RE Engine 0.93 / 0.80 · REDkit 1.00 / 1.00 (ou 0.95 / 0.83 pra economizar 2,5 ms) |
| Model resolution | `WorkingScale` | tamanho da passada 1 (as outras seguem) | 1.0 (0.70 é um preset barato bom) |
| Edge guard (halo) | `EdgeGuardMode` | 0 off · 1 soften · 2 sem clarear · 3 luma lock · 4 detail only · 5 detail only + banda | 0 — com força e resolução por passada a guarda deixou de ser necessária; o 4 custa iluminação |
| Edge guard strength / threshold / radius | `EdgeGuard` / `EdgeThreshold` / `EdgeRadius` | quanto / o que conta como silhueta (salto de 1/z) / largura da banda em px | 1.0 / 0.10 / 10–12 |
| Lock between passes | `EdgeBetweenPasses` | experimental; perde detalhe, deixe desligado | false |
| Reversible proxy | `ReversibleMode` | 3 = híbrido (composto), 4 = híbrido + replace (a resposta do modelo é a imagem) | 4 nos três jogos |

Ajuste do modelo que a gente usa: Intensity 1.5 (1.4 no REDkit), Local structure 1.1, Local tone 0.8,
Skin 1.2 (0.89 no REDkit), Preset 3, auto skin mask, Detail strength 1.1, Colour 1.0. Style é gosto
por cena: Natural pra realismo, Cinematic pra impacto, Default pra "sem filtro".

## Presets por faixa de placa (ponto de partida — reporte o seu ms)

O detalhe vem da passada 1; as seguintes acrescentam acabamento e brilho. Então o jeito barato de
descer a escada é: passada 1 cheia, as seguintes menores, e só depois baixar o Model resolution.
Só a linha da 5090 é medida (4K); as outras são estimativa proporcional — abra uma issue com a
placa, a resolução e o ms que o menu mostra.

| Placa | Passadas | Força passada 2 / 3 | Resolução passada 2 / 3 | Model resolution | ~ms |
|---|---|---|---|---|---|
| RTX 5090 / 5080, 4K | 3 | 0.93 / 0.50 (RE Engine) · 1.00 / 0.89 (REDkit) | 93 / 80 · 100 / 100 | 100% | 18–21 |
| RTX 4090 / 4080, 4K | 3 | 0.90 / 0.50 | 90 / 70 | 100% | ~22–26 |
| RTX 4070 / 3080, 1440p | 2 | 0.80 | 70 | 85% | ~10 |
| RTX 3070 / 4060, 1440p | 1 | — | — | 75% | ~7 |

`presets/OptiScaler.lowend-2x.ini` é o ponto de partida de 2 passadas. Duas passadas com a segunda
a 70% custam ~1,5x uma passada só, e continua sendo multi-pass de verdade: cada passada na própria
feature do modelo, alimentada com a resposta da anterior.

## O que este fork é, ao lado dos outros multi-pass

Multi-pass no DLSS-NR está sendo feito em paralelo: os PRs [#23](https://github.com/Dagherbou/OptiScaler_DLSSNR/pull/23)
e [#26](https://github.com/Dagherbou/OptiScaler_DLSSNR/pull/26) no repo original, e os forks do
wilsjo2 e do y4my4my4m. O que este branch acrescenta em cima de "rodar o modelo de novo" é a parte
que fez mais de uma passada ficar usável aos nossos olhos: força por passada, resolução por passada
com o resíduo composto, a guarda de borda pela profundidade, e presets medidos em três motores.
Ficaríamos felizes de ver qualquer parte disso entrar no repo original.

## O que aprendemos sobre o halo

A borda clara em volta de um personagem escuro num fundo claro não é erro de uma passada só: é
contraste de borda que o modelo puxa, somado pelas passadas. O que ataca a soma funciona (passadas
seguintes mais fracas e menores, edge guard); o que tenta apagar depois perde detalhe (tentamos
travar o brilho entre passadas — ainda está no menu, desligado). O modelo da NVIDIA em 1 passada
ainda puxa a parte dele; essa só uma atualização do modelo resolve.

## Créditos

Construído sobre o OptiScaler_DLSSNR do Dagherbou e o projeto OptiScaler. Resolve por resíduo
casado do PR do hhkbble. O resto do `feat/dlssnr-multipass-frame-ahead` é do THERMOTRON, com o
Claude de parceiro de laboratório.
