# Notas técnicas — Laboratório de Sentinel

## 1. Provisionamento
- Entendi que o Sentinel não é um recurso independente: ele "habilita" sobre um
  Log Analytics Workspace já existente. Não dá pra criar o Sentinel sem workspace primeiro.
- Resource Group serve só como agrupamento lógico — não tem custo próprio, facilita
  organizar (e depois, se quiser, apagar tudo de uma vez).

## 2. Conector Azure Activity — o passo que mais me travou
- Instalar a solução no Hub de Conteúdo NÃO conecta o conector. É só o pacote
  disponível. Conectar de fato exige configurar uma "diagnostic setting", que
  hoje é feito via um assistente de Azure Policy (efeito DeployIfNotExists).
- Errei o escopo na primeira tentativa: apliquei a política no Resource Group
  (rg-sentinel-lab) em vez da assinatura inteira. Como o Log de Atividades é um
  recurso *da assinatura*, a política nunca achava nada pra corrigir
  (indicador: "0 de 0 recursos" na tela de Conformidade).
- Corrigi excluindo as atribuições erradas e recriando com o escopo certo
  (a assinatura "Azure for Students" como um todo, não o resource group).
- Lição: sempre conferir se o escopo escolhido no assistente é o nível certo
  pro tipo de recurso que a política afeta.

## 3. Primeiros dados
- Os primeiros 8 eventos que chegaram no workspace eram os próprios logs de
  criação da diagnostic setting (DIAGNOSTICSETTINGS/WRITE, POLICIES/DEPLOYIFNOTEXISTS).
  Achei interessante perceber que configurar o pipeline já gera dado dentro
  do próprio pipeline.

## 4. KQL — o que usei
- `take N` → olhar uma amostra bruta dos dados, sem filtro.
- `project campo1, campo2, ...` → escolher só as colunas relevantes, mais fácil
  de ler que a tabela crua.
- `where campo == "valor"` → filtrar por condição exata.
- Encadear dois `where` filtra por múltiplas condições (equivalente a "E").

## 5. Analytics Rule
- Regra: detectar exclusão de resource group (`RESOURCEGROUPS/DELETE` com
  status Success).
- Rodei "Rule runs (Preview)" achando que era um botão de "rodar agora" — na
  verdade é só o histórico de execuções passadas. Pra testar sem esperar o
  agendamento de 1h, o caminho certo é abrir a regra em modo edição e usar
  "Testar com dados atuais" na aba de lógica.
- Validei criando e deletando um resource group descartável (rg-teste-delete)
  e confirmando que o log gerado batia com a query da regra.

## 6. Custos / Azure for Students
- Nenhum recurso criado até aqui cobra por "estar ligado" — workspace e
  Sentinel cobram por volume de dado ingerido (GB), e estou bem abaixo do
  limite gratuito do trial de 31 dias.
- O único tipo de recurso que exigiria desligar entre sessões seria uma VM —
  não usei nenhuma neste lab.
- Configurei um alerta de orçamento no Cost Management como precaução.

## 7. Próximos passos (ideias para expandir o lab depois)
- Conectar uma segunda fonte de dados (ex: Microsoft Entra ID sign-in logs).
- Explorar automação de resposta (playbook simples com Logic Apps).
- Testar uma regra baseada em falha de autenticação, não só exclusão de recurso.

---

## [20/08/2026] — Fase 2: Entra ID e Defender for Cloud

Objetivo: conectar o Microsoft Entra ID como fonte de dados no Sentinel.

O que tentei primeiro:

Abri o conector "Microsoft Entra ID" em Data Connectors
Erro: Client 'heitor.francisco@cs.cruzeirodosul.edu.br' does not have authorization to perform action 'microsoft.aadiam/diagnosticSettings/read'
Causa: minha conta está no tenant institucional da faculdade (Cruzeiro do Sul), onde não tenho a role Global Administrator nem Security Administrator

Pesquisa de validação:

Busquei no Microsoft Q&A e encontrei um caso idêntico de outro estudante com Azure for Students
Confirmado: essa é uma limitação conhecida da conta, não um erro de configuração meu
Alternativa sugerida pela Microsoft (criar tenant próprio) não é viável sem cartão de crédito

Pivot para Defender for Cloud:

Instalei a solution via Content Hub
Conectei "Subscription-based Microsoft Defender for Cloud (Legacy)"
Habilitei só o plano gratuito (Foundational CSPM / "GPSN básico")
Descobri que o plano gratuito não gera SecurityAlerts (só recomendações de postura) — confirmado via KQL: SecurityAlerts | take 100 retornou 0 resultados

Terceira Analytics Rule:

Primeira versão da query observava MICROSOFT.INSIGHTS/DIAGNOSTICSETTINGS/WRITE — não funcionou
Debug: rodei AzureActivity | where OperationNameValue has "PRICINGS" | take 20 e descobri que alternar planos do Defender gera MICROSOFT.SECURITY/PRICINGS/WRITE
Corrigi a query da regra para usar essa operação

Incidente de custo evitado:

Durante o teste, ativei sem querer o plano PAGO do Defender ("GPSN do Defender", $5/recurso/mês) em vez do gratuito
Desativei imediatamente (0 recursos na assinatura = sem cobrança gerada, mas corrigido por precaução)

Problema de tuning identificado:

Ao mudar o lookback da regra para 24h, ela recriou o mesmo alerta a cada execução horária → 16 incidentes duplicados para o mesmo evento
Corrigido revertendo o lookback para 1h (igual à frequência de execução)
Fechei 15 incidentes em lote como duplicatas, triei o restante como Benign Positive

Workbook criado:

"Visão Geral - Atividade da Assinatura" com 3 gráficos: top operações, top callers, linha do tempo


Se algo não aparecer como esperado, me manda um print da tela do terminal (com os comandos e o resultado) que eu te ajudo a diagnosticar.
