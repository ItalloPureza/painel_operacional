# Guia para IA — manutenção do Painel Operacional

> Leia este arquivo inteiro antes de mexer em qualquer coisa. Ele foi escrito para uma IA (ou pessoa técnica) que vai
> ajudar alguém a manter este painel. Tudo aqui foi aprendido na prática, em um painel real de produção do qual este
> modelo é uma cópia com dados fictícios.

## 1. O que é este projeto

Um painel de operação em **Power BI** de uma empresa SaaS fictícia, **Alfa**, com dois produtos: **Alfa** (sigla
**AF**, o principal) e **Kappa** (sigla **KP**). Ele reúne, em 56 páginas, os números de 8 setores:

| Setor | Telas | Assunto |
|---|---|---|
| Gerência | 6 | Cohort AF/KP, inativações, resumos por setor e por produto |
| Global | 10 | Cancelamentos, churn financeiro, carteiras, idade dos clientes, TTFV, migrações |
| Lives | 4 | Participação, desempenho, clientes e CSAT das lives (ids `pg_agendamento_*`) |
| Suporte | 4 | Resultados, CSAT, atendimentos, empresas |
| Onboarding | 11 | Performance, No Show, cancelamentos, SLA de atividades, migrações, ligações, WhatsApp |
| Ongoing | 11 | O mesmo do Onboarding para a carteira em acompanhamento |
| Relacionamento | 5 | Cancelamentos, reversões, Ouvidoria, clientes, reversão × concessões |
| Homologação | 4 | App, guias, CSAT, tempo de entrega |

Mais a **HomePage** (portal com um card por setor, cadeado de acesso e busca de painéis) e a tela **Saúde dos dados**.

O projeto está no formato **PBIP** com relatório **PBIR** (cada página e cada visual é um arquivo JSON) e modelo em
**TMDL** (cada tabela é um arquivo texto). Isso permite editar tudo por script, sem abrir o Power BI.

Todos os dados vêm de **uma planilha** (`dados/dados_exemplo.xlsx`, uma aba por tabela) com pessoas, clientes, CNPJs,
valores e datas **inventados**. Período dos dados: jan a 26/10/2026 (o painel mostra de ago/2026 em diante).

## 2. Pastas

```
painel-operacional-modelo/
├─ Painel Operacional.pbip                 # abra este no Power BI Desktop
├─ Painel Operacional.Report/
│  ├─ definition/
│  │  ├─ report.json                       # tema, filtros do relatório, recursos registrados
│  │  └─ pages/
│  │     ├─ pages.json                     # ordem das páginas e página ativa
│  │     └─ <pagina>/page.json + visuals/<visual>/visual.json
│  └─ StaticResources/RegisteredResources/ # logo, ícones, tema (JSON)
├─ Painel Operacional.SemanticModel/
│  └─ definition/
│     ├─ model.tmdl, expressions.tmdl      # parâmetro CaminhoPlanilha
│     ├─ relationships.tmdl
│     ├─ roles/Setor.tmdl                  # segurança por linha (RLS)
│     └─ tables/*.tmdl                     # colunas, medidas, partições (Power Query)
├─ dados/dados_exemplo.xlsx                # a fonte de tudo
├─ README.md                               # para pessoas
└─ GUIA-PARA-IA.md                         # este arquivo
```

Nomes internos de página são estáveis (`pg_onboarding_05`); o texto visível é o `displayName`. A HomePage tem o nome
interno `660767115bcfb826629b`.

## 3. Regra de ouro (a mais importante)

**Nunca o Power BI Desktop e um script escrevem ao mesmo tempo.** Ao salvar, o Desktop grava o que tem em memória por
cima dos arquivos e desfaz qualquer edição feita no disco enquanto ele estava aberto.

1. Peça para a pessoa **salvar e fechar** o Desktop.
2. Edite os arquivos.
3. Valide (seção 10) e só então peça para reabrir.

Se a pessoa mexeu no Desktop, peça para salvar e fechar antes de você ler os arquivos: senão você lê uma versão velha.
Sintoma de conflito: "minhas mudanças sumiram depois que salvei".

Ao abrir ou fechar o Desktop por script, controle só a instância que você abriu (guarde o PID e feche por ele, nunca
por nome do processo: a pessoa pode ter outro painel aberto).

## 4. Como os dados entram

- Parâmetro **`CaminhoPlanilha`** (em `expressions.tmdl`): caminho absoluto da planilha. Ao mudar a pasta de lugar,
  troque o parâmetro (Desktop: Transformar dados → Editar parâmetros) e atualize.
- Cada tabela "de fonte" tem uma partição Power Query igual a esta:

```
let
    Fonte = Excel.Workbook(File.Contents(CaminhoPlanilha), null, true),
    Aba = Fonte{[Item="Helpdesk_Tickets",Kind="Sheet"]}[Data],
    Cabecalhos = Table.PromoteHeaders(Aba, [PromoteAllScalars=true]),
    Tipos = Table.TransformColumnTypes(Cabecalhos, {{"id", Int64.Type}, ...})
in
    Tipos
```

- **O cabeçalho de cada aba = o `sourceColumn` de cada coluna da tabela no TMDL.** Uma coluna a menos na aba quebra a
  atualização daquela tabela ("coluna não encontrada"); uma a mais é ignorada.
- O nome da aba é o nome da tabela cortado em 31 caracteres (limite do Excel).
- **Trocar a planilha por uma fonte real** (banco, API): substitua só a partição da tabela, desde que a consulta nova
  devolva **as mesmas colunas com os mesmos nomes e tipos**. Todo o resto (colunas calculadas, medidas, telas) continua.
  Use parâmetros para servidor e banco, como `CaminhoPlanilha`, para trocar de ambiente sem editar consultas.
- Ao importar uma tabela nova de um banco, **nunca traga colunas secretas** (chaves de API, tokens, IDs de gateway de
  pagamento, hashes de senha): remova-as no Power Query logo depois da navegação (`Table.RemoveColumns(...,
  MissingField.Ignore)`) e não crie a coluna no TMDL. Elas iriam para qualquer pessoa que abrir o arquivo publicado.
- Datas de APIs que gravam em UTC (o helpdesk, no painel original) devem ser convertidas para o horário local **no
  Power Query**, logo depois da leitura (`DateTimeZone.SwitchZone(..., -3)`), e só nas colunas que estão em UTC.

## 5. Tabelas

### Fontes (vêm da planilha)
| Grupo | Tabelas | O que é |
|---|---|---|
| Helpdesk | `Helpdesk_Tickets`, `Helpdesk_Survey`, `Helpdesk_Cancelamentos`, `Helpdesk_Empresas`, `Helpdesk_Clientes`, `Helpdesk_Clientes_Empresas`, `Helpdesk_Chat`, `fConcessoes` | Tickets de todos os serviços (suporte, reuniões, lives, homologação, ouvidoria), avaliações (CSAT), pedidos de cancelamento, chat mensal, concessões |
| CRM de Customer Success | `CRM_Atividades`, `CRM_Backup_Diario`, `CRM_Clientes_Atual` | Atividades dos analistas; **retrato diário** da carteira (um registro por cliente por dia: status, dono, uso) |
| Sistema de contratos | `Contratos_historico_*` (12 tabelas) | Retrato dos contratos e itens (mensalidade por produto) em datas de corte; `status_id` 3 e 5 = ativo |
| Portal e sistema legado | `Gestao_Contratos_*`, `Legado_Clientes` | Contratos aprovados no portal; consulta de clientes do sistema legado |
| WhatsApp | `WhatsApp_Atendimentos`, `WhatsApp_CSAT`, `WhatsApp_Envios`, `WhatsApp_Contatos` | Conversas, notas, disparos de templates, contatos |
| Telefonia | `Ligacoes_API` | Ligações por ramal |
| Intranet | `Time_Usuarios`, `Time_Equipes`, `Time_EquipeSetor`, `RH_ComposicaoMensal`, `Meta_Def`, `Meta_Valor` | Pessoas, equipes, setor de cada equipe, composição das equipes **mês a mês**, metas |

### Calculadas (DAX, não vêm da planilha)
- `Calendario` (2020–2030; hierarquia `'Ano e Mês'` usada pelo filtro Período; a data/hora automática está
  **desligada** de propósito) e o grupo de cálculo **`Competência`**.
- Pessoas e acesso: `dColaborador` (composição atual), `dColaboradorMes` (pessoa × mês), `dPessoaSetor` (ramal),
  `dMapaSetor` (equipe → setor do painel), `dAcessoSetor` (quem vê qual setor), `dSetorPagina`, `dDeParaOwnerTeam`
  (fila do helpdesk → setor).
- Carteira e cliente: `dCarteira`, `dTTFV`, `fPeriodos`, `fCohort`, `dMesDeVida`, `fInativacoes`, `fMigracoesClientes`,
  `fGlobalCancelamentos`, `fGlobalReceita`, `dBuscaCliente`.
- Navegação e saúde: `dPaineis` (busca de painéis da Home), `dReferenciaDados` ("Dados até" de cada tela),
  `dSaudeTabelas`, `dPaginaTabelas`; `Meta_Vinculo` (medida do painel → meta da intranet).
- A tabela `Atualizacao` guarda medidas gerais (acesso, navegação, "Atualizado em").

### Relações
`Calendario[Date]` filtra cada fato por **uma** data ativa; as outras datas do fato são relações **inativas**, usadas
só em medidas com nome explícito via `USERELATIONSHIP`. Não há relação bidirecional nem muitos-para-muitos — mantenha
assim. Tabelas sem relação entre si conversam por `TREATAS` dentro das medidas.

## 6. Regras de negócio (o "porquê" de cada número)

1. **Uma data por tipo de número.** Ticket e cancelamento: **fechamento**. SLA de ticket: contado pelo fechamento,
   medido até a resolução. No Show e Lives: **criação** (é o dia da reunião/live). CSAT: data da avaliação. Ligação:
   início. Atividade: conclusão. Carteira/status/mensalidade: o retrato do dia. Um único filtro de período por tela,
   sempre pelo `Calendario`.
2. **Equipe do mês, não a atual.** Cada fato tem uma coluna oculta `Mês da Composição` (aaaamm, pela data que a tela
   usa) e busca equipe/setor em `dColaboradorMes` naquele mês. Quem mudou de equipe continua contando para a equipe
   antiga nos meses antigos. Medidas "(por pessoa)" usam `TREATAS( SUMMARIZE( dColaboradorMes, email, AnoMes ), <email
   do fato>, <fato>[Mês da Composição] )`. Mês sem composição gravada = cópia do último, marcada "Provisória".
3. **Quem vê o quê** (segurança):
   - `dAcessoSetor` (email, Setor; `*` = tudo) é a regra única. Admin, diretoria, gerentes e a equipe **Projetos** veem
     tudo; os demais veem os setores das equipes em que estão vinculados (vínculo **atual**); todos veem o Global.
   - Função RLS `Setor` (em `roles/Setor.tmdl`): `"*" IN meus || Fato[Setor] IN meus`. No Desktop a conta do Windows
     (UPN com `\`) tem acesso total, para testar a navegação; no Service o UPN é o e-mail.
   - Medidas `Tem Acesso`, `Transp Cadeado`, `Mensagem Acesso` e `Nav <Setor>` cuidam do cadeado na Home e da
     navegação condicional (bloqueado volta para a Home). **`Nav <Setor>` devolve o `displayName` exato da 1ª tela do
     setor** — renomeou a tela, ajuste a medida.
   - Equipe nova entra sozinha se o cadastro da equipe tiver um `setor_id` com tela (2 Ongoing, 3 Onboarding, 4
     Suporte, 5 Homologação, 6 Relacionamento, 17 Lives). Setor novo com tela = editar `dMapaSetor` e
     `dColaboradorMes` (mesmo mapa nos dois).
   - Exceção decidida: `dCarteira`, `dTTFV` e o retrato diário **não têm RLS** (alimentam o Global, aberto a todos).
     Não exponha essas tabelas numa tela de setor sem pensar nisso.
4. **Cartões** (grupo de cálculo `Competência`, filtro oculto em cada cartão `*_kpi_N`): item **"Período inteiro"**
   para números que acontecem no tempo (tickets, cancelamentos, ligações, CSAT); item **"Mês mais recente"** para fotos
   (carteira, pendências) e percentuais sobre a base do 1º dia do mês. Sem valor: 0 para contagem e R$; "—" para
   percentual, média e texto.
5. **"Dados até"** = data do registro mais recente da **tabela principal da tela** (extração > atualização > data do
   registro), sem filtros e sem RLS. Fica em `dReferenciaDados`, uma linha por tela, filtrada por um filtro de página
   oculto.
6. **Metas**: `Meta_Def` (definição, escopo empresa/setor/área), `Meta_Valor` (alvo por mês; percentuais em **pontos**:
   2,10 = 2,10%; tempo em segundos), `Meta_Vinculo` (qual medida usa qual meta). Metas de quantidade e de R$ são
   **derivadas**: meta % × clientes (ou mensalidade) do 1º dia do mês. Uma meta só aparece quando o recorte do
   visual é o mesmo dela (um produto, um serviço); senão fica vazia de propósito.
7. **CSAT**: escala 1 a 10; nota 0 conta como 1 (`MAX( nota, 1 )`) em toda conta de CSAT.
8. **Contas "de hoje"** (dias sem contato, dias em aberto) contam até a **data dos dados**, não até `TODAY()`: sem
   atualização, o número não pode crescer sozinho.
9. **Filtro que não chega** (fatos sem relação com a coluna da lateral): a medida testa `ISFILTERED( T[coluna] )`
   (nunca a tabela inteira) e segue por `TREATAS`; sem campo equivalente, fica **vazia** com o filtro — vazio é melhor
   que um total sem recorte.
10. **Filtro de período**: filtro de relatório `Calendario[Date] >= 01/08/2026`; o slicer Período mostra Ano = 2026 e
    meses ago–dez. **Na virada do ano, inclua o ano novo à mão** no slicer (Ano e Mês são independentes).

## 7. Padrões visuais (valem para toda tela nova)

- Canvas fixo **1920 × 1080**; faixa azul no topo (título, abas do setor, botão Início, "Atualizado em" e "Dados até"
  à direita); lateral esquerda de filtros em grade (caixas de 74 px, 12 px entre elas).
- Ordem dos filtros: **Período > Pessoas** (setor/área > equipe > pessoa) **> Sistema** (produto) **> Outros** (ID e
  CNPJ do cliente primeiro).
- Datas **dd/mm/aa** (eixo por mês pode ser `mmm/aa`); número com vírgula **sempre com 2 casas**; contagem sem
  vírgula; **números cheios** (nunca "14 Mil": `labelDisplayUnits = 1D`); horas `hh:mm:ss` (mesmo acima de 24 h).
- Gráficos de barra/coluna por categoria em ordem **A→Z** (a posição de cada pessoa não muda); exceções: eixos de
  tempo, faixas e escalas de nota.
- Listas com uma linha por registro não levam coluna de contagem (daria sempre 1): mostram identificação.
- "?" de ajuda de 18 px no canto superior direito de cada quadro, explicando a conta.
- Só a HomePage visível no Service: toda outra página tem `"visibility": "HiddenInViewMode"` no `page.json`; a
  navegação é pelos botões. Página nova nasce oculta.
- Fonte **Segoe UI** em tudo (fontes fora da lista do Power BI não existem no Service e o texto some).

## 8. Armadilhas do PBIR/TMDL (cada uma custou horas)

**Formato de visual**
- Descobrir um formato JSON: peça para a pessoa criar o elemento à mão no Desktop, salvar e fechar; leia o
  `visual.json` e copie. Não adivinhe.
- Botão: sem `transparency: "0D"` no fill a cor fica "lavada". Borda padrão do botão vem **ligada**: desligue com
  `objects.outline` `show:false` **sem** `selector`. Desligue também `visualContainerObjects.background` e `border`.
- Canto arredondado: `objects.shape` `tileShape:'rectangleRounded'` + `rectangleRoundedCurve` (inteiro, % do lado
  menor; ~12 px = `round(12*100/lado_menor)`).
- Botão "sem fundo": `fill.show = false` sem selector (com `selector: default` o tema pinta de azul).
- Texto cortado embaixo em Segoe UI: altura do botão de texto ≥ 2,2 × o tamanho da fonte (pt).
- Imagem/botão transparente **ainda captura o clique**. Nunca ponha uma caixa por cima de área clicável; texto e
  ícone por cima de um card clicável recebem a mesma navegação do card.
- Imagem substituída com o mesmo nome continua em cache: use um nome novo (`_v2`) e reaponte os visuais.
- Tabela (`tableEx`): não ponha `"active": true` nas projeções (só aquela coluna aparece).
- Camadas: o Desktop renumera o `z` de 1000 em 1000 ao salvar; visual novo com z baixo fica atrás dos quadros brancos.
- Valor de filtro (`filterConfig` → `Where` → `In` → `Values`) é o `{"Literal":{"Value":"'x'"}}` **puro**. Com
  `{"expr": …}` o Desktop recusa o visual.
- Formatação condicional (`FillRule`): cores como `{"Literal":{"Value":"'#hex'"}}` **sem** `expr`; com `expr` o
  Desktop desenha, mas qualquer edição depois quebra com "Erro ao renderizar o relatório".
- Navegação fixa usa o **nome interno** da página; navegação por medida usa o **displayName**.

**DAX**
- Medida em cartão que devolve `BLANK()` mostra "(Em branco)"; devolva `""` quando o vazio deve ficar vazio.
- Medida que filtra uma coluna que também está no eixo/filtro da tela: `KEEPFILTERS(...)`, senão todas as células
  mostram o mesmo total.
- "Maior valor por cliente e item" dentro de `SUMX( SUMMARIZE(...) )` com `CALCULATE( MAX() )` erra em gráficos com
  várias linhas: use `GROUPBY` + `MAXX( CURRENTGROUP() )`. Confira com `SUMMARIZECOLUMNS` (como o visual consulta).
- `MAXX`/`MINX` não aceitam coluna booleana: a tabela calculada inteira entra em erro e tudo que depende dela some.
  Use `IF( col, 1, 0 )`.
- Grupo de cálculo é aplicado também às medidas de dentro: use `REMOVEFILTERS( 'Competência' )` no `CALCULATE` do item.
  Exige `discourageImplicitMeasures` (já ligado).
- Formato com vírgula no fim (`#,0,`) não divide por mil no Desktop: divida na medida.

**Modelo e memória**
- Coluna calculada com filtro que depende da linha sobre coluna de alta cardinalidade estoura a memória no recálculo:
  reduza antes (`CALCULATETABLE( VALUES(...), <filtros da linha> )`) e só então compare.
- Coluna nova numa tabela grande (com muitas colunas calculadas e dependentes) faz o Desktop recalcular tudo ao abrir.
  Campo novo para uma tela só = consulta própria e pequena.
- Consulta nativa nova (`Value.NativeQuery`) exige aprovação manual no Desktop; prefira navegação + `Table.SelectRows`.
- Caminhos com mais de 260 caracteres: o Desktop abre um relatório vazio sem avisar. Trabalhe em pastas curtas.

## 9. Tarefas comuns

**Nova tela**
1. Copie a pasta de uma tela parecida em `pages/` com um nome interno novo (`pg_<setor>_NN`); troque `name` e
   `displayName` no `page.json`; mantenha `"visibility": "HiddenInViewMode"` e o filtro de página do setor
   (`dSetorPagina[Setor]`).
2. Ponha o nome interno em `pages.json` (`pageOrder`).
3. Ajuste os visuais (cada `visual.json` tem um `name` único na página).
4. Adicione a tela às abas do setor (botões de navegação nas outras telas do setor), a `dPaineis` (busca da Home) e
   a `dReferenciaDados` ("Dados até").
5. Valide (seção 10).

**Nova medida**: em `tables/<tabela>.tmdl`, bloco `measure 'Nome' = <DAX>` com `formatString` no padrão (2 casas,
`dd/mm/yy`), `displayFolder` e `lineageTag` novo (GUID). Descreva a regra numa linha `///` acima da medida.

**Nova tabela de fonte**: crie a aba na planilha, o arquivo `tables/<Tabela>.tmdl` (colunas com `sourceColumn` e
partição no molde da seção 4) e `ref table <Tabela>` em `model.tmdl`. Se o fato tiver setor, crie a coluna `Setor`
e acrescente a tabela à função RLS.

**Virada do ano**: no slicer Período de cada tela, inclua o ano novo e os meses dele (filtros ocultos `periodoAno` /
`periodoMes`).

## 10. Como validar sem quebrar nada

1. JSON: carregue todos com um único processo (`python -c "import json,glob; [json.load(open(f,encoding='utf-8')) for
   f in glob.glob('**/*.json', recursive=True)]"`). Grave sempre em UTF-8 com `ensure_ascii=False` (acentos).
2. Modelo: valide o TMDL sem abrir o Desktop com a DLL que vem com ele:
   `Add-Type -Path "C:\Program Files\Microsoft Power BI Desktop\bin\Microsoft.PowerBI.Tabular.dll";
   [Microsoft.AnalysisServices.Tabular.TmdlSerializer]::DeserializeDatabaseFromFolder("<...>\definition")`.
3. Números: com o painel aberto, o motor local (msmdsrv) aceita consultas DAX pela `Microsoft.PowerBI.AdomdClient.dll`
   da mesma pasta. A porta está no processo `msmdsrv` em LISTEN (`Get-NetTCPConnection -OwningProcess`). Use para
   comparar números antes × depois de uma mudança. Só leia; feche sem salvar.
4. Teste em **uma** tela antes de replicar em todas.

## 11. Nomes deste modelo

| Nome | Significado |
|---|---|
| Alfa (AF), Kappa (KP) | Os dois produtos fictícios; "Só Kappa" = cliente sem o produto principal (ID não numérico) |
| Helpdesk | Sistema de tickets e atendimento (suporte, reuniões, lives, homologação, ouvidoria, cancelamentos) |
| CRM | Plataforma de Customer Success: atividades dos analistas e retrato diário da carteira |
| Contratos | Sistema de contratos e mensalidades (retratos em datas de corte) |
| Legado | Sistema de clientes antigo, consultado por API; "ID Legado" é o código do cliente nele |
| WhatsApp | Plataforma de atendimento e disparos pelo WhatsApp |
| Status de Saúde | Saúde do cliente no acompanhamento: Cruzeiro (melhor), En Route, Alert, Mayday; TakingOff/Runway = em implantação |
| Gama | Uma segunda linha de contratos do portal |
| Retrato | Foto diária da carteira (um registro por cliente por dia) |
| Carteira | Clientes ativos (e bloqueados/suspensos) de um analista, equipe ou da empresa |
| No Show | Reunião marcada que não aconteceu (falta, remarcação, tentativa de contato) |
| TTFV | Tempo até o cliente ter o primeiro valor (usar ≥ 30% das empresas do contrato), contado por entrada do cliente |
| Cohort | Clientes agrupados pelo mês de entrada, acompanhados mês a mês de vida |
| Reversão | Pedido de cancelamento que terminou com o cliente ficando |
| Concessão | Desconto, redução ou isenção concedida numa negociação |
| N2 | Segundo nível de uma equipe; vê só os setores das equipes em que está vinculado |
| Composição | Quem estava em cada equipe em cada mês |

## 12. Sobre os dados fictícios

- Gerados por script com semente fixa: ~900 clientes, ~120 pessoas, ~20 mil tickets, retrato diário de 14/07 a
  26/10/2026. Não há nenhum dado real. Conferido: as 39 tabelas carregam da planilha sem erro em ~5 min.
- A tela Saúde dos dados mostra fontes "atrasadas": os dados param em 26/10/2026 e a conta é contra a hora atual.
- Alguns visuais podem mostrar pouco ou nada (ex.: colunas que dependem de campos que a planilha deixa vazios, como
  aviso prévio). Isso é do exemplo, não do painel.
- Para recriar a planilha com outro tamanho ou outro período, foi usado um gerador próprio (não incluído). Para usar
  dados reais, siga a seção 4: troque as partições, mantendo as colunas.
