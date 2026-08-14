# TrilhaFit

App de acompanhamento de treinos para duas pessoas (Leo e Joana), com meta individual
configurável (padrão 50 treinos) até 31/12 do ano corrente. Distribuído via link direto
(grupo do WhatsApp), sem sistema de login visível ao usuário.

**Nome do produto**: TrilhaFit — tagline "trilhando sua saúde, junto com a turma".
Renomeado de "Meta Gym" (nome original) depois de encontrar academias reais usando
"Meta Gym"/"MetaGym" na Espanha, Colômbia, Chile e México, além de um app cripto-fitness
com o mesmo nome. "Trilha" foi escolhido, entre outras opções pesquisadas, por estar livre
no espaço fitness e por encaixar melhor na história da marca (ver `meta-gym-brand` /
brand book) — o símbolo (check que vira linha ascendente) já era sobre trajetória antes
mesmo do nome existir. **Nome do repositório/pasta (`meta-gym/`) e da URL pública foram
mantidos de propósito**, para não quebrar o link já compartilhado no grupo do WhatsApp —
plumbing técnico (nomes de pasta, chaves do Firebase) é interno e não precisa bater com o
nome visível da marca.

## Stack

- **Frontend**: HTML + CSS + JavaScript puro (ES module), arquivo único `meta-gym/index.html`.
- **Persistência**: Firebase Realtime Database, projeto `meta-gym-861ca`.
- **Autenticação**: Firebase Anonymous Auth — invisível ao usuário, só gate de escrita.
- **PWA**: `manifest.json` + `sw.js` (service worker, cache do app shell) + `icon.svg`.
- **Hospedagem**: GitHub Pages, repositório `leoapmonteiro/projects`, branch `main`, raiz.
- **URL pública**: https://leoapmonteiro.github.io/projects/meta-gym/
- **Console Firebase**: https://console.firebase.google.com/project/meta-gym-861ca/database

## Regras de negócio

1. **Meta configurável por pessoa**: padrão 50 treinos até 31/12 do ano corrente (ano
   calculado dinamicamente via `new Date().getFullYear()`), mas cada um pode ajustar a
   própria em Perfil (`profile.goal`).
2. **Um check-in por pessoa por dia** — o formulário de marcação só aparece se ainda não
   houver registro para a data de hoje (`hasCheckinToday`).
3. **Identidade por dispositivo, sem senha**: na primeira abertura, a pessoa escolhe "Leo"
   ou "Joana"; a escolha fica salva em `localStorage` (`metagym_identity_v1`) e só muda via
   "trocar" com confirmação explícita. Esse é o mecanismo de UI que evita marcar no cadastro
   errado — a trava real de escrita agora vem da autenticação (ver Segurança).
4. **Observação obrigatória**: todo check-in exige um relato em texto ("Como foi o
   treino?") — funciona como diário de treino (sensações, energia, dores, evolução) e
   alimenta o painel de insights automáticos.
5. **CRUD completo, mas só dos próprios dados**: cada pessoa cria/edita/remove apenas os
   próprios check-ins. Editar preserva `id` e `date` originais e sobrescreve o resto.
6. **Cardio é aditivo**: o botão "+ Cardio" soma a marca "Cardio" aos grupos musculares
   selecionados, sem apagar o tipo de treino (ex: Treino A) já escolhido.
7. **Sincronização automática**: toda marcação/edição/remoção grava direto no Firebase; os
   dois dispositivos recebem a atualização quase em tempo real via listener (`onValue`).
8. **Reações são a única escrita "cruzada"**: é o único caso em que alguém escreve dentro do
   nó do parceiro (`/metagym/{outro}/checkins/{id}/reactions/{eu}`) — permitido porque a
   regra de segurança não distingue quem está autenticado, só que alguém está.

## Modelo de dados (Firebase Realtime Database)

```
/metagym
  /leo
    /profile
      objetivo, altura, pesoInicial, plano: "A"|"B", planOpen, goal (num, padrão 50),
      photo (data URI JPEG comprimido, ou ""), criadoEm
    /checkins
      /{id}
        id, date: "YYYY-MM-DD", tipoLabel, musculos: [...], academia, peso, obs, ts,
        reactions: { leo?: "🔥"|"👏"|"💪", joana?: "🔥"|"👏"|"💪" }   // opcional
  /joana
    /profile   {...}
    /checkins
      /{id}    {...}
```

- `id` gerado por `uid()` (timestamp base36 + sufixo aleatório) — nunca contém caracteres
  inválidos para chave do Firebase (`.`, `#`, `$`, `[`, `]`, `/`).
- `checkins` é um **objeto chaveado por id** no Firebase (não array) — convertido para array
  no cliente via `normalizePerson()` a cada snapshot recebido.
- `photo`: JPEG comprimido no cliente via `<canvas>` (recorte quadrado central, 240×240,
  qualidade 0.72) antes de virar data URI e ir pro Firebase — nunca o arquivo original.
  Tipicamente 15–40 KB, bem abaixo de qualquer limite prático de nó do Realtime Database.
- `musculos` é uma lista de tags dentre: Peito, Costas, Ombro, Bíceps, Tríceps, Perna,
  Glúteo, Core, Cardio.
- `academia` é uma das 5 fixas (Condomínio, Peralta, Hage, Cristal, Crossfit) ou texto livre.

## Segurança (Firebase Rules)

```json
{
  "rules": {
    "metagym": {
      "leo":   { ".read": true, ".write": "auth != null" },
      "joana": { ".read": true, ".write": "auth != null" }
    }
  }
}
```

- **Leitura continua aberta** (sem espera de handshake de auth) — o comparativo e a
  atividade do parceiro carregam instantaneamente.
- **Escrita exige `auth != null`** — o app faz `signInAnonymously()` sozinho, sem tela de
  login. Isso é uma trava "leve": impede escrita direta por fora do app (ex: alguém dando
  `curl -X PUT` na URL do banco), mas **não** amarra cada participante a um dispositivo
  específico — qualquer sessão anônima autenticada pode escrever em `/metagym/leo` OU
  `/metagym/joana` (é assim que as reações cruzadas funcionam). Decisão consciente: a
  alternativa (trava por `ownerUid`, primeiro dispositivo a reivindicar o nó) foi descartada
  porque bloquearia a pessoa para sempre se ela limpasse os dados do navegador ou trocasse
  de celular, sem meio de recuperação self-service.
- **Importante (herdado)**: o app lê `/metagym/leo` e `/metagym/joana` com dois listeners
  *separados* — nunca o nó pai `/metagym` (regra de leitura não sobe de filho pra pai).

Validar sempre com REST antes de considerar uma mudança de regra pronta:

```bash
DB="https://meta-gym-861ca-default-rtdb.firebaseio.com"
curl -s "$DB/metagym/leo.json"                                    # deve dar 200, sem auth
curl -s -X PUT -d '"x"' "$DB/metagym/leo/_t.json"                 # deve dar 401 (permission denied)
TOKEN=$(curl -s -X POST "https://identitytoolkit.googleapis.com/v1/accounts:signUp?key=API_KEY" \
  -d '{"returnSecureToken":true}' | grep -o '"idToken": *"[^"]*"' | sed 's/.*"\(.*\)"/\1/')
curl -s -X PUT -d '"x"' "$DB/metagym/leo/_t.json?auth=$TOKEN"     # deve dar 200
curl -s -X DELETE "$DB/metagym/leo/_t.json?auth=$TOKEN"           # limpar o teste
```

## Funcionalidades

- **Hero**: anel de progresso (X/meta), sequência de dias, dias restantes no ano, ritmo
  semanal necessário para bater a meta.
- **Lembrete**: banner no topo se o treino de hoje ainda não foi marcado (sem push —
  só aparece quando o app é aberto).
- **Check-in do dia**: tipo de treino (botões rápidos do plano ativo + "Outro" + "+ Cardio"
  aditivo), grupos musculares, academia (5 fixas + "Outra"), peso opcional, observação
  obrigatória.
- **Edição/Remoção**: CRUD completo — "editar" reabre o formulário pré-preenchido (mesmo
  id/data); "remover" em qualquer item do histórico.
- **Essa semana**: recapitulação automática dos últimos 7 dias (treinos, grupos, academias),
  sempre visível, sem precisar esperar segunda-feira nem gerar manualmente.
- **Comparativo**: cards lado a lado Leo vs Joana com foto, progresso e data do último treino.
- **Atividade do parceiro**: últimos 5 treinos do outro, com observação e botões de reação
  (🔥 👏 💪) — a única escrita cruzada entre os dois nós.
- **Insights** (por pessoa): conquistas (badges), painel de leitura automática dos dados
  (desequilíbrio muscular, menções a cansaço/dor no diário, tendência de peso, ritmo vs.
  realidade recente, melhor sequência), heatmap anual estilo GitHub, frequência semanal,
  distribuição por grupo muscular/academia, evolução de peso.
- **Conquistas**: badges por volume (10/25 treinos, meta batida) e por sequência (7/14/30
  dias seguidos), calculados no cliente a partir do histórico — nada extra salvo no Firebase.
- **Planos sugeridos**: "A" — ABC clássico; "B" — foco pernas/glúteo + preservação de massa
  magra, com aviso sobre ajuste conforme orientação médica/nutricional.
- **Perfil**: foto (upload próprio, comprimido no cliente) e meta de treinos do ano,
  editáveis a qualquer momento.
- **Compartilhar**: mensagem de resumo motivacional pro grupo do WhatsApp via Web Share API
  ou clipboard — só um resumo de texto, não é mais necessário para a sincronização em si.
- **Backup**: exporta JSON com os dados de ambos.
- **PWA**: instalável via "Adicionar à Tela de Início"; funciona offline para leitura do
  último estado sincronizado (o service worker cacheia o app shell, não os dados —
  esses continuam vindo do Firebase quando há conexão).

## Decisões de design relevantes (e porquês)

- **Sem app nativo / sem loja de app**: distribuição via link direto — o site precisa
  funcionar sem qualquer login.
- **Claude Artifacts descartado como hospedagem final**: exige login na conta Claude mesmo
  com o link compartilhado no modo mais aberto — inviável para quem não tem conta.
- **localStorage é só cache**: nunca fonte da verdade — a fonte real é sempre o Firebase.
- **`<!DOCTYPE html>`/viewport explícitos**: obrigatório fora do Artifact, senão o celular
  renderiza a página como "desktop" reduzido.
- **Insights automáticos são regras, não IA**: o painel "O que os dados mostram" é
  estatística simples (dias desde o último treino de cada grupo, contagem de palavras-chave
  no diário, tendência de peso) calculada no cliente — evita precisar de uma API key da
  Claude embutida numa página pública estática, o que não seria seguro.
- **Auth leve, não trava por dispositivo**: ver seção Segurança — prioriza nunca trancar
  alguém pra fora dos próprios dados, mesmo pagando o preço de uma trava mais fraca.
- **Ícone PWA em SVG, não PNG**: sem ferramenta de geração de imagem raster disponível no
  processo de build; funciona bem no `manifest.json` (Chrome/Android), suporte no
  `apple-touch-icon` do iOS é mais inconsistente entre versões — vale checar visualmente
  como fica ao adicionar à tela de início num iPhone.

## Posicionamento e concorrência

A visão de longo prazo (declarada pelo usuário) é crescer de "app do casal" pra plataforma
de constância com IA para famílias e turmas de amigos, com competição saudável e
premiação para quem está no caminho certo. Fases, do que está no ar até o mais distante:

1. **Dupla** (no ar) — meta individual, diário obrigatório, comparativo, insights
   automáticos, badges.
2. **Família / turma** — trocar os 2 participantes fixos por N por grupo, com ranking
   coletivo.
3. **Competição com premiação** — períodos de desafio, pontuação, prêmio pra quem manteve
   o ritmo (formato do prêmio ainda em aberto).
4. **IA como treinador pessoal** — evoluir o painel de insights (hoje regra estatística
   local) pra leitura de padrão mais rica, sem expor "IA" no nome da marca (decisão de
   marketing: nome carrega emoção, não tecnologia — tech vira obsoleta, marca não deveria).

**Concorrência direta já mapeada** (pesquisa via WebSearch, não é análise formal de
mercado, mas real o suficiente pra informar decisão de produto):

| App | O que faz | Onde o TrilhaFit é diferente |
|---|---|---|
| **Podyo** ("Desafio Fitness") | Tribos, check-in diário, ranking de grupo, desafios coletivos | Diário de treino como dado central (não só check-in binário) + insights automáticos |
| **Brasa** (dentro do GRAU Club) | Sequência diária, ranking entre amigos, "escudos" de streak, badges | Nasceu pra grupos pequenos/íntimos antes de virar rede social; foco em constância compartilhada |
| **GymRats** | Ligas entre amigos/colegas, pontuação por treino, recompensas | Sem login, abre por link — pensado pra quem não é early adopter de app de fitness |

Vale reabrir esses três (Podyo, Brasa/GRAU Club, GymRats) antes de desenhar a Fase 2
(família/turma) — eles já resolveram parte do problema de ranking/gamificação em grupo.

## Convenções de trabalho neste repo

Para publicar mudanças no app:
1. Editar `meta-gym/index.html` (e `manifest.json`/`sw.js`/`icon.svg` se necessário).
2. `git add ... && git commit -m "..." && git push`.
3. Aguardar ~30–90s de propagação do GitHub Pages — validar com `curl` buscando um trecho
   novo na URL pública antes de avisar que está pronto.
4. Se a mudança tocar em regras do Firebase ou no modelo de dados, rodar os testes REST da
   seção Segurança antes de considerar pronto.

## Ideias de expansão (backlog, não implementado)

1. **Registro de carga por exercício** (séries/reps/peso por exercício específico, não só
   grupo muscular) — vira log de treino de verdade, permite ver progressão de força.
   Esforço alto, mais fricção no dia a dia — considerar como campo opcional avançado.
2. **Fotos de progresso mensal** (Firebase Storage, linha do tempo separada dos dados
   numéricos) — diferente da foto de perfil (avatar) já implementada.
3. **Exportar para planilha (Google Sheets)** — hoje só existe backup em JSON.
4. **Virada de ano**: não há hoje um plano para 01/01 — a meta de 2027 deveria "zerar" o
   progresso, mas o histórico de 2026 deveria continuar acessível (arquivo, não apagar).
   **Decidir antes de dezembro.**
5. **Assistente de insights com IA de verdade** — hoje o painel "O que os dados mostram" é
   regra estatística local (ver Decisões de design). Uma versão com chamada real à API da
   Claude exigiria uma function/proxy segurando a API key fora do cliente (ex: Firebase
   Cloud Function), o que por sua vez exige o plano pago Blaze do Firebase — mesma trava de
   "lembrete push de verdade" (ver abaixo). Vale revisitar junto.
6. **Lembrete push de verdade** (notificação mesmo com o app fechado) — hoje só existe o
   banner "on-open" (item já implementado). A versão real precisa de Firebase Cloud
   Messaging + uma function agendada, que exige ativar o plano Blaze (cartão cadastrado,
   ainda gratuito nesse volume de uso).
