# Meta Gym

App de acompanhamento de treinos para duas pessoas (Leo e Joana), com meta individual
de 50 treinos cada até 31/12 do ano corrente. Distribuído via link direto (grupo do
WhatsApp), sem sistema de login.

## Stack

- **Frontend**: HTML + CSS + JavaScript puro (ES module), arquivo único `meta-gym/index.html`.
- **Persistência**: Firebase Realtime Database, projeto `meta-gym-861ca`, sem autenticação de usuário.
- **Hospedagem**: GitHub Pages, repositório `leoapmonteiro/projects`, branch `main`, raiz.
- **URL pública**: https://leoapmonteiro.github.io/projects/meta-gym/
- **Console Firebase**: https://console.firebase.google.com/project/meta-gym-861ca/database

## Regras de negócio

1. **Meta**: 50 treinos por pessoa (não combinada) até 31/12 do ano corrente — calculado
   dinamicamente via `new Date().getFullYear()`, nunca hardcoded.
2. **Um check-in por pessoa por dia** — o formulário de marcação só aparece se ainda não
   houver registro para a data de hoje (`hasCheckinToday`). Editar/remover não abre essa
   trava de volta para o mesmo dia (a não ser que o registro do dia seja removido).
3. **Identidade por dispositivo, sem senha**: na primeira abertura, a pessoa escolhe "Leo"
   ou "Joana"; a escolha fica salva em `localStorage` (`metagym_identity_v1`) e só muda via
   "trocar" com confirmação explícita. Esse é o mecanismo que evita marcar no cadastro
   errado — não há trava de segurança real por trás disso (ver seção Segurança).
4. **Observação obrigatória**: todo check-in exige um relato em texto ("Como foi o
   treino?") — funciona como diário de treino (sensações, energia, dores, evolução), não é
   campo opcional. Serve de insumo para análises de padrão pedidas sob demanda.
5. **CRUD completo, mas só dos próprios dados**: cada pessoa cria/edita/remove apenas os
   próprios check-ins — a interface nunca oferece editar o check-in do parceiro. Editar
   preserva `id` e `date` originais e sobrescreve o resto do registro.
6. **Cardio é aditivo**: o botão "+ Cardio" soma a marca "Cardio" aos grupos musculares
   selecionados, sem apagar o tipo de treino (ex: Treino A) já escolhido.
7. **Sincronização automática**: toda marcação/edição/remoção grava direto no Firebase (sem
   passo manual de compartilhar código); os dois dispositivos recebem a atualização quase em
   tempo real via listener (`onValue`).
8. **Sem autenticação real**: a separação leo/joana é só uma convenção de interface por
   dispositivo. As regras do Firebase liberam leitura/escrita em `/metagym/leo` e
   `/metagym/joana` para qualquer requisição, sem verificar quem está pedindo.

## Modelo de dados (Firebase Realtime Database)

```
/metagym
  /leo
    /profile              { objetivo, altura, pesoInicial, plano: "A"|"B", planOpen, criadoEm }
    /checkins
      /{id}                { id, date: "YYYY-MM-DD", tipoLabel, musculos: [...],
                              academia, peso, obs, ts }
  /joana
    /profile               {...}
    /checkins
      /{id}                {...}
```

- `id` gerado por `uid()` (timestamp base36 + sufixo aleatório) — nunca contém caracteres
  inválidos para chave do Firebase (`.`, `#`, `$`, `[`, `]`, `/`).
- `checkins` é um **objeto chaveado por id** no Firebase (não array) — convertido para array
  no cliente via `normalizePerson()` a cada snapshot recebido.
- `musculos` é uma lista de tags dentre: Peito, Costas, Ombro, Bíceps, Tríceps, Perna,
  Glúteo, Core, Cardio.
- `academia` é uma das 5 fixas (Condomínio, Peralta, Hage, Cristal, Crossfit) ou texto livre.

## Segurança (Firebase Rules)

```json
{
  "rules": {
    "metagym": {
      "leo":   { ".read": true, ".write": true },
      "joana": { ".read": true, ".write": true }
    }
  }
}
```

**Importante**: o app lê `/metagym/leo` e `/metagym/joana` com dois listeners *separados* —
nunca o nó pai `/metagym`. Regra de leitura não sobe de um filho liberado para o nó
ancestral (já causou bug real em produção — ver histórico de commits "Fix cloud sync").
Qualquer mudança no modelo de dados deve preservar essa correspondência exata entre
caminho lido e caminho liberado nas regras. Validar sempre com REST antes de considerar
pronto:

```bash
curl -s "https://meta-gym-861ca-default-rtdb.firebaseio.com/metagym/leo.json"
# deve devolver null ou dados — nunca {"error":"Permission denied"}
```

**Trade-off aceito conscientemente**: sem login, qualquer pessoa que descubra a URL do
banco (embutida no código-fonte público da página) pode ler/escrever em `/metagym/leo` e
`/metagym/joana`. Risco considerado baixo — dados de treino pessoal, uso familiar, sem
informação financeira. Se isso mudar (dado mais sensível, mais participantes, uso além do
círculo de confiança), revisitar com Firebase Auth (ex: anônimo + regra por `auth.uid`).

## Funcionalidades

- **Hero**: anel de progresso (X/50), sequência de dias, dias restantes no ano, ritmo
  semanal necessário para bater a meta.
- **Check-in do dia**: tipo de treino (botões rápidos do plano ativo + "Outro"), grupos
  musculares (chips multi-seleção), academia (5 fixas + "Outra"), peso opcional,
  observação obrigatória.
- **Edição**: qualquer check-in (hoje ou do histórico) tem botão "editar" — reabre o
  formulário pré-preenchido; salvar sobrescreve o registro original (mesmo id/data).
- **Remoção**: botão "remover" em qualquer item do histórico, ou "Desfazer" no card de hoje.
- **Comparativo**: cards lado a lado Leo vs Joana — progresso e data do último treino.
- **Insights** (por pessoa): frequência semanal (últimas 8 semanas), distribuição por grupo
  muscular, distribuição por academia, evolução de peso (linha).
- **Planos sugeridos**: "A" — ABC clássico (peito/tríceps, costas/bíceps, perna/ombro); "B"
  — foco pernas/glúteo + preservação de massa magra, com aviso sobre ajuste conforme
  orientação médica/nutricional (relevante por uso de medicação para emagrecimento). Cada
  pessoa escolhe o próprio plano; isso também alimenta os botões rápidos de tipo de treino.
- **Compartilhar**: gera mensagem de resumo motivacional (progresso, sequência, dias
  restantes) para o grupo do WhatsApp via Web Share API ou clipboard.
- **Backup**: exporta JSON com os dados de ambos via capability `downloads` (dentro de
  Artifact) ou download nativo do navegador (fora dele).

## Decisões de design relevantes (e porquês)

- **Sem app nativo / sem loja de app**: distribuição via link direto aberto pelo grupo do
  WhatsApp — por isso o site precisa funcionar sem qualquer login.
- **Claude Artifacts descartado como hospedagem final**: exige login na conta Claude mesmo
  com o link compartilhado no modo mais aberto — inviável para quem não tem conta (Joana).
  Usado só como ambiente de prototipagem inicial.
- **localStorage é só cache**: leitura instantânea + resiliência offline, nunca fonte da
  verdade — a fonte real é sempre o Firebase; toda escrita vai direto pra lá.
- **`<!DOCTYPE html>`/viewport explícitos**: necessários porque o Artifact injeta isso
  automaticamente, mas fora dele (GitHub Pages) precisa ser escrito à mão — sem a tag de
  viewport, celular renderiza a página como "desktop" reduzido.

## Convenções de trabalho neste repo

Para publicar mudanças no app:
1. Editar `meta-gym/index.html`.
2. `git add meta-gym/index.html && git commit -m "..." && git push`.
3. Aguardar ~30–90s de propagação do GitHub Pages antes de considerar a mudança no ar —
   validar com `curl` buscando um trecho novo do arquivo na URL pública antes de avisar
   que está pronto.

## Ideias de expansão (backlog, não implementado)

Ordenadas aproximadamente por relação esforço/impacto:

1. **Instalar como PWA** — `manifest.json` + service worker simples para "Adicionar à Tela
   de Início" funcionar como app de verdade (ícone, abre sem barra do navegador). Baixo
   esforço, ganho grande de percepção de produto acabado.
2. **Heatmap anual (estilo GitHub)** — calendário do ano colorindo os dias treinados; dá
   uma visão de constância muito mais forte que os cards de hoje.
3. **Metas intermediárias / badges** — marcos ao bater 10/25/50 treinos ou sequências de
   7/14/30 dias; reforça o incentivo mencionado no pedido original.
4. **Alertas de desequilíbrio muscular** — hoje os insights só mostram a distribuição
   passiva; poderia alertar ativamente ("você não treina perna há 12 dias").
5. **Resumo semanal automático** — toda segunda, montar um texto pronto (dias treinados,
   grupos, peso) pra compartilhar no grupo, sem precisar abrir o app e copiar manualmente.
6. **Reações do parceiro dentro do app** — um "🔥"/"👏" que a Joana manda num treino do Leo
   (e vice-versa) sem precisar sair pro WhatsApp.
7. **Lembrete diário** — notificação push (Web Push + service worker) se ainda não marcou
   até certo horário. Esforço médio-alto; funciona mal em iOS fora de PWA instalada (ver
   item 1 como pré-requisito).
8. **Registro de carga por exercício** (séries/reps/peso, não só grupo muscular) — vira log
   de treino de verdade, permite ver progressão de força. Esforço alto, mais fricção no
   dia a dia — considerar como campo opcional avançado, não obrigatório.
9. **Fotos de progresso** — upload mensal opcional (Firebase Storage), linha do tempo
   visual separada dos dados numéricos.
10. **Meta configurável por pessoa** — hoje 50 é fixo no código; poderia virar ajustável
    (relevante se um dos dois precisar recalibrar por causa de tratamento/lesão).
11. **Exportar para planilha (Google Sheets)** — hoje só existe backup em JSON; CSV/Sheets
    facilitaria análise fora do app para quem preferir Excel.
12. **Virada de ano** — não há hoje um plano para 01/01: a meta de 2027 deveria "zerar" o
    contador de progresso, mas o histórico de 2026 deveria continuar acessível (arquivo, não
    apagar). **Decidir antes de dezembro.**
13. **Assistente de insights dentro do próprio app** — hoje a análise de padrões é feita
    sob demanda, numa conversa com Claude lendo os dados via REST. Poderia virar um botão
    "gerar insights" que chama a API da Claude direto do app e mostra o resumo na tela.
14. **Autenticação leve (Firebase Anonymous Auth)** — se o círculo de confiança do app
    crescer além de Leo e Joana, vale trocar a convenção de dispositivo por uma trava real
    de escrita por participante.
