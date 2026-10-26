# Auditoria Cidadã de BUs

Site que espelha o Boletim de Urna (BU) digital de cada seção eleitoral,
exatamente como o TSE já publica em `dadosabertos.tse.jus.br`, para o
cidadão comparar com a cópia física afixada no seu local de votação.

Feito por **Coringando**. Código aberto (MIT) — veja [LICENSE](LICENSE).

## O que este site NÃO é

Isto **não é uma "apuração paralela"**. Os dados exibidos vêm da mesma fonte
oficial (TSE) — o site não faz nenhuma contagem independente de votos. O
valor dele está em facilitar a conferência individual: "o que está na tela
bate com o papel que eu vi colado na minha seção?".

## Por que não tem ranking nem contador público de "divergência"

Essa decisão de design é intencional e importante manter caso alguém for
contribuir com o projeto:

- Como o BU digital exibido vem direto do TSE, um clique de "algo errado"
  **nunca prova** divergência nenhuma por si só — ele só mede "quantas
  pessoas clicaram", o que é trivial de manipular (script, bot, campanha
  coordenada).
- Um contador público, e pior, um **ranking** de seções "mais denunciadas",
  é material pronto pra viralizar como "prova" de fraude em massa antes de
  qualquer verificação — o mesmo padrão que alimentou desinformação
  eleitoral e os ataques de 8 de janeiro de 2023 no Brasil.
- Por isso: **"Conferi, está de acordo" é o único contador público**
  (confirmação nunca pode virar arma de desinformação). Relatos de
  divergência vão para uma **fila de moderação privada** (`/moderacao`,
  com senha) e só entram em qualquer estatística depois de confirmados por
  um humano — e mesmo assim, sem formar ranking.

Pull requests que reintroduzam esse tipo de contador público não serão
aceitos sem uma solução robusta contra automação/brigading.

---

## Quero só usar (sem instalar nada de programação)

Precisa só do **Docker Desktop** instalado ([link oficial](https://www.docker.com/products/docker-desktop/)).

1. Baixe o código deste repositório: botão verde **Code → Download ZIP**
   no GitHub (ou `git clone` se preferir), e extraia numa pasta.
2. Dentro da pasta, copie `.env.example` para `.env` e preencha
   `MODERACAO_SENHA` e `SESSION_SECRET` com valores fortes (o próprio
   arquivo explica como gerar).
3. Abra um terminal nessa pasta e rode:
   ```
   docker compose up -d --build
   ```
4. Acesse **http://localhost:8080**.

Isso roda **100% no seu computador** — nenhum dado sai da sua máquina,
nenhuma assinatura de VPS ou nuvem é necessária. Para desligar:
`docker compose down` (os dados ficam salvos; rode `up -d` de novo depois).

Para popular com dados reais do TSE, veja "Coleta de dados" abaixo.

## Segurança se for deixar o site acessível pela internet

Por padrão o site só responde em `localhost` (só no seu próprio PC). Se
você for expor numa rede ou na internet (ex.: usando um serviço de nuvem):

- **Nunca** deixe `MODERACAO_SENHA` ou `SESSION_SECRET` com valores fracos
  ou repetidos de outro lugar — quem souber a senha de moderação pode
  confirmar/rejeitar relatos de divergência.
- Coloque um proxy com HTTPS na frente (Caddy, Nginx, Cloudflare Tunnel...)
  — o site em si não faz TLS.
- Veja a seção "Privacidade (LGPD)" mais abaixo.

---

## Quero contribuir com o código

### Stack

- Next.js 14 (App Router) + TailwindCSS
- PostgreSQL + Prisma ORM
- Scripts de coleta/totalização via `tsx` (rodados por cron fora do app)

### Rodando em modo desenvolvimento

```bash
cp .env.example .env.local   # preencha MODERACAO_SENHA, SESSION_SECRET, e DATABASE_URL
                              # (aponte pra um Postgres local ou use o docker-compose.yml
                              # só para o serviço "db": docker compose up -d db)
npm install
npx prisma migrate deploy
npm run seed:exemplo         # dados fictícios, só para testar a UI
npm run dev
```

### Coleta de dados

`scripts/baixar-bus.ts` é uma implementação real: pergunta pro catálogo
CKAN de dados abertos do TSE (API pública, sem chave) os recursos do pacote
`resultados-<ano>-boletim-de-urna`, baixa o recurso "Boletim de Urna" do
turno escolhido de cada UF (um `.zip` contendo um `.csv` com uma linha por
candidato/seção) e filtra só as linhas do cargo Presidente, agrupando por
seção.

**1º e 2º turno são registros diferentes** (mesma zona/seção/ano, turno
diferente) — o banco trata isso como chave composta (`uf+zona+seção+ano+
turno`), nunca sobrescrevendo um pelo outro. Na consulta por seção
(`/bu/UF/ZONA/SECAO`), se você não especificar o turno na URL
(`?turno=1` ou `?turno=2`), o site mostra automaticamente o mais recente
que existir pra aquela seção.

**Antes de rodar a coleta completa**, valide com uma UF de exemplo:

```bash
npm run inspecionar:bu -- 2022 SP      # 1º turno
npm run inspecionar:bu -- 2026 MG 2    # 2º turno
```

Isso baixa só o recurso de uma UF, mostra a lista de arquivos dentro do
zip, o cabeçalho real do CSV e tenta extrair os votos de uma seção —
confira se os números batem. Se um ano futuro mudar o layout, o mesmo
comando mostra o cabeçalho real e o erro aponta exatamente qual coluna
ajustar no mapa `COLUNAS` em `src/lib/parser-bu-csv.ts`.

**Estados grandes passam de vários GB descompactados** (o CSV do TSE tem
linha pra todos os cargos — presidente, governador, senador, deputados —
não só Presidente) — por isso a leitura é em streaming
(`src/lib/zip-csv-stream.ts`): o zip nunca é descompactado inteiro na
memória, só a linha atual é processada por vez, e a coleta grava em lotes
de 500 seções por transação.

Depois de confirmar, rode a coleta completa:

```bash
npm run coletar -- 2022            # todas as UFs, 1º turno
npm run totalizar -- 2022          # totaliza o 1º turno

npm run coletar -- 2026 MG 2       # só uma UF, 2º turno (teste antes de soltar pra todas)
npm run coletar -- 2026 2          # todas as UFs, só o 2º turno
npm run totalizar -- 2026 2        # totaliza o 2º turno
```

Agende via cron: `coletar` a cada 5 min no dia da eleição (ou 1x/dia fora
do período eleitoral), `totalizar` logo depois de cada coleta — do turno
que estiver em andamento.

### Ideias de melhoria em aberto (contribuições bem-vindas)

- Mapear número de candidato → nome/coligação na tabela `ApuracaoOficial`
  (hoje `legenda` é só o número — ver TODO em `scripts/totalizar.ts`).
- Consulta "sob demanda" (baixar/cachear o estado na hora em que alguém
  pergunta, em vez de depender de uma coleta manual prévia) — ver issues.
- Upload de evidência (foto do BU físico) no relato de divergência, visível
  só pra quem modera.

## Variáveis de ambiente

Ver `.env.example`. **Nunca** suba um `.env` de verdade pro git — o
`.gitignore` já ignora `.env` e `.env.local`, mas confira antes de um
`git add .`.

## Deploy em um serviço de nuvem (alternativa ao Docker local)

O `Dockerfile` funciona em qualquer serviço que builde a partir dele
(Railway, Render, Easypanel, Fly.io...), com um Postgres externo:

1. Crie um serviço a partir deste repositório, build via `Dockerfile`.
2. Configure as variáveis: `DATABASE_URL` (Postgres com SSL), `SESSION_SECRET`,
   `MODERACAO_SENHA`.
3. Aponte o domínio pra porta `8080`.

O container roda `prisma migrate deploy` automaticamente antes de subir o
servidor (com novas tentativas se o banco demorar a responder).

## Privacidade (LGPD)

- Nenhum IP é armazenado — só um hash irreversível (IP+UA+segredo), usado
  apenas para limitar ações por dispositivo.
- Nenhuma foto de BU físico é exibida publicamente; evidências anexadas a
  um relato (quando implementado o upload) ficam visíveis só para quem
  modera.
- Relatos de divergência não carregam nome nem identificação.

## Licença

MIT — veja [LICENSE](LICENSE). Use, copie, modifique e distribua à vontade.
