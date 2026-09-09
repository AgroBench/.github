# AgroBench — benchmarking regional com dados anonimizados e recompensa em blockchain

## Repositórios

Não é monorepo: a submissão é a org [AgroBench](https://github.com/AgroBench). Cada pasta local com `.git` é um repo separado.

| Pasta local | Repo | URL | Papel |
|---|---|---|---|
| `backend/` | backend | https://github.com/AgroBench/backend | API Go + adapter Solana |
| `frontend/` | AgroBenchFront | https://github.com/AgroBench/frontend | App Vue (produtor/instituição) |
| `landing-page/` | agrobenchlanding | https://github.com/AgroBench/landing-page | Landing |
| `pitch-deck/` | pitch-deck | https://github.com/AgroBench/pitch-deck | Deck 10 slides |
| `programs/` | programs | https://github.com/AgroBench/programs | Programa Anchor (escrow stake + pool + split) |

- Program id (Devnet, **deployado e inicializado**): [`EytN8UaXrfTQc6Pq4AdQbQyJwUX37ddXsV7URayBBLrN`](https://explorer.solana.com/address/EytN8UaXrfTQc6Pq4AdQbQyJwUX37ddXsV7URayBBLrN?cluster=devnet)
- Mint USDC Devnet (Circle): `4zMMC9srt5Ri5X14GAgXhaHii3GnPAEERYPJgZJDncDU`
- Ordem de leitura: [programs](https://github.com/AgroBench/programs) (`lib.rs`) → [backend](https://github.com/AgroBench/backend) (`chain.go`) → [AgroBenchFront](https://github.com/AgroBench/AgroBenchFront) (`stake.js`) → [explorer](https://explorer.solana.com/address/EytN8UaXrfTQc6Pq4AdQbQyJwUX37ddXsV7URayBBLrN?cluster=devnet)
- Guia Devnet: [AgroBench/programs](https://github.com/AgroBench/programs) (`DEPLOY-STATUS.md`, `PASSO-A-PASSO-DEVNET.md`).

## O problema

Produtores rurais tomam decisões de compra (adubo, defensivos, insumos) sem nenhuma referência de mercado confiável. Não existe um "quanto o vizinho está pagando" acessível — cada produtor negocia isolado, sem saber se está pagando um preço justo pela sua região.

Ao mesmo tempo, bancos, seguradoras e cooperativas precisam desses mesmos dados agregados (custo médio, produtividade média por região/cultura) para calcular risco de crédito rural e seguro agrícola — e hoje pagam caro por pesquisas de mercado desatualizadas ou trabalham com estimativas.

O AgroBench resolve as duas pontas ao mesmo tempo, sem expor o dado individual de ninguém.

## Mercado

O Plano Safra 2025/2026 movimenta R$ 605,2 bilhões em crédito rural subsidiado no Brasil. Todo esse volume depende de avaliação de risco — hoje feita com dados de safras anteriores, autodeclaração do produtor e estimativas de instituições como CONAB, sem uma fonte de custo/produtividade regional atualizada e granular.

- **TAM**: o universo de crédito rural e seguro agrícola no Brasil, hoje limitado por falta de dado confiável de custo e produtividade regional — R$ 605,2 bilhões movimentados por ciclo agrícola no Plano Safra.
- **SAM**: bancos, seguradoras e cooperativas de crédito que operam originação e precificação de risco agrícola nas regiões produtoras do Sul/Sudeste, dispostos a pagar por dado agregado atualizado em vez de estimativa histórica.
- **SOM**: cooperativas e bancos regionais do Rio Grande do Sul, começando por Passo Fundo e microrregião — a base de produtores que o AgroBench recruta primeiro, com cobertura suficiente para gerar um agregado estatisticamente relevante por cultura.

O AgroBench monetiza um dado que hoje ninguém compila de forma estruturada: o custo real, por insumo, por produtor, atualizado a cada safra.

## Como funciona, em uma frase

O produtor contribui dados criptografados sobre sua propriedade; o sistema valida e agrega esses dados por região; o produtor vê de graça como ele se compara à média regional, e instituições pagam para consultar o mesmo agregado — receita que retorna automaticamente, via smart contract, para as wallets dos produtores que contribuíram.

## Arquitetura técnica

### 1. Coleta e envio (produtor)

- O produtor usa um app local para registrar seus dados de custo/produtividade por safra.
- Identidade no protocolo = uma wallet Solana (chave pública). Nome, CPF/CNPJ e endereço civil nunca vão on-chain nem para o payload de dados.
- Localização é generalizada para microrregião — nunca coordenada exata — o que elimina o risco de re-identificação por cruzamento com imagens de satélite ou registros públicos de imóveis.

### 2. Commit-reveal (integridade do dado)

- Antes de enviar o payload, o produtor publica **on-chain o hash** dos dados (commit).
- O payload é enviado em seguida, criptografado ponta a ponta com a chave pública do ambiente de validação (reveal).
- Isso impede que o produtor decida depois se envia ou não os dados após saber se seriam "aceitos": o commit existe antes da validação e não pode ser alterado. Isso elimina o viés de seleção no dataset.

### 3. Validação

- O payload é decifrado exclusivamente dentro de um TEE (enclave de hardware isolado). Nem o operador da infraestrutura acessa o conteúdo em texto puro — o TEE garante isso por construção, não por política interna.
- A validação cruza o dado declarado com fontes públicas (sensoriamento remoto, CONAB) para detectar inconsistências. Um dado que diverge do padrão esperado para aquela área e cultura é rejeitado antes de compor o agregado.

### 4. Agregação regional

- Após validado, o dado individual é usado apenas para calcular estatísticas agregadas (média, mediana) por região × cultura × período.
- O valor bruto individual não fica acessível fora do processo de agregação. O que existe publicamente é só o agregado.

### 5. Atestação e recompensa base

- O validador assina uma atestação on-chain confirmando que aquele hash foi validado e compõe o agregado.
- Essa atestação libera a **recompensa base em USDC-SPL** direto na wallet do produtor, por ter contribuído — independente de quem depois consulta o painel.

### 6. Painel de benchmarking

- O produtor consulta de graça e vê, por exemplo: *"a média de custo de adubo na sua região foi R$ 1.200/ha; você gastou R$ 1.500/ha"* — informação acionável para renegociar com fornecedores.
- Bancos, seguradoras e cooperativas consultam o mesmo painel via assinatura paga, para calcular risco de crédito e seguro agrícola com dados reais e atualizados em vez de estimativas.

### 6.1 Wallet simplificada (onboarding sem fricção)

O produtor não precisa entender blockchain, seed phrase ou gas fee para participar. No primeiro acesso ao app:

- Uma **wallet Solana é criada automaticamente em segundo plano**, vinculada ao login do produtor.
- O login usa e-mail, telefone e CPF. O CPF nunca é gravado on-chain nem entra no payload de dados — ele existe **apenas no banco de identidade do AgroBench**, armazenado como hash irreversível, usado só para recuperação de conta e para a verificação de CAR (seção 6.3). Esse vínculo CPF → wallet existe e é assumido explicitamente: é o único ponto onde identidade real e identidade on-chain se tocam, e por isso é o dado mais protegido do sistema — acesso restrito, hash e não texto puro, e nunca exposto a validadores, instituições compradoras ou qualquer parte externa.
- A chave privada da wallet fica custodiada de forma não-custodial-assistida: criptografada no dispositivo do produtor, com recuperação via confirmação por SMS vinculada ao hash de CPF armazenado. O produtor nunca vê nem precisa copiar uma seed phrase.
- Taxas de rede (gas) das transações on-chain (commit, atestação) são pagas pelo protocolo. O produtor nunca precisa ter SOL na carteira.
- A experiência do produtor é "criei uma conta no app". A wallet existe de fato por trás, é exportável e pode ser migrada para controle total do produtor a qualquer momento.

### 6.2 Exemplos de dados granulares e como o benefício cresce

Quanto mais detalhado o dado contribuído, mais precisa fica a agregação regional — o que aumenta o valor pago por bancos e seguradoras — e maior a recompensa do produtor. A estrutura é em três camadas:

**Nível básico** (contribuição mínima, recompensa base):
- Cultura plantada e área total (ex.: soja, 50 ha)
- Custo total da safra (um número agregado)

**Nível intermediário** (recompensa maior — dado mais raro e mais útil):
- Custo discriminado por insumo (adubo, defensivo, semente, combustível, mão de obra)
- Produtividade por hectare (sacas/ha)
- Data de plantio e colheita
- Fornecedor utilizado por categoria de insumo — o nome do fornecedor não é exposto publicamente; ele entra apenas no cálculo de "preço médio pago por tipo de insumo na região"

**Nível avançado** (recompensa mais alta — dado denso, alto valor para seguradora e banco):
- Tipo de solo e histórico de rotação de cultura na área
- Uso de irrigação (sim/não, tipo de sistema)
- Ocorrência de pragas/doenças na safra e método de manejo usado
- Perdas por evento climático (seca, granizo, geada) com percentual estimado da área afetada
- Nível de mecanização (própria, terceirizada, tipo de maquinário)

Cada camada adicional de detalhe aumenta o valor do agregado regional. O smart contract de distribuição pondera a recompensa do produtor pela granularidade e pela raridade do que ele contribuiu, não apenas pelo fato de ter contribuído.

### 6.3 Barreira contra empresas se passando por produtor

O acesso gratuito ao painel existe para recompensar quem contribui dado real — não é uma porta grátis para quem só quer consultar. A defesa contra isso combina cinco mecanismos:

- **Prova de contribuição real e validada, não posse de wallet**: o acesso gratuito é liberado por atestação on-chain de dado validado (seção 3) naquele ciclo e região. Uma wallet sem contribuição não vê nada.
- **Verificação de CAR (Cadastro Ambiental Rural) contra a base pública do SICAR**: o oráculo de verificação é operado pelo próprio AgroBench, consultando a API pública do SICAR no onboarding, e confirma que o CAR informado existe, está ativo e ainda não foi vinculado a nenhuma outra wallet do protocolo. Essa checagem acontece uma única vez, fora da cadeia, e o resultado (aprovado/rejeitado) é o que é registrado on-chain — nunca o CAR em si. A propriedade continua pseudônima publicamente; o que muda é que toda contribuição está lastreada em uma propriedade real e verificável, o que torna fraude em escala uma questão de fraudar CARs reais, não de gerar contas.
- **Staking reembolsável**: contribuir exige travar um depósito em USDC, devolvido após a validação passar em ciclos consecutivos. Isso encarece a criação de wallets fake para obter acesso — sai mais barato para uma empresa assinar o plano pago diretamente.
- **Carência de três ciclos de contribuição validados**: a wallet só ganha acesso ao painel gratuito depois de ter dado validado em três ciclos seguidos (três safras ou três janelas de atualização, conforme a cultura). Isso elimina o padrão "entrar, capturar o dado, sumir".
- **O plano gratuito nunca mostra o que interessa para uma empresa**: o acesso gratuito do produtor mostra apenas os comparativos de custo e produtividade do nível básico (seção 6.2) da própria região e cultura. Solo, histórico de rotação de cultura e perdas por evento climático não aparecem no painel gratuito em nenhuma hipótese — mesmo quando o próprio produtor contribuiu esse nível de detalhe. Esse dado só existe agregado dentro do relatório pago, porque seu valor é para quem avalia risco de crédito e seguro, não para o benchmarking de custo do produtor.

### 7. Distribuição da receita (smart contract)

- O pagamento das assinaturas das instituições entra em um **pool de receita** controlado por smart contract na Solana.
- O contrato mantém o registro de quais wallets contribuíram para cada agregado (região × período × cultura).
- Ao fim de cada ciclo, a receita daquele agregado é **dividida proporcionalmente entre as wallets que contribuíram** para ele. Um agregado mais raro — cultura ou região sub-representada — distribui uma fatia maior por wallet, porque menos produtores dividem aquele pool. A ponderação também considera a granularidade do dado enviado (seção 6.2): um produtor com dado de nível avançado recebe peso maior na divisão do que um produtor com dado de nível básico.
- **O pool também sustenta a operação.** Antes da divisão entre produtores, o contrato desconta do pool (a) os custos de infraestrutura — validação em TEE, taxas de rede das transações on-chain (commit, atestação, distribuição) e o gas pago em nome dos produtores (seção 6.1) — e (b) uma taxa percentual fixa dos mantenedores do protocolo. Só o saldo restante é distribuído às wallets. Os percentuais são parâmetros do contrato, públicos e auditáveis on-chain.
- **Frequência de pagamento: mensal, em lote.** As assinaturas do período se acumulam no pool e a distribuição roda uma vez por mês, batendo o total arrecadado contra o registro de contribuições. Isso evita disparar uma transação a cada consulta paga de uma instituição — reduz custo de rede e dá previsibilidade de renda ao produtor.

## Por que essa arquitetura resolve a tensão original

Produtores não querem expor produtividade e custo individual — é informação competitiva. O setor precisa de dados agregados para funcionar melhor: crédito, seguro, política pública. O AgroBench separa identidade (wallet) de conteúdo (dado agrícola), garante que o dado bruto é visto uma única vez, dentro de um TEE, e nunca fica publicamente vinculado a uma pessoa ou propriedade — e cria um incentivo direto e recorrente (benchmarking gratuito + recompensa em USDC) para o produtor participar.

## Estado da demo (hackathon)

A **chain está ligada na Solana Devnet** (`adapters.chain=solana`): commit, atestação e CAR são Memo; a recompensa base sai em USDC-SPL da treasury para a wallet do produtor. O app gera a keypair no dispositivo (ed25519 + base58); o saldo em `GET /wallet` é o da Devnet.

O que ainda é mock: TEE (enclave no processo da API), SICAR, CONAB, SMS e pagamento (Stripe). Stake on-chain no hackathon é Memo — não há escrow Anchor travando USDC do produtor. O pool mensal é job no backend + transferência da treasury, não um programa próprio.

Treasury da demo: [6q35hKFa6vFEy1Huon58gTUs9nkXrgubn496BkAKNSsU](https://explorer.solana.com/address/6q35hKFa6vFEy1Huon58gTUs9nkXrgubn496BkAKNSsU?cluster=devnet) (`?cluster=devnet`). Guia: `SOLANA-DEVNET.md`.

## Limitações assumidas

Duas limitações são deliberadamente assumidas neste desenho, não escondidas:

1. **A validação depende de um TEE controlado pelo protocolo, não de zero-knowledge proofs.** Isso é pseudonimização com garantia de hardware, não anonimização criptográfica plena. A evolução para eliminar esse ponto de exposição é migrar a etapa de validação para aprendizado federado, onde o modelo treina localmente no dispositivo do produtor e apenas os pesos atualizados saem da propriedade — nenhuma parte, nem o validador, chega a ver o dado bruto.
2. **Existe um vínculo CPF → wallet, guardado como hash pelo AgroBench, usado para recuperação de conta e verificação de CAR.** Esse vínculo é o único ponto de identidade real no sistema e está sujeito a obrigações de proteção de dados pessoais sob a LGPD, incluindo base legal explícita, finalidade restrita (recuperação de conta e verificação de CAR) e direito de exclusão pelo titular. O modelo de recompensa em cripto atrelado a uma atividade econômica regular também será submetido a análise de enquadramento junto à CVM e ao Banco Central antes de operação em produção.
