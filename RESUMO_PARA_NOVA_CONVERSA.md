## Contexto rápido

Jogo: "AS QUATRO VONTADES — DUELO NA BOCA SECA", RPG de navegador (`https://marcelomartinsjorge.github.io/aq4v-duelo/`). Estamos corrigindo bugs de chroma-key (remoção de fundo verde) e escala/tamanho nas animações do personagem **Kronk**. Tudo documentado em detalhe em `RESUMO_PROJETO_JOGO.md`, dentro da pasta do jogo (`jogo_completo`) — **essa é a fonte de verdade, leia essa primeiro**, este arquivo aqui é só um resumo rápido pra retomar sem reler tudo.

Esta conversa parou porque o sandbox Linux (onde rodo ffmpeg/Python) ficou travado com erro de "sem espaço em disco" e não recuperou. Parece ser uma cota da conta no Cowork, não do computador do usuário (que confirmou ter bastante espaço livre). Se o problema persistir na nova conversa, o usuário deve reportar via 👎.

## Causa raiz descoberta (importante!)

O usuário, sem saber que isso causaria problema, **removeu o fundo verde de todos os vídeos do Kronk antes de me enviar** (deixando fundo preto). Isso me forçou a fazer uma conversão sintética preto→verde (`black_to_green.py`) pra poder usar meu removedor de chroma ao vivo — e essa dupla conversão criava artefatos: pontinhos pretos, partes do cabelo/corpo sumindo.

**Fix confirmado e funcionando**: quando o usuário reenviou os vídeos com o **fundo verde real** (não removido), o resultado ficou perfeito, sem precisar de nenhuma conversão sintética. Método usado (detalhado abaixo) foi validado no vídeo do rage-idle e já shipado como v22.

## Status por vídeo/pose

| Pose (classe CSS) | Vídeo novo (fundo verde real) enviado pelo usuário | Status |
|---|---|---|
| `pose-defesa` (rage idle / fúria parado) | `kronkrageidle.mp4` | ✅ **CONCLUÍDO e no ar (v22)**. Vídeo final: `kronkrageidle_v3.webm`, scale `1.43`. Usuário confirmou "ficou perfeito". |
| `pose-ataque` (ataque normal) | `kronkattack.mp4` | ✅ Processado e verificado limpo (frame a frame, sem manchas/buracos). Scale calculado: **1.70**. ⚠️ Webm final (`kronkattack_final.webm`) foi gerado só no sandbox que travou depois — **pode ter sido perdido**, precisa reprocessar (mas o método e o scale já são conhecidos, deve ser rápido). |
| `pose-dano` (dano normal) | `kronkbeenhit.mp4` (ATENÇÃO: existem 2 arquivos de upload com esse nome, um com sufixo hash — usar o mp4 mais recente/maior) | ✅ Processado e verificado limpo. Scale calculado: **1.84**. ⚠️ Mesma situação do ataque — webm pode ter sido perdido no crash, precisa reprocessar. |
| `pose-rageataque` (ataque em fúria) | `kronkrageattack.mp4` | ⏳ Não iniciado. Fonte já conferida: fundo verde bom, sem pillarbox, personagem inteiro (mangueira/maça acima da cabeça no golpe). |
| `pose-ragedano` (dano em fúria) | `kronkinragebeenhit.mp4` | ⏳ Não iniciado. Fonte já conferida: fundo verde bom, sem pillarbox. |
| pose usada ao agarrar/empurrar parede em fúria (conferir nome exato da classe no index.html, provavelmente `pose-pressao` só que em fúria, ou uma pose nova) | `kronkinragepressure.mp4` | ⏳ Não iniciado. Fonte já conferida: fundo verde bom, sem pillarbox. |
| `pose-entrarage` (entrando em fúria) | `kronkentersrage.mp4` | ⏳ Não iniciado. **Mais complicado**: tem pillarbox preto (barras) no topo/base do vídeo, E o usuário avisou que **a cor do chroma muda durante o vídeo** (a cena provavelmente tem corte/mudança de iluminação no meio) — vai precisar de detecção adaptativa por trecho, não um único threshold fixo pro vídeo inteiro. |
| `pose-ragetonormal` (saindo da fúria) | (nenhum vídeo próprio) | ⏳ Não iniciado. Usuário disse: depois que o `kronkentersrage` estiver pronto, é só **inverter a ordem dos frames** dele pra gerar o "saindo da fúria" (produção em reverse). |
| `pose-base` (idle normal) | (usuário mencionou ter erros mas não reenviou vídeo ainda) | ⏳ Não verificado com fonte nova — perguntar ao usuário se tem o vídeo com fundo verde original do idle normal, ou se o `kronk_idle_norm.webm` atual (que passou na auditoria desta sessão) já está OK. |

## Método que funcionou (usar em todos os vídeos novos)

1. **Detectar o fundo real do vídeo específico** — cada um tem um tom de verde ligeiramente diferente (variei de RGB≈(74,143,86) a (94,155,98) nos vídeos já conferidos, todos com "score" nativo de fundo entre ~50 e ~74, sendo `score = min(G-R, G-B)`). Amostrar cantos/bordas do frame (longe do personagem) pra saber o score real do fundo antes de escolher o limiar.
2. **Detectar pillarbox/letterbox** (barras pretas) se houver — atenção: a borda entre a barra preta e o conteúdo real costuma ter um degradê suave, não um corte seco. Recortar com margem de segurança (a técnica que funcionou: sample várias colunas/linhas até achar onde o valor realmente estabiliza no verde limpo, não só onde para de ser preto).
3. **Pré-chave (`prekey`) com limiar ajustado** especificamente ao verde daquele vídeo: uso `LOW=15`, `HIGH`= algo uns 15-20 pontos abaixo do score mínimo observado no fundo real daquele vídeo (ex.: se o fundo mede score~57, uso HIGH=40). Repintar o fundo como verde puro e saturado (0,255,0) com folga.
4. **Verificar rodando o algoritmo AO VIVO exato do jogo** (o mesmo `score=min(G-R,G-B)`, LOW=15/HIGH=55, cicatrização de buraco raio 6, isolamento raio 40/limiar 15, despill) em cima do resultado do passo 3 — script `chroma_sim.py` (documentado no `RESUMO_PROJETO_JOGO.md`, seção "CORREÇÃO v20"). Gerar composite sobre fundo branco/xadrez e **olhar visualmente** antes de aceitar — não confiar só em métricas automáticas de "blob cinza" (elas dão muito falso-positivo em cabelo escuro).
5. **Medir a escala correta**: `char_h_frac` = altura da silhueta do personagem como fração da altura do frame (percentil 0.3–99.7 do canal alpha), pegar o frame com maior valor ao longo da animação. `A = 1.0323` (proporção real da caixa `#kronk` dentro do `.cena` 16:9 — ver "CORREÇÃO v20" pra entender essa constante). `Va = largura/altura do vídeo`. Se `Va > A`: `rhf = A/Va` (limitado pela largura); senão `rhf = 1.0`. `onscreen_h = rhf * char_h_frac`. `scale = target / onscreen_h`, onde `target = 0.9405` (o onscreen_h do idle de referência, que usa scale 1.0). Aplicar esse `scale` em `transform:scale()` na classe CSS da pose correspondente, com `transform-origin:center bottom`.
6. **Encodar**: `ffmpeg -framerate 24 -i frames/f_%03d.png -c:v libvpx-vp9 -pix_fmt yuv420p -crf 30 -b:v 0 -row-mt 1 saida.webm`
7. Copiar o `.webm` final pra dentro de `jogo_completo`, atualizar `data-video` da canvas certa no `index.html`, atualizar o `transform:scale()` da pose, bumpar `BUILD_VERSION`.

## Deploy

Sempre que terminar mudanças em `jogo_completo`:
```powershell
robocopy "C:\Users\kuresto\Downloads\jogo\novaversão\Jogo1novaversao\jogo_completo" "C:\Users\kuresto\Downloads\jogo\novaversão\Jogo1novaversao\repo_temp" /E /XD .git
cd "C:\Users\kuresto\Downloads\jogo\novaversão\Jogo1novaversao\repo_temp"
git add -A
git commit -m "mensagem"
git push
```
`BUILD_VERSION` atual no `index.html`: **v22**.

## Próximos passos sugeridos (em ordem)

1. Reprocessar `kronkattack.mp4` (scale já sabido: 1.70) e `kronkbeenhit.mp4` (scale já sabido: 1.84) — deve ser rápido, método já validado.
2. Reprocessar `kronkinragebeenhit.mp4`, `kronkrageattack.mp4`, `kronkinragepressure.mp4` (mesmo método, calcular scale do zero pra cada).
3. Resolver `kronkentersrage.mp4` (chroma muda de cor no meio do vídeo — investigar em que frame muda, talvez precisar de dois thresholds diferentes por trecho, ou pedir pro usuário regravar essa cena com iluminação/verde consistente se a variação for grande demais pra corrigir automaticamente).
4. Gerar `kronkragetonormal` invertendo os frames do `kronkentersrage` já corrigido.
5. Perguntar ao usuário sobre o `pose-base` (idle normal) — se ele tem vídeo com fundo verde original pra essa pose também, ou se os erros que ele mencionou lá já foram resolvidos.
6. Bumpar `BUILD_VERSION`, atualizar `RESUMO_PROJETO_JOGO.md` com uma seção nova, e dar o deploy final.
