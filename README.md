# Laboratório de Circuitos – Projeto Integrador (AOC 2026)

Universidade Federal de Roraima (UFRR) · Curso de Ciência da Computação

Disciplina: DCC301 – Arquitetura e Organização de Computadores · Semestre: 2026.2 · Prof. Herbert Oliveira Rocha

| Integrante | Matrícula |
|---|---|
| Rafael da Silva | 2021022827 |
| Jaqueline Braga Meneses | 2023010478 |

Simulador: **Logisim-Evolution 5.0.0**. Para abrir um circuito, use *Arquivo → Abrir* e escolha o `.circ`; o circuito principal de cada arquivo já abre como "main".

## Estrutura

```
README.md
relatorio/
  relatorio_tecnico.docx        (fonte do relatório)
  relatorio_tecnico.pdf         (versão de entrega)
parte1/
  parte1_memoria.circ           main: MEMORIA_PARTE1 (também MEMORIA_PARTE1_FIOS, versão só com fios)
  parte1_cache.circ             main: CACHE_PARTE1 (toda com fios)
  parte1_planilha.xlsx          traço de 16 acessos, indicadores e teste de conflito
  vetores/                      vetores de teste do Logisim
  evidencias/                   prints dos testes T-01 a T-05
parte2/
  parte2_cabeada.circ           main: CPU_CABEADA (UC cabeada + caminho de dados)
  parte2_microprogramada.circ   main: CPU_MICRO (UC microprogramada + caminho de dados)
  microcodigo.txt               conteúdo da ROM de microcódigo (04 22 90 99 5d)
  tabela_tempo.xlsx             tabela de tempo dos 5 estados
  vetores/                      vetores de teste (CPU, T-09 e componentes)
  evidencias/                   prints dos testes da Parte II
componentes/                    componentes individuais (um .circ por componente)
```

## Parte I – Subsistema de memória

Pinos de MEMORIA_PARTE1: `A5`, `A4` (seleção de dispositivo), `END[4]` (endereço interno A3..A0), `ENTRADA[8]` (dado a gravar), `ESCREVER`, `RELOGIO`, `ZERAR` (entradas); `SAIDA[8]` e `ERRO` (saídas).

Mapa: 00 = ROM, 01 = RAM, 10 = banco de registradores, 11 = E/S. Na RAM, a paridade ímpar é gravada junto com o dado; na leitura, ela é recalculada e comparada, e uma divergência acende `ERRO`.

A cache (CACHE_PARTE1) é de mapeamento direto, com 4 linhas e blocos de 2 bytes. O endereço de 6 bits se divide em rótulo A5..A3, índice A2..A1 e deslocamento A0. Ela tem contador de acertos com display de 7 segmentos e o modo "só primos" (PRIMO4).

### Testes realizados

| Teste | Circuito | Resultado |
|---|---|---|
| T-01: decodificação e leitura/escrita | MEMORIA_PARTE1 | OK (prints) |
| T-02: paridade/erro | MEMORIA_PARTE1 | OK (prints) |
| Vetor `teste_MEMORIA_PARTE1.txt` | MEMORIA_PARTE1 e MEMORIA_PARTE1_FIOS | 16/16 |
| Vetores do banco de registradores, da RAM e da ROM | BANCO_REG, MEMORIA_RAM, MEMORIA_ROM | 21/21, 14/14, 4/4 |
| T-03, T-04, T-05 (vetor `teste_CACHE_T03_T05.txt`) | CACHE_PARTE1 | OK, 9/9 |
| Traço de 16 acessos (`teste_CACHE_traco.txt`) | CACHE_PARTE1 | 50/50 |
| Modo primos (`teste_CACHE_primos.txt`) | CACHE_PARTE1 | 26/26 |

Os vetores podem ser executados pela linha de comando:

```
java -jar logisim-evolution-5.0.0-all.jar -w CACHE_PARTE1 parte1/vetores/teste_CACHE_traco.txt parte1/parte1_cache.circ
```

Também é possível usar *Simular → Vetor de teste* dentro do Logisim.

## Parte II – Unidade de controle e caminho de dados

Os dois arquivos têm o mesmo caminho de dados (`CAMINHO_DADOS`). A diferença é a unidade de controle:

- `parte2_cabeada.circ` usa `CPU_CABEADA` = `UC_CABEADA` (3 FF D + AND/OR/NOT, sem ROM) + `CAMINHO_DADOS`.
- `parte2_microprogramada.circ` usa `CPU_MICRO` = `UC_MICRO` (µPC `CONTADOR_UPC` + ROM de microcódigo `04 22 90 99 5D`) + `CAMINHO_DADOS`.

Os estados não usados (101, 110, 111) foram tratados como indiferentes e voltam a um estado válido em 1 ciclo.

**Formato da instrução de 8 bits.** O enunciado não define o formato, então adotamos este (estilo MIPS, com o imediato sobreposto):

| Bits | 7–6 | 5–4 | 3–2 | 1–0 |
|---|---|---|---|---|
| Tipo R | opcode = 00 | rs (= rd) | rt | funct (00 add, 01 sub, 10 and, 11 or) |
| beq | opcode = 01 | rs | rt | imm = IR[3:0], 4 bits com sinal |

Como 8 bits não comportam rs, rt, rd e funct ao mesmo tempo, o destino é o próprio rs (rd = rs). O imediato do beq ocupa IR[3:0], ou seja, os seus 2 bits altos coincidem com o campo rt. O PC é endereçado em bytes (PC ← PC + 4), e a ROM de programa (16 × 8) usa PC[5:2] como endereço. O sinal B da UC vem do opcode: B = IR7'·IR6.

**Modo CARGA.** Não existe instrução de carga imediata, então os registradores são inicializados por pinos. Com R = 1 e CARGA = 1, escolha o registrador em CREG1 CREG0, coloque o valor em CDADO e dê um pulso em CLK. No modo CARGA, a saída RD1 mostra o registrador escolhido. Para executar, volte com CARGA = 0 e R = 0.

**Programa de teste na ROM** (R1 = 5, R2 = 3, R0 = R3 = 0):

| End. | Hex | Instrução | Efeito |
|---|---|---|---|
| 0x00 | 18 | add R1,R2 | R1 = 8 (T-07) |
| 0x04 | 71 | beq R3,R0,+1 | desvio tomado: PC vai para 0x0C (T-08) |
| 0x08 | 14 | add R1,R1 | pulada |
| 0x0C | 51 | beq R1,R0,+1 | desvio não tomado: PC segue para 0x10 (T-08) |
| 0x10 | 19 | sub R1,R2 | R1 = 5 |
| 0x14 | 1A | and R1,R2 | R1 = 1 |
| 0x18 | 1B | or R1,R2 | R1 = 3 |
| 0x1C | 4F | beq R0,R3,−1 | laço de parada |

**Testes por vetor** (Logisim-Evolution 5.0.0, pasta `parte2/vetores/`):

| Vetor | Circuito | Resultado |
|---|---|---|
| teste_CPU_T06_T07_T08.txt (28 ciclos) | CPU_CABEADA e CPU_MICRO | 68/68 em cada |
| teste_T09_UC.txt | UC_CABEADA e UC_MICRO | 16/16 em cada (T-09) |
| teste_ULA8.txt | ULA8 | 80/80 |
| teste_CONTROLE_ULA.txt | CONTROLE_ULA | 16/16 |
| teste_DETECTOR_101.txt | DETECTOR_101 | 31/31 |
| teste_CONTADOR_UPC.txt | CONTADOR_UPC | 35/35 |
| teste_SOMA_MAIS_4.txt | SOMA_MAIS_4 | 9/9 |
| teste_EXT_SINAL_4_8.txt | EXT_SINAL_4_8 | 16/16 |

Os vetores sequenciais usam as colunas `<set> <seq>`. O reset é feito com R = 1 e um pulso de CLK.

## Onde está cada componente

| Nº | Componente | Arquivo e subcircuito |
|---|---|---|
| 01 | Flip-flop D e JK | FF_D: bits de validade (parte1_cache), estado da UC_CABEADA, CONTADOR_UPC, DETECTOR_101. O JK está em componentes/01-FlipFlop-JK.circ. |
| 02 | MUX de 4 entradas | MUX4_8: saída da memória (parte1_memoria), cache, MUX ALUSrcB/ALUSrcA/PCSource (CAMINHO_DADOS), dentro da ULA8 |
| 03 | XOR com AND/OR/NOT | xor2: comparador de rótulos (parte1_cache) e paridade (parte1_memoria) |
| 04 | Somador de 8 bits com constante 4 | SOMA_MAIS_4 em CAMINHO_DADOS (PC4 = PC + 4) |
| 05 | ROM de 8 bits | MEMORIA_ROM (parte1_memoria), ROM de programa (CAMINHO_DADOS), ROM de microcódigo (UC_MICRO) |
| 06 | RAM de 8 bits | MEMORIA_RAM (parte1_memoria) |
| 07 | Banco de registradores | BANCO_REG: região 0x20–0x2F (parte1_memoria) e leitura de rs/rt (CAMINHO_DADOS) |
| 08 | Somador de 8 bits | SOMADOR8: endereço da cache (parte1_cache) e soma/subtração na ULA8 |
| 09 | Detector de "101" | DETECTOR_101 / TESTE_DETECTOR_101 (parte2_cabeada) |
| 10 | ULA de 8 bits | ULA8 + CONTROLE_ULA (CAMINHO_DADOS) |
| 11 | Extensor de sinal 4→8 | EXT_SINAL_4_8: imediato do beq (CAMINHO_DADOS) |
| 12 | Máquina de estados com portas | UC_CABEADA (parte2_cabeada) |
| 13 | Contador síncrono | CONTADOR_UPC: µPC da UC_MICRO (parte2_microprogramada); CONTADOR_3/CONTADOR_4 na cache |
| 14 | Paridade ímpar | PARIDADE_IMPAR (parte1_memoria) |
| 15 | Mapas de Karnaugh | equações de N0, N1, N2 e dos sinais da UC_CABEADA (relatório) |
| 16 | Decodificador de 7 segmentos | DEC_7SEG_S: acertos da cache, estado das UCs e das CPUs |
| 17 | Detector de primo | Primo4 (parte1_cache) |

A pasta `componentes/` guarda também as versões individuais de cada componente.

## Divisão do trabalho

| Integrante | Responsabilidade |
|---|---|
| Jaqueline Braga Meneses | Parte I – subsistema de memória e cache (`parte1/`) |
| Rafael da Silva | Parte II – unidade de controle cabeada e microprogramada (`parte2/`) |

## Declaração de uso de Inteligência Artificial

Usamos o assistente Claude (Anthropic) como apoio. Ele ajudou na revisão e organização dos circuitos no Logisim (por exemplo, a conversão de túneis para fios e a correção de ligações), na geração e execução de vetores de teste, no preenchimento da planilha e na redação e organização do relatório e deste README. Todos os circuitos e resultados foram conferidos e testados pela dupla no Logisim-Evolution, e cada integrante é capaz de explicar o funcionamento dos circuitos na arguição.
