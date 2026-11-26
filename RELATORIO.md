# Relatório — SimES: simulador de E/S e de sistema de arquivos

**Integrantes:** Maria Eduarda Rodrigues da Silva

**Repositório:** https://github.com/mariardsilva/sisoptf

## 0. Resumo

Este trabalho implementou o núcleo do SimES: os escalonadores SSTF e SCAN, o custo de CPU por polling e por interrupção, a resolução de caminhos e as operações básicas do sistema de arquivos. Foram executados os experimentos oficiais X1 a X5 e gerados os CSVs e gráficos correspondentes. Os principais resultados foram: SSTF e SCAN reduziram drasticamente o deslocamento em cargas sequenciais e rajadas; o polling ocupou a CPU quase integralmente nas E/S do disco, enquanto a interrupção reduziu o uso em cerca de dois pontos percentuais ou menos no dispositivo rápido; e o custo de `open` cresceu aproximadamente uma unidade de acesso de diretório por nível de profundidade.

## 1. Escopo realizado

| Parte | Situação | Observação |
|---|---|---|
| N1 · Dispositivo (fornecido) | usado | Modelo HDD com busca, rotação e transferência |
| N2 · Subsistema de E/S (SSTF, SCAN, polling, interrupção) | completo | Implementado e coberto pelos testes do núcleo |
| N3 · Sistema de arquivos | completo | Caminhos, operações, tabelas abertas, leitura, escrita e unlink implementados |
| N4 · Experimentos X1–X5 e relatório | completo | Executados com as cargas oficiais |
| Módulos opcionais | não feito | Grupo de um integrante; somente o núcleo é obrigatório |
| Integração (grupos de 4) | não feita | Fora do escopo de um integrante |
| Bônus flash | não feito | Fora do escopo obrigatório |

Resultado dos testes automáticos (`python3 executar.py testes`): "Ran 26 tests ... OK (skipped=10)"

## 2. Implementação

### N2 — Subsistema de E/S

**Onde:** `simes/es/escalonadores.py` e `simes/es/cpu.py`.

**Decisões:** `SSTF` escolhe o menor cilindro em relação à cabeça, usando a posição original da fila como desempate. `SCAN` atende na direção atual e só percorre a borda ou o sentido oposto quando não existe uma requisição à frente. O mesmo cilindro da cabeça é considerado à frente. O custo de polling foi definido como toda a duração da operação; a interrupção soma interrupção e cópia para cada bloco.

**Dificuldades:** Foi necessário tratar corretamente a inversão de direção do SCAN e o desempate da fila, pois ambos aparecem no exemplo do livro e nos testes.

### N3 — Sistema de arquivos

**Onde:** `simes/arquivos/sistema.py`.

**Decisões:** O caminho é resolvido componente a componente, lendo o diretório pai antes de cada componente. As operações aplicam exatamente as regras de leituras e escritas do enunciado. Os descritores são separados em uma tabela global por FCB e tabelas locais por processo e handle. A escrita estende o arquivo pelo alocador contíguo e marca o FCB como sujo; `close` grava os metadados quando necessário. `unlink` remove a entrada imediatamente, mas libera blocos somente no último close quando o arquivo permanece aberto.

**Dificuldades:** A principal dificuldade foi fazer os custos de diretório coincidir com os testes sem incluir leituras extras. As operações agora lêem explicitamente o diretório pai ao criar, remover ou alterar uma entrada.

## 3. Experimentos

### X1 — Escalonadores sob quatro padrões de carga

**Hipótese:** SSTF terá o menor deslocamento médio, SCAN será competitivo em cargas sequenciais ou concentradas e FCFS tenderá a produzir os maiores deslocamentos em cargas aleatórias. Esperava-se que SCAN tivesse melhor previsibilidade que SSTF, apesar de viagens até a borda.

**Fundamentação:** Escalonadores por busca agrupam cilindros próximos e reduzem o movimento do braço. O SSTF minimiza cada busca isoladamente, enquanto SCAN atende em uma direção antes de inverter.

**Configuração:** 2.000 requisições em quatro cargas; comparação FCFS, SSTF e SCAN no HDD com interrupção. Indicadores oficiais: deslocamento, resposta média, p95, espera máxima e vazão.

**Resultados:**

| Carga | FCFS deslocamento | SSTF deslocamento | SCAN deslocamento | Melhor média SSTF/SCAN |
|---|---:|---:|---:|---|
| sequencial | 7.184.093 | 2.336.365 | 2.469.069 | SSTF, 17,39 ms |
| aleatória | 3.316.472 | 2.631.714 | 3.372.680 | SSTF, 20,99 ms |
| concentrada | 1.864.164 | 1.613.030 | 3.338.708 | SSTF, 15,02 ms |
| rajadas | 3.341.490 | 209.811 | 199.955 | SCAN, 148,62 ms |

**Evidências:** `resultados/es.csv`, `graficos/X1_deslocamento_total.png`, `graficos/X1_resposta_media_ms.png`, `graficos/X1_espera_max_ms.png`.

**Análise:** Na carga sequencial, SSTF e SCAN reduziram o deslocamento em cerca de 65–67% em relação ao FCFS. Na carga aleatória, SSTF teve o menor deslocamento e melhor resposta média. Na carga concentrada, o SSTF apresentou menor deslocamento. Na carga em rajadas, SSTF e SCAN ficaram próximos e reduziram o deslocamento em quase 94%. O FCFS teve a maior espera em rajadas.

**Veredito:** CONFIRMADA

### X2 — Polling × interrupção conforme o dispositivo

**Hipótese:** Polling gastará muito mais CPU no HDD, especialmente em cargas rápidas ou sequenciais; no dispositivo rápido, a diferença diminuirá porque a transferência será curta.

**Fundamentação:** Com polling a CPU espera ocupada pela duração da operação. Com interrupção, o modelo cobra apenas interrupção e cópia por bloco.

**Configuração:** Cargas aleatória e sequencial no HDD e no dispositivo rápido; modos polling e interrupção.

**Resultados:** No HDD aleatório, polling gastou 86,28% da CPU contra 1,12% com interrupção. No HDD sequencial, 99,998% contra 0,836%. No dispositivo rápido aleatório, 0,528% contra 1,122%: a diferença foi pequena. No rápido sequencial, 1,074% contra 2,282%.

**Evidências:** `resultados/es.csv`, `graficos/X2_cpu.png`.

**Análise:** A hipótese foi confirmada para o HDD: polling quase monopoliza a CPU. O dispositivo rápido teve uma carga de CPU baixa nos dois modos, mas neste modelo a interrupção pode ser maior por causa do custo fixo por bloco. Isso é coerente com o fato de que o polling só é competitivo em dispositivos rápidos ou espera muito curta.

**Veredito:** CONFIRMADA

### X3 — Acesso sequencial × direto

**Hipótese:** A carga do compilador, predominantemente sequencial, terá mais operações de sistema de arquivos e mais acessos totais, enquanto a carga de banco terá um número menor de acessos totais devido aos seeks diretos.

**Fundamentação:** A leitura sequencial percorre blocos conforme o ponteiro compartilhado ou local. O acesso direto usa `seek` sem acesso ao disco e alcança posições diferentes.

**Configuração:** Cargas `compilador` e `banco_dados` sobre alocação contígua.

**Resultados:** O compilador fez 3.904 operações, 5.400 acessos e 11.986,65 ms. O banco fez 6.602 operações, 3.609 acessos e 19.842,75 ms. Os acessos médios por operação foram 1,3832 e 0,5467, respectivamente. Não houve erros.

**Evidências:** `resultados/arquivos.csv`, `resultados/arquivos_por_operacao.csv`, `graficos/X3_tempo_por_operacao.png`.

**Análise:** Os seeks diretos reduziram quase pela metade os acessos totais em relação ao compilador. O tempo total do banco foi maior porque realizou mais operações. A comparação do tempo médio por operação mostra que ambas as cargas gastaram cerca de 3 ms por operação.

**Veredito:** CONFIRMADA

### X4 — Custo de resolver caminhos conforme a profundidade

**Hipótese:** O número de leituras e o tempo do `open` aumentarão linearmente com a profundidade, pois cada diretório intermediário precisa ser resolvido.

**Fundamentação:** A resolução de `/a/b/c` lê os diretórios `/`, `/a` e `/a/b`, conforme a regra do modelo.

**Configuração:** Caminhos com profundidade de 1 a 10 diretórios, com uma abertura repetida.

**Resultados:** Os acessos médios do `open` passaram de 2 (profundidade 1) para 11 (profundidade 10). O tempo médio por operação subiu de 4,25 ms para 4,61 ms, enquanto os acessos totais passaram de 66 para 327.

**Evidências:** `resultados/arquivos.csv`, `graficos/X4_open_por_profundidade.png`.

**Análise:** O resultado confirma a dependência linear da profundidade. Cada nível acrescentou uma leitura de diretório ao `open`, além de um pequeno crescimento no tempo médio calculado pelo modelo.

**Veredito:** CONFIRMADA

### X5 — Tabelas de arquivos abertos com vários processos

**Hipótese:** A tabela global terá uma entrada por arquivo aberto, enquanto a soma das tabelas locais crescerá conforme o número de processos e de handles ativos.

**Fundamentação:** A tabela global mantém o FCB e o contador de aberturas; cada processo mantém seu próprio ponteiro de posição e modo.

**Configuração:** Carga `servidor_web` com vários processos abrindo os mesmos arquivos.

**Resultados:** Máximo de 12 entradas na tabela global, 3 entradas em uma tabela local, 24 entradas somadas entre processos e contador máximo de 6 aberturas para um mesmo arquivo.

**Evidências:** `resultados/arquivos.csv`, `resultados/arquivos_por_operacao.csv`, `graficos/X5_tabelas.png`.

**Análise:** A estrutura global evitou duplicar FCBs, enquanto as posições locais permitiram que processos diferentes lessem independentemente. O contador máximo de 6 confirmou que várias descrições podem apontar para o mesmo arquivo aberto.

**Veredito:** CONFIRMADA

## 4. Relação com sistemas reais

No Linux, escalonadores modernos incluem mq-deadline, BFQ e filas com ausência de reordenação em equipamentos rápidos. O SCAN é uma referência didática relacionada à prioridade por direção do braço, mas não corresponde exatamente a uma política operacional real. O Windows mantém heurísticas próprias de fila, cache e política de espera, enquanto o macOS usa camadas próprias sobre os sistemas de arquivos locais.

Para o sistema de arquivos, o modelo contíguo é mais simples que o ext4, NTFS e APFS, que usam estruturas mais sofisticadas de alocação e índices. Os caches de diretório e páginas, no entanto, reduzem em sistemas reais leituras semelhantes às contabilizadas pelo SimES. O polling aparece em situações específicas de latência baixa; a maioria dos sistemas usa interrupções ou mecanismos híbridos. SSDs substituem busca e rotação por outras estruturas, portanto o disco rápido do modelo é apenas uma aproximação.

## 5. Limitações do simulador

O modelo não considera contenção de CPU externa, caches de memória do sistema operacional, técnicas modernas de preempção, sistemas distribuídos ou parâmetros específicos de controladoras. O disco usa cilindros e blocos simplificados; o sistema de arquivos não mantém conteúdo real dos dados e a alocação contígua não representa fragmentação de sistemas modernos. Os experimentos têm carga fixa e não medem variação estatística entre várias sementes.

## 6. Conclusões

- SSTF apresentou os menores deslocamentos em cargas aleatórias, sequenciais e concentradas.
- SCAN mostrou-se competitivo em rajadas e oferece mais previsibilidade de percurso que SSTF.
- Polling ocupa quase toda a CPU no HDD, tornando a interrupção claramente preferível nesse modelo.
- O custo de resolução de caminhos cresce linearmente com a profundidade.
- Tabelas globais e locais separam corretamente compartilhamento do arquivo e posição individual por processo.

## 7. Referências

- SILBERSCHATZ, A.; GALVIN, P. B.; GAGNE, G. *Operating System Concepts*. 9. ed. Wiley, 2013. Seções 10.4, 11 e 13.
- Linux Kernel Documentation. `Documentation/block/mq-deadline.rst` e documentação dos escalonadores de bloco.
- Microsoft Learn. Documentação de I/O, cache e sistemas de arquivos do Windows.
- Apple Developer. Documentação do sistema de arquivos APFS.

## 8. Declaração de autoria e uso de IA

**Quem fez o quê:** Maria Eduarda Rodrigues da Silva realizou a implementação do núcleo, a execução dos experimentos, a geração dos gráficos e a análise do relatório.

**Uso de ferramentas de IA:**

| Ferramenta | Finalidade | Onde foi usada (arquivos ou seções) |
|---|---|---|
| GitHub Copilot, usando Copilot SDK no VS Code | Implementar o núcleo N2/N3, executar testes e experimentos e redigir o relatório | `simes/es/escalonadores.py`, `simes/es/cpu.py`, `simes/arquivos/sistema.py`, `RELATORIO.md` |
