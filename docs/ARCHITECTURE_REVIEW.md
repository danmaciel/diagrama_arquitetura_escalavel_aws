# Revisão técnica das decisões arquiteturais

**Status:** análise de projeto; recomendações não implantadas.

## 1. Disponibilidade por zona

O desenho apresenta EC2 em três zonas, ALB em duas sub-redes públicas e RDS em duas AZs. Isso **não significa** que todos os recursos são automaticamente tolerantes à perda de qualquer AZ. Para uma implantação:

- Criar sub-redes privadas de aplicação adequadamente dimensionadas em cada AZ utilizada pelo Auto Scaling Group.
- Manter o ALB em pelo menos duas AZs, com target groups e health checks efetivos.
- Distribuir instâncias e dimensionar capacidade de reposição após falhas, sem supor que `min=3, max=6` representa 2 EC2 por AZ.
- Definir políticas de Auto Scaling, warm-up, tempo de inicialização, drenagem de conexões e testes de indisponibilidade.

**Trade-off:** maior capacidade ociosa eleva o custo, mas reduz o tempo para reagir a falhas.

## 2. Acesso externo e defesa em profundidade

O WAF do desenho está associado à distribuição CloudFront, onde filtra solicitações na borda. O ALB é a origem HTTPS em sub-redes públicas; o Security Group do ALB pode usar a prefix list gerenciada do CloudFront. Isso, isoladamente, não diferencia *qual* distribuição CloudFront originou a requisição. Avaliar também um header de origem compartilhado, validado em regras do ALB/WAF, com controle e rotação do valor. Para novos desenhos, avaliar ainda a alternativa **CloudFront VPC origins** com ALB privado, quando adequada ao caso.

- TLS entre navegador–CloudFront e CloudFront–ALB, certificados e redirecionamento HTTP→HTTPS.
- Security Group da aplicação aceitando somente a origem do ALB, e banco aceitando somente a aplicação.
- Credenciais no Secrets Manager ou autenticação IAM do RDS; evitar chaves permanentes e segredos embutidos.
- IAM Role mínima para EC2; auditoria de ações e políticas em implantação real.

**Trade-off:** medidas de segurança acrescentam configuração, custo operacional e pontos de observação; devem ter evidências de teste.

## 3. Saída das sub-redes privadas

Um NAT Gateway único numa determinada AZ pode deixar o egress das demais AZs indisponível durante uma falha na AZ desse NAT. Para cargas que precisam de internet e HA, avaliar:

- NAT Gateway em cada AZ e roteamento local para sua AZ.
- VPC endpoints para serviços AWS com suporte, reduzindo dependência de NAT.
- Alternativas de custo para ambientes de demonstração, esclarecendo a perda de resiliência.

O NAT **não é necessário para toda comunicação privada**: EC2→RDS usa rede privada da VPC.

## 4. Banco e objetivos de recuperação

RDS PostgreSQL Multi-AZ tradicional mantém um standby síncrono em outra AZ e automatiza o failover. Esse standby não aumenta a capacidade de leitura. Os números anunciados no desenho original — RTO inferior a dois minutos e RPO de minutos — **não foram demonstrados**. A documentação AWS aponta duração típica de failover de 60–120 segundos, com variações; a aplicação ainda deve lidar com conexões interrompidas, retry apropriado e atualização de DNS.

- **HA:** reduzir impacto de falha local/da AZ usando Multi-AZ.
- **Backup:** ter cópias restauráveis, retenção e testes de recuperação, sem confundir backup com standby.
- **DR regional:** copiar backups para outra região e documentar o processo completo de restauração, ambiente de computação, DNS, segredos e reconexão.
- **Metas RPO/RTO:** são requisitos e resultados *medidos*; só podem ser afirmados depois de exercícios concretos.

**Trade-off:** RTO/RPO agressivos exigem maior investimento e mecanismos de replicação/standby.

## 5. Observabilidade

CloudWatch pode reunir métricas de EC2, ALB e RDS, mas métricas de memória de EC2 exigem coleta adicional (por exemplo, CloudWatch Agent). Alarmes/SNS precisam de limites, destinos e rotinas de resposta comprovados. É recomendável testar alarmes e realizar exercícios de diagnóstico.

## 6. Custo e operação

Sem região/instâncias/tamanho de armazenamento/tráfego, não há cálculo confiável. Os maiores vetores de custo provavelmente envolvem EC2 com capacidade mínima, RDS Multi-AZ, NAT Gateways/transferência entre AZs, ALB, tráfego CloudFront e armazenamento de backups. Dimensionamento deve partir de requisitos e precificação atual.

## 7. Decisões ainda abertas

| Decisão | Opções | Informação necessária |
| --- | --- | --- |
| ALB público ou privado | ALB público com restrições; CloudFront VPC origins | Compatibilidade, operação e segurança |
| NAT Gateway | Único (demo) ou por AZ | Disponibilidade e custos |
| Autenticação no RDS | IAM DB Auth; Secrets Manager | Linguagem, driver, política de rotação |
| Escala EC2 | CPU/requests; métricas customizadas | Perfil de carga e tempo de boot |
| DR | Backup restaurável; warm standby; active-passive regional | RPO, RTO e orçamento |
| Provisionamento | Terraform; AWS CDK; CloudFormation | Requisitos da equipe e CI |

## Evidências disponíveis

Diagrama PNG e fonte Draw.io, além de documentação técnica. **Não há** medições de performance, IaC ou logs de failover neste repositório.

## Referências

- [RDS Multi-AZ failover](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZ.Failover.html)
- [NAT Gateway basics](https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-basics.html)
- [CloudFront — Restrict access to ALBs](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/restrict-access-to-load-balancer.html)
