# Daemon Bayesiano de Otimização — Documentação Técnica

Este documento é um README avançado / whitepaper técnico do daemon de otimização adaptativa (protótipo). Está organizado para ser acessível a engenheiros e administradores de sistema que queiram entender, operar ou estender o projeto.

## Sumário

- Overview técnico
- Arquitetura do daemon
- Fluxo de execução
- Sistema de coleta de métricas
- Memória interna (média móvel / RNN degenerada)
- Sistema de políticas (perfis 000–100)
- Governança de energia (CPU governor, TDP, turbo)
- Substituição dinâmica de módulos (ZRAM)
- Cooldowns adaptativos
- Por que o sistema funciona como uma “rede neural degenerada”
- Tabelas, diagramas textuais e explicações
- Resumo funcional
- Observações de implementação
- Possíveis melhorias futuras

---

## Overview técnico

O daemon é um processo contínuo que monitora métricas do sistema (uso de CPU, temperatura, load average), mantém uma memória simples (média dos últimos N ciclos), quantiza a média em perfis discretos (000–100) e aplica políticas de controle sobre:

- Governor da CPU (scaling_governor)
- Limites TDP via Intel RAPL
- Configuração dinâmica de ZRAM (número de streams + algoritmo)

O ciclo ocorre a cada ~5 segundos e usa arquivos em `/etc/bayes_mem` para persistência de estado. Cooldowns adaptativos são usados para evitar chattering (trocas rápidas).

### Contrato técnico (inputs / outputs / sucesso / erro)

- Inputs: leituras de `/proc/stat`, saída de `sensors`, `uptime`.
- Outputs: alterações em `/sys/devices/system/cpu/.../scaling_governor`, arquivos em `/sys/class/powercap/...` (Intel RAPL), reconfiguração do módulo zram, logs em `/var/log/bayes_mem/bayes.log`.
- Erros/limitações: permissões root exigidas para várias operações, ausência de interfaces (Intel RAPL, zram), `sensors` não instalado. O daemon possui fallbacks básicos (ex.: temperatura default = 40°C).

---

## Arquitetura do daemon

Principais componentes:

- `main()` — inicialização e loop principal
- Coleta de métricas — `get_cpu_usage`, `get_temp`, `get_loadavg`, `get_load_variance`
- Memória histórica — `faz_o_urro` (arquivo + média móvel)
- Política — `determine_policy_key_from_avg`
- Atuadores — `apply_cpu_governor`, `apply_tdp_profile`, `apply_zram_config`, ( `apply_turbo_boost` está presente mas desativado )
- Cooldown / utilitários — `calc_dynamic_cooldown`, `calc_impact_cooldown`

Persistência em disco (exemplos):

```
/etc/bayes_mem/
    cpu_history
    last_gov
    last_power
    last_zram_streams
    gov_cooldown
    power_cooldown
    cooldown_zram
    last_stat
/var/log/bayes_mem/bayes.log
```

Arquitetura em camadas (texto): Sensores → Processamento de sinais → Selector (quantização) → Controladores → Persistência/Logging

---

## Fluxo de execução (loop principal)

1. `initialize_directories()` — garante diretórios e arquivos
2. Loop infinito:
   - `apply_all()` → coleta, inferência e ações
   - `sleep 5`

`apply_all()` efetua:

- `current_usage = get_cpu_usage()`
- `avg_usage = faz_o_urro(current_usage)`
- `policy_key = determine_policy_key_from_avg(avg_usage)`
- `apply_cpu_governor(policy_key)`
- `apply_tdp_profile(policy_key)`
- `apply_zram_config(policy_key)`

---

## Sistema de coleta de métricas

- CPU usage: leitura de `/proc/stat`, comparação com última leitura (`last_stat`) para calcular porcentagem de uso real.
- Temperatura: usa `sensors` (lm-sensors). Pega o primeiro valor em °C e corta decimais. Fallback = 40°C.
- Load average: leitura via `uptime` para obter L1, L5, L15.
- Variação de carga: `|L1 - L5|` — sinal de picos/gradiente.

Trecho conceitual:

```bash
cpu_line=$(grep -E '^cpu ' /proc/stat)
# prev read from last_stat or default
# compute diffs -> usage%
```

---

## Memória interna (média móvel / RNN degenerada)

`faz_o_urro(new_val)` implementa:

- leitura do histórico (`HISTORY_FILE`)
- append do `new_val`
- truncar para `MAX_HISTORY` (default 5)
- calcular média aritmética e persistir sequência
- retornar média

Comportamento: funciona como um hidden state de curto prazo — equivalente a uma célula recorrente mínima. Robustez: cria arquivos se ausentes; evita divisão por zero.

---

## Sistema de políticas (perfis 000–100)

Mapeamento por thresholds:

- >= 90 → `100`
- >= 80 → `080`
- >= 60 → `060`
- >= 40 → `040`
- >= 20 → `020`
- >= 5  → `005`
- else   → `000`

As chaves alimentam os módulos de atuação (governor, TDP, ZRAM).

---

## Governança de energia (CPU governor, TDP, turbo)

- Governor: mapeamento chaves→governor (`powersave` / `performance`). Aplica-se em `/sys/devices/system/cpu/cpufreq/policy*/scaling_governor`.
- Turbo Boost: módulo presente (`/sys/devices/system/cpu/cpufreq/boost`) mas controlado por `apply_turbo_boost` (desativado por padrão no deploy atual).
- TDP: mapeia perfis para pares (MIN/MAX) e escreve em Intel RAPL (`constraint_*_power_limit_uw`). Uso de arquivos `last_power` e `power_cooldown` para persistência e cooldown.

Exemplo simplificado:

```bash
# aplicar governor
for policy in /sys/devices/system/cpu/cpufreq/policy*; do
  echo "$cpu_gov" | sudo tee $policy/scaling_governor
done
```

---

## Substituição dinâmica de módulos (ZRAM)

Objetivo: ajustar compressão e número de dispositivos zram conforme carga.

Fluxo:

1. swapoff /dev/zram*
2. modprobe -r zram
3. modprobe zram num_devices="$streams"
4. para cada /dev/zram*: set comp_algorithm, disksize, mkswap, swapon
5. persistir `last_zram_streams` e `last_zram_algorithm`

Observação: operação intrusiva — cooldowns maiores (fator 2.0). Requer privilégios.

---

## Cooldowns adaptativos

- `calc_dynamic_cooldown()` calcula um cooldown base (ex.: 7s) e o ajusta por temperatura e variação de carga.
- `calc_impact_cooldown(factor)` aplica multiplicadores (1.2 turbo / 1.5 TDP / 2.0 ZRAM).
- Cada subsistema tem arquivo de cooldown (`gov_cooldown`, `power_cooldown`, `cooldown_zram`).

Intenção: evitar chattering, aumentar conservadorismo em condições térmicas ou de alta variação.

---

## Por que o sistema funciona como uma "rede neural degenerada"

- Inputs: vetor de features (usage, temp, load, delta)
- Memória: `faz_o_urro` → hidden state persistido
- Pesos: thresholds, tabelas, multiplicadores (fixos)
- Ativação: quantização em perfis e ações (governor/TDP/ZRAM)
- Forward pass: sensores → memória → política → ação

Falta: backprop, ajuste de weights, loss. Portanto: estrutura RNN presente; aprendizado ausente.

---

## Tabelas, diagramas textuais e explicações

Diagrama ASCII (alto nível):

```
+--------------------+
| Sensores / Inputs  |
+---------+----------+
          |
          v
+--------------------+
| Processamento      |
+---------+----------+
          |
          v
+--------------------+
| Memória (faz_o_urro)| --> +-----------------+
+---------+----------+      | Policy selector |
          |                   +-----------------+
          v                           |
    +-----------------------------+   v
    | Actuators / Controllers     |<--+
    +-----------------------------+
```

Tabela exemplo de mapeamento (resumo):

| Perfil | Governor   | TDP (MIN/MAX) | ZRAM (streams / alg) |
|--------|------------|---------------|----------------------|
| 000    | powersave  | 3W / 0W       | 0 / none             |
| 005    | powersave  | 30% / 0%      | CORES*15% / zstd     |
| 020    | powersave  | 50% / 10%     | CORES*30% / lz4hc    |
| 040    | powersave  | 70% / 20%     | CORES*45% / lz4      |
| 060    | performance| 80% / 30%     | CORES*60% / lzo      |
| 080    | performance| 90% / 40%     | CORES*50% / lzo      |
| 100    | performance| MAX / 50%     | CORES / lzo-rle      |

---

## Resumo funcional

- Monitora, decide e atua continuamente. Persiste estado. Usa cooldowns adaptativos. Atua sobre governor, TDP e ZRAM. Tem memória curta e heurística.

---

## Observações de implementação

- Requer privilégios para manipular `/sys`, `modprobe`, `swapon`.
- Intel RAPL, zram e `sensors` podem não estar disponíveis em todas as plataformas — o daemon deve checar presença de interfaces e ter fallbacks.
- Reconfigurações de ZRAM são intrusivas; considerar janelas de manutenção.
- Logging: usar logrotate e níveis de log. Atualmente usa `/var/log/bayes_mem/bayes.log`.
- Melhorar tratamento de erros nas escritas em /sys (verificação e rollback quando possível).

---

## Possíveis melhorias futuras

1. Reescrever em Python — melhor tratamento de erros, testes e bibliotecas de métricas.
2. Transformar em RNN real / RL para aprendizado on-line: coletar dados e treinar política.
3. Modularizar em `sensor`, `memory`, `policy`, `actuator` com testes unitários.
4. Adicionar exportador Prometheus para observability.
5. Substituir arquivos planos por sqlite ou state store atômico.
6. Graceful shutdown e signal handling.
7. Simulador de cargas para validar políticas e cooldowns.

---

## Bloco raw (conteúdo original completo)

<raw>
Beleza. Vou te explicar **como funciona esse daemon**, mas **de forma técnica, concisa e organizada**, sem perder o clima “engenheiro underground escrevendo um Frankenstein funcional”.

---

# 🧠 **VISÃO GERAL — O QUE ESSE BICHO É**

Esse script é um **daemon de otimização adaptativa** que:

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

Ele lê `/proc/stat` duas vezes, compara os deltas e calcula:

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
