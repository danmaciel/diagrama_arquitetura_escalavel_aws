# Plano de validação da arquitetura AWS

**Status:** testes planejados; nenhum resultado de implantação foi fornecido neste repositório.

O objetivo deste plano é transformar as decisões do diagrama em verificações reproduzíveis, com critérios de aceite acordados **antes** da execução.

## Preparação

1. Definir requisitos: disponibilidade/SLO, latência, volume de requisições, crescimento de dados, RPO, RTO e orçamento máximo.
2. Criar conta ou ambiente isolado, limites de custos, tags de recursos, IAM mínimo e processo de teardown.
3. Provisionar infraestrutura como código com revisão de diferenças e configuração em fonte versionada **sem segredos**.
4. Implantar uma aplicação simples com endpoint de saúde e uma transação de leitura/escrita no banco.
5. Instrumentar logs, métricas e alarmes, com timestamps padronizados para medir os experimentos.

## Matriz de validação

| ID | Cenário | Procedimento | Evidência necessária |
| --- | --- | --- | --- |
| V01 | Entrada pela borda | Requisição HTTPS pelo domínio/CloudFront | Response, certificado e métricas do ALB |
| V02 | Bloqueio de acesso direto | Tentar acessar a origem ALB sem os controles de origem esperados | Resposta negada e logs relevantes |
| V03 | Segurança de banco | Comprovar ausência de endpoint público e restrições de SG | Configuração e teste de rede |
| V04 | Perda de uma instância | Encerrar EC2 controladamente; acompanhar health checks/ASG | Linha do tempo, duração e erros observados |
| V05 | Aumento de demanda | Gerar carga gradual controlada | Throughput, latência p95/p99, escala e custos |
| V06 | Indisponibilidade de AZ | Exercício apropriado em ambiente não produtivo | Impacto observado, recuperação e capacidade restante |
| V07 | Failover RDS Multi-AZ | Executar failover controlado em ambiente de testes | Duração medida, perda de conexões e recuperação da aplicação |
| V08 | Recuperação por backup | Restaurar snapshot em ambiente isolado | Tempo de restauração, integridade e idade dos dados |
| V09 | Recuperação regional | Restaurar recursos em outra região e alterar apontamento | RTO/RPO medidos e checklist operacional |
| V10 | Alertas | Simular degradação e atingir limite de alarme | Eventos CloudWatch, notificação SNS e tempo até resposta |
| V11 | Saída privada e dependência NAT | Testar acesso externo após indisponibilidade do NAT da AZ | Falhas observadas e comparação com NAT por AZ/endpoints |
| V12 | Custos | Registrar consumo em ensaio com duração definida | Estimativa versus custo real e ações de teardown |

## Modelo de registro de experimento

Para cada ensaio, documentar:

- **ID do cenário** e data;
- **ambiente** e versão da infraestrutura;
- **hipótese** e métrica objetivo;
- **procedimento executado**, com condições iniciais;
- **resultado observado** e duração medida;
- **logs/gráficos/evidências** sem credenciais nem dados pessoais;
- **análise de desvios**, ação corretiva e decisão de aceite.

## Critérios de conclusão do estudo

O projeto só deve anunciar números concretos de RTO, RPO, desempenho ou disponibilidade **depois** que os testes correspondentes forem executados e suas evidências estiverem documentadas.

A mera presença de RDS Multi-AZ, Auto Scaling, backup ou múltiplas AZs não comprova uma meta quantitativa.

## Cuidados

Execute testes destrutivos somente em ambiente de laboratório e com recursos descartáveis. Use limites de orçamento e verifique a remoção dos recursos cobrados ao final. Não conduza experimentos de indisponibilidade em ambiente produtivo sem planejamento específico.
