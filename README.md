# RV32I Pipelined — Extensão Completa do ISA

Processador RISC-V de 32 bits (`RV32I`) com pipeline de 5 estágios em SystemVerilog, direcionado à placa **DE2-115** (Intel Cyclone IV E — `EP4CE115F29C7`).

Este projeto parte do **projeto base** (8 instruções iniciais e infraestrutura de pipeline, hazards e periféricos) e concentra-se na **implementação completa das instruções das Etapas 01, 02 e 03**, detalhando abaixo apenas as instruções adicionadas e as modificações necessárias na microarquitetura.

---

## Base do Projeto (Herdada)

O conjunto e a infraestrutura a seguir já vêm prontos do projeto base e foram mantidos como fundação:

- **Instruções base (8):** `ADD`, `SUB`, `OR`, `AND`, `SLT` (R-type), `LW` (I-type load), `SW` (S-type) e `BEQ` (B-type).
- **Pipeline e hazards base:** estágios `IF → ID → EX → MEM → WB`, forwarding `EX/MEM → EX` e `MEM/WB → EX` (`src/pl_forward.sv`), detecção de load-use stall (`src/pl_hazard.sv`), resolução de desvio no estágio `EX` com flush de 2 ciclos, e banco de registradores com escrita em `negedge clk` (`src/pl_regfile.sv`).
- **Infraestrutura de FPGA e ferramentas:** PLL 50 MHz → 10 MHz (`src/pl_top.sv`), controlador MMIO (`SW`, `KEY`, `LEDR`, `LEDG`, UART RS-232 9600 8N1 e contador de ciclos em `src/pl_mmio.sv`), montador (`assembler/assembler.py`) e captura via dump serial (`dump/serial_dump.py`).

---

## Visão Geral de Cobertura (RV32I)

| Categoria | Total ISA | Base | Implementadas nas Etapas | Total Atual |
|---|:---:|:---:|:---:|:---:|
| R-type | 10 | 5 | +5 (Etapa 01) | **10 / 10** |
| I-type aritmético/lógico/shift | 9 | 0 | +8 (Etapa 02) + `SLTIU`\* | **9 / 9** |
| I-type load | 5 | 1 (`LW`) | +4 (Etapa 03) | **5 / 5** |
| S-type | 3 | 1 (`SW`) | +2 (Etapa 03) | **3 / 3** |
| B-type | 6 | 1 (`BEQ`) | +5 (Etapa 03) | **6 / 6** |
| U-type | 2 | 0 | +2 (Etapa 02) | **2 / 2** |
| J-type / Jump (`JAL`, `JALR`) | 2 | 0 | +2 (Etapa 03) | **2 / 2** |
| **Total** | **37** | **8** | **+28 (+1 extra)** | **37 / 37** |

> \* Além das 8 instruções listadas na Etapa 02, `SLTIU` (`funct3 = 3'b011`) também está decodificada em `src/pl_alu_ctrl.sv`, completando as 37 instruções do subconjunto listado.

---

## Instruções Implementadas por Etapa

### Etapa 01 — 27/10/2026: R-type

#### Aritmética, lógica e deslocamentos (R-type)

| Instrução | Tipo | Opcode | `funct3` | `funct7` | Status |
|---|:---:|:---:|:---:|:---:|:---:|
| `XOR` | R | `0110011` | `100` | `0000000` | ✅ |
| `SLL` | R | `0110011` | `001` | `0000000` | ✅ |
| `SRL` | R | `0110011` | `101` | `0000000` | ✅ |
| `SRA` | R | `0110011` | `101` | `0100000` | ✅ |
| `SLTU` | R | `0110011` | `011` | `0000000` | ✅ |

**Modificações necessárias:**
- **`src/pl_alu_ctrl.sv`:** decodificação de `Funct3` e `Funct7[5]` no modo `ALUOp = 2'b10` para selecionar os códigos `ALU_XOR` (`4'd03`), `ALU_SLL` (`4'd06`), `ALU_SRL` (`4'd07`), `ALU_SRA` (`4'd08`) e `ALU_SLTU` (`4'd12`).
- **`src/pl_alu.sv`:** implementação das 5 operações em 32 bits (`SrcA ^ SrcB`, deslocamentos usando `SrcB[4:0]`, deslocamento aritmético com `$signed(SrcA) >>> SrcB[4:0]` e comparação sem sinal `SrcA < SrcB`).

---

### Etapa 02 — 03/11/2026: I-type e U-type

#### Aritmética, lógica e deslocamentos com imediatos (I-type)

| Instrução | Tipo | Opcode | `funct3` | `funct7` / `imm[11:5]` | Status |
|---|:---:|:---:|:---:|:---:|:---:|
| `ADDI` | I | `0010011` | `000` | — | ✅ |
| `ANDI` | I | `0010011` | `111` | — | ✅ |
| `ORI` | I | `0010011` | `110` | — | ✅ |
| `SLTI` | I | `0010011` | `010` | — | ✅ |
| `SLLI` | I | `0010011` | `001` | `0000000` | ✅ |
| `SRLI` | I | `0010011` | `101` | `0000000` | ✅ |
| `SRAI` | I | `0010011` | `101` | `0100000` | ✅ |
| `XORI` | I | `0010011` | `100` | — | ✅ |

#### Imediato superior (U-type)

| Instrução | Tipo | Opcode | Comportamento | Status |
|---|:---:|:---:|---|:---:|
| `LUI` | U | `0110111` | `rd = imm[31:12] << 12` (`0 + ImmExt`) | ✅ |
| `AUIPC` | U | `0010111` | `rd = PC + (imm[31:12] << 12)` | ✅ |

**Modificações necessárias:**
- **`src/pl_control.sv`:** reconhecimento dos opcodes `I_TYPE` (`7'b0010011`, `ALUOp = 2'b11`, `ALUSrc = 1`, `RegWrite = 1`), `LUI` (`7'b0110111`, `ALUSrcA = 2'b10`, `ALUSrc = 1`, `ALUOp = 2'b00`) e `AUIPC` (`7'b0010111`, `ALUSrcA = 2'b01`, `ALUSrc = 1`, `ALUOp = 2'b00`).
- **`src/pl_sign_ext.sv`:** extensão de sinal do imediato de 12 bits (`Instr[31:20]`) para `I_TYPE` e montagem do imediato superior (`{Instr[31:12], 12'b0}`) para `LUI`/`AUIPC`.
- **`src/pl_alu_ctrl.sv`:** inclusão do caso `ALUOp = 2'b11` para mapear `Funct3` (e `Funct7[5]` em `SRLI`/`SRAI`) nas operações da ALU.
- **`src/pl_pipe_pkg.sv` e `src/pl_datapath.sv`:** propagação do sinal `alu_src_a[1:0]` no registrador `ID/EX` e adição do multiplexador `SrcA` da ALU no estágio `EX` (`2'b00`: `fwd_srca`, `2'b01`: `id_ex.pc` para `AUIPC`, `2'b10`: `32'b0` para `LUI`).

---

### Etapa 03 — 10/11/2026: Memória, Desvios e Jumps

#### Acesso à memória (I-type load e S-type)

| Instrução | Tipo | Opcode | `funct3` | Largura / Extensão | Status |
|---|:---:|:---:|:---:|---|:---:|
| `LB` | I | `0000011` | `000` | 8 bits com sinal | ✅ |
| `LH` | I | `0000011` | `001` | 16 bits com sinal | ✅ |
| `LBU` | I | `0000011` | `100` | 8 bits sem sinal (zero) | ✅ |
| `LHU` | I | `0000011` | `101` | 16 bits sem sinal (zero) | ✅ |
| `SB` | S | `0100011` | `000` | Escrita de 1 byte | ✅ |
| `SH` | S | `0100011` | `001` | Escrita de 2 bytes | ✅ |

**Modificações necessárias (memória):**
- **`src/pl_pipe_pkg.sv`:** propagação de `funct3` para o registrador `EX/MEM` (`ex_mem.funct3`).
- **`src/pl_dmem.sv`:** adição da entrada `ByteEnable[3:0]` para escrita seletiva por lane de byte na RAM síncrona.
- **`src/pl_datapath.sv`:**
  - **Stores (`SB`, `SH`, `SW`):** deslocamento do dado (`write_data << {alu_result[1:0], 3'b000}`) e geração de `ram_byte_enable` conforme `ex_mem.funct3` e `alu_result[1:0]`.
  - **Loads (`LB`, `LH`, `LBU`, `LHU`, `LW`):** extração da sub-palavra lida (`dmem_rd >> byte_shift`) e extensão de sinal ou zero conforme `ex_mem.funct3`. Acessos ao MMIO continuam restritos a palavras completas (`funct3 == 3'b010` com `alu_result[1:0] == 2'b00`).

#### Desvios condicionais (B-type)

| Instrução | Tipo | Opcode | `funct3` | Condição | Status |
|---|:---:|:---:|:---:|---|:---:|
| `BNE` | B | `1100011` | `001` | `rs1 != rs2` | ✅ |
| `BLT` | B | `1100011` | `100` | `rs1 < rs2` (signed) | ✅ |
| `BGE` | B | `1100011` | `101` | `rs1 >= rs2` (signed) | ✅ |
| `BLTU` | B | `1100011` | `110` | `rs1 < rs2` (unsigned) | ✅ |
| `BGEU` | B | `1100011` | `111` | `rs1 >= rs2` (unsigned) | ✅ |

**Modificações necessárias (branches):**
- **`src/pl_datapath.sv`:** substituição da condição fixa de `BEQ` no estágio `EX` por um bloco `case (id_ex.funct3)` avaliando `fwd_srca` e `fwd_srcb` (já após forwarding), mantendo o cálculo de destino `id_ex.pc + id_ex.imm_ext` e o flush de `IF/ID` e `ID/EX` quando tomado.

#### Jumps (J-type e I-type)

| Instrução | Tipo | Opcode | `funct3` | Alvo / Retorno | Status |
|---|:---:|:---:|:---:|---|:---:|
| `JAL` | J | `1101111` | — | `PC = PC + imm_J`, `rd = PC + 4` | ✅ |
| `JALR` | I | `1100111` | `000` | `PC = (rs1 + imm_I) & ~1`, `rd = PC + 4` | ✅ |

**Modificações necessárias (jumps):**
- **`src/pl_control.sv` e `src/pl_pipe_pkg.sv`:** decodificação de `JAL` (`Jump = 2'b01`, `RegWrite = 1`) e `JALR` (`Jump = 2'b10`, `RegWrite = 1`), propagando `jump[1:0]` pelo registrador `ID/EX`.
- **`src/pl_sign_ext.sv`:** suporte ao imediato de 20 bits deslocado com extensão de sinal do tipo J (`JAL`) e de 12 bits do tipo I para `JALR`.
- **`src/pl_datapath.sv`:** cálculo do alvo em `EX` (`id_ex.pc + id_ex.imm_ext` para `JAL`; `(fwd_srca + id_ex.imm_ext) & 32'hfffffffe` para `JALR`), ativação de `pc_src` (com o mesmo flush de `IF/ID` e `ID/EX` usado em branches) e seleção de `ex_result = id_ex.pc + 32'd4` para escrita em `rd` e forwarding via `EX/MEM`/`MEM/WB`.

---

## Resumo das Modificações nos Módulos

| Arquivo | Papel nas Etapas 01–03 |
|---|---|
| `src/pl_alu.sv` | Operações `XOR`, `SLL`, `SRL`, `SRA` e `SLTU` (Etapa 01, reutilizadas na Etapa 02) |
| `src/pl_alu_ctrl.sv` | Decodificação de `R-type` estendido (Etapa 01) e `ALUOp = 2'b11` para `I-type` (Etapa 02) |
| `src/pl_control.sv` | Opcodes `I_TYPE`, `LUI`, `AUIPC` (Etapa 02) e `JAL`, `JALR` (Etapa 03) |
| `src/pl_sign_ext.sv` | Imediatos I, U (`LUI`/`AUIPC`) e J (`JAL`) |
| `src/pl_pipe_pkg.sv` | Campos `alu_src_a` e `jump` em `id_ex_t`; `funct3` em `ex_mem_t` |
| `src/pl_dmem.sv` | Escrita mascarada por byte (`ByteEnable[3:0]`) para `SB` e `SH` |
| `src/pl_datapath.sv` | Mux `ALUSrcA` (`LUI`/`AUIPC`), 6 condições de branch em `EX`, cálculo/retorno `PC+4` de `JAL`/`JALR`, alinhamento e extensão em sub-word loads/stores |

---

## Estrutura do Repositório

```text
repo_apresentacao/
├── src/                        RTL SystemVerilog do processador, MMIO, UART e testbench
├── assembler/                  Montador Python (assembler.py) e programas de teste (.asm)
│   ├── etapa01_test.asm        Programa de teste das instruções R-type (Etapa 01)
│   ├── etapa03_test.asm        Programa de teste de memória sub-word, branches e jumps (Etapa 03)
│   └── hello_e2.asm            Programa integrado com resultados esperados em CSV
├── dump/                       Captura/validação UART (serial_dump.py, test_serial_dump.py, capturas)
├── quartus/                    Projeto Quartus Prime (pl_top.qpf, pl_top.qsf, .sdc, .sof e MIF/HEX)
└── etapa03_placa_01.*          Evidência de captura na placa para a Etapa 03
```

---

## Fluxo Rápido de Montagem, Síntese e Validação

1. **Montar programa para FPGA com dump serial embarcado:**
   ```bash
   cd assembler
   python3 assembler.py --dump etapa03_test.asm
   ```
   O script gera e sincroniza `instruction.mif`, `data.mif`, `program.hex` e `data.hex` em `src/` e `quartus/`.

2. **Sintetizar e gravar na DE2-115 (Quartus Prime):**
   - Abrir `quartus/pl_top.qpf` (dispositivo `EP4CE115F29C7`, top-level `pl_top` com PLL 50 MHz → 10 MHz) e compilar/programar o `.sof`.

3. **Capturar e validar via UART (9600 8N1):**
   ```bash
   python3 dump/serial_dump.py COM3 --raw-out dump/captura.bin --out dump/captura.txt --check-etapa03
   ```
   Também há testes unitários offline do parser/validador em `python3 -m unittest discover -s dump -p 'test_serial_dump.py' -v`.
