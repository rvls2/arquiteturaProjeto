# Arquitetura Projeto

### Equipe:
- Deyvson Gabriel Bastos Fernandes (dgbf)
- Lucas José Duarte Cavalcanti (ljds)
- Milla Rwana de Araújo Silva (mras3)
- Rayssa Vitória Lima da Silva (rvls2)

## Introdução
Nesse projeto, buscamos implementar novas instruções a um [projeto pré-definido](https://github.com/joaopmarinho/Risc-v-Pipeline). Implementamos na ordem de categoria das instruções, iniciando com lógicas, aritméticas e de deslocamento, seguido por desvios condicionais e memória, finalizando com chamadas de funções. Para implementar cada instrução, foi preciso alterar módulos de controle, desvio, gerador de imediato, unidade lógica aritmética.

## Instruções Lógicas e Aritméticas
Sinais das instruções:  
| Instruções | ALUSrc | MemtoReg | RegWrite | RWSel | MemRead | MemWrite | ALUOp | Branch | JalR |
|------------|--------|----------|----------|-------|---------|----------|-------|--------|------|
| 'sub, xor, slt' | 0 | 0        | 1        | 0     | 0       | 0        | 10    | 0      | 0    |
| 'addi, slti' | 1    | 0        | 1        | 0     | 0       | 0        | 10    | 0      | 0    |

Como o projeto base não tinha instruções I-Type, precisamos implementar os sinais desse tipo de instrução no módulo de Controller. Este tipo utiliza imediato (ALUSrc = 1), escreve nos registradores (RegWrite = 1) e ALUOp = 10. Segue as modificações deste módulo:  
  …  
  logic [6:0] …, I_TYPE;  
  
  …  
  assign I_TYPE = 7'b0010011;  // I type  
  
  assign ALUSrc = (... || Opcode == I_TYPE);  
  assign RegWrite = (... || Opcode == I_TYPE);  
  …  
  assign ALUOp[1] = (... || Opcode == I_TYPE);  

O próximo módulo a ser alterado foi o ALUController, essa modificação é necessária para alterar a saída Operation para sua utilização na alu para as instruções de sub e xor (que antes resultavam no Operation = 0010, operação na alu que realiza soma). sub gera o Operation = 0110 e xor gera o Operation = 0011. Segue as modificações deste módulo:  

  …  
  assign Operation[0] = … ||  
      ((ALUOp == 2'b10) && (Funct3 == 3'b100) && (Funct7 == 7'b0000000));  // R\I-xor  
  
  assign Operation[1] = … ||  
      ((ALUOp == 2'b10) && (Funct3 == 3'b100) && (Funct7 == 7'b0000000)); // R\I-xor  
  
  assign Operation[2] =  … ||  
      ((ALUOp == 2'b10) && (Funct3 == 3'b000) && (Funct7 == 7'b0100000)); // R\I-sub  

No módulo imm_Gen, foi preciso incluir instruções I_Type no mesmo imediato gerado por instruções de load, pois a geração de imediato desses tipos são iguais.  

No módulo alu, foi preciso implementar novos casos para valores de Operation. Observamos que a instrução addi já estava implementada, desde que seu imediato era escolhido antes de entrar na alu, ele usa o Operation de soma (0010). Seguem as modificações deste módulo:  
