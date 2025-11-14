# Documentação Técnica — Daemon Bayesiano

Este arquivo complementa `README-DAEMON.md` e contém uma versão orientada à navegação, além de um diagrama mermaid disponível em `docs/diagrams.mmd`.

## Conteúdo

- Overview
- Arquitetura
- Fluxo de execução
- Módulos (sensores, memória, policy, actuators)
- Operação e deploy

## Notas rápidas de deploy

O repositório inclui `Bansky/prototipos/proto-AGI/install.sh` — um instalador que escreve o script em `/usr/local/bin/bayes_opt.sh` e cria uma unit file systemd `bayes_opt.service`.

Antes de rodar em produção:

- revisar `install.sh`
- testar em VM/container
- garantir `lm-sensors`, `intel_rapl` (se aplicável) e `zram` suportados

## Diagrama

O diagrama principal está em `docs/diagrams.mmd` (Mermaid). Abra em um visualizador com suporte a Mermaid para ver o fluxo gráfico.
