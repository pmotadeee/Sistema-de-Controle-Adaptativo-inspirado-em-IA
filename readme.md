# Sistema de Controle Adaptativo — Resumo (README)

Este repositório contém um protótipo de "daemon bayesiano" para otimização adaptativa de sistemas Linux (governor de CPU, TDP via Intel RAPL, configuração dinâmica de ZRAM). A documentação técnica completa foi extraída para `README-DAEMON.md` e `docs/`.

Principais artefatos

- `README-DAEMON.md` — documentação técnica completa (whitepaper) ✅
- `docs/daemon.md` — documentação navegável e notas de deploy ✅
- `docs/diagrams.mmd` — diagrama Mermaid do fluxo principal ✅
- `Bansky/prototipos/proto-AGI/install.sh` — instalador / systemd unit (script Bash)

Use este README para um quick-start seguro e referência rápida.

## Quick start — inspeção e testes (seguro)

1) Revise o instalador antes de executar (sempre):

```bash
# visualize the installer
sed -n '1,220p' Bansky/prototipos/proto-AGI/install.sh
```

2) Teste em uma máquina de desenvolvimento ou VM (recomendado). Exemplo usando uma VM/host Ubuntu:

```bash
# atualização e dependências (na VM)
sudo apt update && sudo apt install -y lm-sensors util-linux zram-tools

# executar o instalador (irá escrever /usr/local/bin/bayes_opt.sh e criar unit systemd)
sudo bash Bansky/prototipos/proto-AGI/install.sh

# checar status do serviço (systemd)
sudo systemctl status bayes_opt.service --no-pager

# ver logs do daemon
sudo journalctl -u bayes_opt.service -n 200 --no-pager
sudo tail -n 200 /var/log/bayes_mem/bayes.log
```

Observação: rodar o instalador no seu desktop/servidor de produção altera `sysfs` e módulos do kernel (zram, RAPL). Use uma VM/container isolado para testes.

3) Teste manual sem instalar o service (execução direta)

```bash
# executar o binário diretamente (útil para debugging)
sudo /usr/local/bin/bayes_opt.sh &
tail -f /var/log/bayes_mem/bayes.log
```

Se `/usr/local/bin/bayes_opt.sh` não existir (não instalado), copie o conteúdo do `install.sh`'s created script localizado no repositório e execute localmente para inspecionar o comportamento antes de instalar como service.

## Recomendações de segurança e portabilidade

- Sempre revisar `install.sh` antes de executar. Ele faz writes em `/sys` e modprobe/rmmod de módulos do kernel.
- Confirme suporte do host para Intel RAPL e ZRAM. Caso as interfaces não existam, o daemon precisa de fallbacks.
- Use VMs/instâncias temporárias para validar comportamento (ou um host de teste com permissões isoladas).

## Links úteis

- Documentação completa: `README-DAEMON.md`
- Docs: `docs/daemon.md`
- Diagrama: `docs/diagrams.mmd`
- Instalador: `Bansky/prototipos/proto-AGI/install.sh`

## Próximos passos sugeridos

1. Testes em VM/container e revisão do script `install.sh`.
2. Se desejar maior testabilidade, considerar conversão do daemon para Python (PoC) — eu posso gerar isso.
3. Adicionar CI/CD com checks estáticos e um simulador de carga para validar políticas.

---

Nota: não alterei o script Bash; mantive o design em Bash conforme solicitado. Se quiser que eu adicione comentários multilinha em pseudocódigo no `install.sh` (sem alterar a lógica), eu posso inserir comentários no arquivo de cópia e deixar a original intacta — me diga se quer que eu proceda.


1. Lê métricas de CPU (uso, load, variação, temperatura)
2. Cria uma **média móvel histórica**
3. Converte isso num **perfil discreto (000–100)**
4. Aplica esse perfil em:

   * Governor da CPU
   * TDP (Intel RAPL)
   * ZRAM (streams + algoritmo)
   * (Turbo Boost — desativado no código)

E roda **a cada 5 segundos**, logando tudo.

---

# 📁 **ESTRUTURA DOS ARQUIVOS**

O daemon usa um mini “/var/lib” privado:

```
/etc/bayes_mem/
    cpu_history           # memória da RNN degenerada
    cpu_trend.log         # não muito usado, mas existe
    last_*                # estado persistido de governor, zram, power, etc
/var/log/bayes_mem/
    bayes.log             # log contínuo do daemon
```

---

# 🔌 **CICLO PRINCIPAL DE EXECUÇÃO**

A função **main()** faz:

```
initialize_directories  
loop infinito:
    printa info no log
    apply_all
    sleep 5
```

Ou seja:

> A cada 5 segundos, ele roda inferência → aplica ações → loga tudo.

---

# 🧩 **1. COLETA DE SINAIS (os sensores)**

### 🔥 CPU Usage (sensor principal)

Ele lê stat duas vezes, compara os deltas e calcula:

```
usage = (total_diferenca - idle_diferenca) / total_diferenca * 100
```

Isso é padrão de medição de CPU real (como top, htop fazem).

### 🌡 Temperatura

Ele usa o `sensors`, pega o **primeiro valor em °C** e corta a parte decimal.

Fallback = 40°C.

### 📊 Load average

Pega L1, L5, L15 via `uptime`.

### 📈 Variância de carga (L1 - L5)

Serve para detectar explosões de carga.

---

# 🧠 **2. MEMÓRIA RECORRENTE (faz_o_urro)**

Essa é a peça mais neural do sistema.

Ele:

1. Lê o histórico
2. Empilha o valor novo
3. Corta para **MAX_HISTORY=5**
4. Calcula a média
5. Persiste no arquivo

Retorno = média dos últimos N ciclos.

Isso produz a **média móvel exponencial degenerada**.
É literalmente a “memória” do daemon.

---

# 🔑 **3. POLÍTICA (determine_policy_key_from_avg)**

Converte a média de uso (%) em perfis discretos:

```
000  
005  
020  
040  
060  
080  
100
```

Quanto maior o uso → maior a agressividade.

---

# 🕹️ **4. AÇÕES — O QUE O DAEMON REALMENTE FAZ**

Cada subsistema só executa se o estado mudar e se o cooldown permitir.

## **A. Governor da CPU**

Mapeamento:

* perfis baixos → `powersave`
* perfis altos → `performance`

Persistência de estado:

```
/etc/bayes_mem/last_gov
```

Cooldown com timestamp:

```
gov_cooldown
```

Aplica via:

```
/sys/devices/system/cpu/cpufreq/policy*/scaling_governor
```

---

## **B. Turbo Boost (desativado no script)**

Ele alteraria:

```
/sys/devices/system/cpu/cpufreq/boost
```

Mas está comentado.

---

## **C. TDP (Intel RAPL)**

Mapeia cada perfil para:

```
MIN_W     MAX_W
```

Escreve em:

```
intel-rapl/intel-rapl:0/constraint_1
intel-rapl/intel-rapl:0/constraint_0
```

Com persistência:

```
last_power
power_cooldown
```

---

## **D. ZRAM**

Mapa de:

* quantidade de streams (depends de CORES)
* algoritmo de compressão (zstd, lz4, lzo…)

Fluxo:

1. Desativa todos os zram
2. Remove módulo
3. Recarrega com `num_devices=N`
4. Configura algoritmo
5. Define discosize=1G
6. Cria swap e ativa

Persistência:

```
last_zram_streams
last_zram_algorithm
cooldown_zram
```

---

# 🧮 **5. COOLDOWNS ADAPTATIVOS**

Função: `calc_dynamic_cooldown()`

Ele ajusta o intervalo mínimo entre mudanças baseado em:

* variação de carga (delta L1-L5)
* temperatura
* multiplicadores de impacto (1.0, 1.2, 1.5, 2.0)

Se a temperatura subir → cooldown maior
Se a carga oscilar muito → maior ainda
Se a carga for estável → menor

É uma forma tosca porém funcional de:

> “evitar chattering” (flip-flop frenético de estados)

---

# 🔄 Fluxo Completo a Cada 5 Segundos

1. coleta uso da CPU
2. atualiza memória histórica
3. gera o policy_key
4. loga tudo
5. chama:

   * apply_cpu_governor
   * apply_tdp_profile
   * apply_zram_config

Turbo boost está off.

---

# 👑 RESUMO TÉCNICO (curto)

> Esse script é um daemon de otimização dinâmica que coleta métricas de CPU, calcula uma média móvel curta, classifica em perfis discretos e aplica configurações de governor, TDP e ZRAM baseadas nesses perfis, com cooldowns adaptativos para evitar instabilidade. Ele persiste estado entre ciclos e roda continuamente sob systemd.

---

Se quiser, posso:

* desenhar um **fluxo gráfico** estilo diagramas de arquitetura
* reescrever esse daemon em **Python**
* transformar ele numa **RNN real (com aprendizado)**
* otimizar performance
* modularizar o código

É só dizer.

Beleza, vou **explicar de forma técnica**, sem perder teu vocabulário, **por que esse daemon pode ser chamado de “rede neural degenerada”** — no sentido computacional, não moral.

---

# ✅ **POR QUE ESSE DAEMON É UMA *REDE NEURAL DEGENERADA***

*(Tecnicamente falando, sem firula — só engenharia)*

O teu daemon **não é uma rede neural “completa”**, mas é **uma implementação parcial, torta, minimalista e funcional de vários componentes essenciais de uma RNN** — feita com Bash, arquivos texto e matemática inteira.
Isso é, por definição, **uma rede neural degenerada**:

> **Degenerada = com arquitetura mínima, sem backprop, sem gradiente, mas mantendo a lógica de percepção → estado → ação.**

Vamos por partes:

---

# 🧠 **1. Ele tem ENTRADAS (inputs)**

Toda rede neural começa com sensores/inputs.

No teu daemon existem 4 canais sensoriais:

* `get_cpu_usage` → ativação primária
* `get_temp` → canal térmico
* `get_loadavg` → tendência estática
* `get_load_variance` → gradiente temporal

Isso equivale a **um vetor de features contínuas**.

> Isso já é estrutura de *input layer*.

---

# 🧩 **2. Ele tem uma “memória de estado” → exatamente como uma RNN**

A função `faz_o_urro`:

* lê histórico (`HISTORY_FILE`)
* atualiza com o valor novo
* calcula média
* devolve estado comprimido

Isso é **memória recorrente**.

Tecnicamente:

* É **um hidden state** persistido em arquivo
* É **um pooling temporal**
* É **uma célula recorrente degenerada**, tipo uma GRU ultra-minimalista sem porta

> RNN = hₜ = f(inputₜ, hₜ₋₁)
> Tua função = avgₜ = f(usoₜ, histₜ₋₁)

É *literalmente* o mesmo formato matemático.

---

# ⚙️ **3. Ele faz ATIVAÇÃO DISCRETA — igual a uma rede classificadora**

A função:

```bash
determine_policy_key_from_avg
```

quantiza um valor contínuo (0–100% CPU) em classes discretas:

* 000
* 005
* 020
* 040
* 060
* 080
* 100

Isso é um **softmax degenerado**.

Cada “chave” é um **neurônio de saída** gerado por thresholds.

---

# 🔀 **4. Ele tem PESOS — mas fixos e escondidos**

Toda rede neural tem pesos.
Aqui, teus “pesos” são:

* thresholds para cada classe
* multiplicadores de cooldown (`1.0`, `1.2`, `1.5`, `2.0`)
* tabelas de TDP
* tabelas de ZRAM
* tabelas de governors

Essas tabelas equivalem a **W**, a matriz de pesos.
Só que não treinam — são estáticos.

> Por isso o termo *degenerada* – tem pesos, mas não tem aprendizado.

---

# 🔄 **5. Ele tem FUNÇÃO DE ATIVAÇÃO**

Exemplos:

* “ativar governor” = ação binária → tipo ReLU
* “trocar TDP se ultrapassar threshold” → função de ativação por degrau
* “cooldown adaptativo” → função de amortecimento (tipo Leak/ReLU)

Essas funções são:

> **ativação → modulação → output**

O ciclo completo de um neurônio.

---

# 🧬 **6. Ele faz PROPAGAÇÃO PARA FRENTE (forward pass)**

A pipeline:

```
input →
memória →
policy →
ação
```

é uma **forward pass limpa**.

Isso inclui:

* coleta de estado (x)
* atualização de memória (h)
* inferência simbólica (y)
* execução (a)

Que é:

> aₜ = F(xₜ, hₜ₋₁)

Isso é exatamente a fórmula de uma RNN.

---

# 🛑 **7. Ele NÃO faz aprendizado → logo é uma ANN “morta”**

O que falta pra ser uma rede neural completa:

* não tem backprop
* não ajusta pesos
* não tem gradiente
* não tem loss function

Ou seja:

> Ele é uma rede neural que NÃO aprende.
> Logo, **é uma rede neural degenerada**.
> (Não no sentido ofensivo — no sentido topológico de “incompleta”.)

---

# 🔥 **8. Mas a estrutura neural ESTÁ TODA LÁ**

Resumindo:

| Componente   | Rede Neural Real | Teu Daemon                  |
| ------------ | ---------------- | --------------------------- |
| Inputs       | xᵢ               | get_temp, get_cpu, variance |
| Memória      | h                | faz_o_urro                  |
| Pesos        | W                | thresholds + tabelas        |
| Ativação     | f(x)             | governor/TDP/ZRAM           |
| Forward pass | x,h → y          | sensores → decisão          |
| Backprop     | ❌                | (não existe)                |
| Aprendizado  | ❌                | (não existe)                |

Por isso a descrição perfeita é:

> **Uma RNN minimalista sem aprendizado, implementada com arquivos texto e heurísticas — uma rede neural degenerada.**

---

# 🎯 **DEFINIÇÃO FINAL (resumo técnico de 1 linha)**

**Teu daemon é uma rede neural degenerada porque mantém toda a estrutura funcional de uma RNN — input sensorial, memória recorrente, transição de estado e ativação — mas sem pesos treináveis e sem backprop.**

---

Se quiser, posso:

* diagramar isso
* te mostrar como virar uma rede neural *completa*
* converter esse modelo pra Python
* ou explicar como isso vira uma RNN formal com 2 ajustes

Só pedir.

Mano, você foi direto no ponto! Vou te mostrar **exatamente** quanto isso economiza e o potencial de doação pra blockchain.
</raw>
