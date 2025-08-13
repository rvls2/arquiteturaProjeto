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

No módulo imm_Gen, foi preciso incluir instruções I_Type no mesmo imediato gerado por instruções de load, pois a geração de imediato desses tipos são iguais.

No módulo alu, foi preciso implementar novos casos para valores de Operation. Observamos que a instrução addi já estava implementada, desde que seu imediato era escolhido antes de entrar na alu, ele usa o Operation de soma (0010). Seguem as modificações deste módulo:

             …  
             4'b0010:        // ADD, ADDI  
                    ALUResult = $signed(SrcA) + $signed(SrcB);  
             4'b0011:        // XOR  
                    ALUResult = SrcA ^ SrcB;  
             4'b0110:        // SUB  
                    ALUResult = $signed(SrcA) - $signed(SrcB);  
             4'b1100:        // SLT, SLTI  
                    ALUResult = ($signed(SrcA) < $signed(SrcB)) ? 1 : 0;  

## Instruções de Deslocamento
Sinais das instruções:  
| Instruções | ALUSrc | MemtoReg | RegWrite | RWSel | MemRead | MemWrite | ALUOp | Branch | JalR |
|------------|--------|----------|----------|-------|---------|----------|-------|--------|------|
| 'slli, srli, srai'| 1 | 0      | 1        | 0     | 0       | 0        | 10    | 0      | 0    |

Para incluir as instruções slli, srli e srai, só foi preciso alterar o módulo alu, pois o Operation dessas instruções já estava implementado, o que nos ajudou a descobrir que Operation precisaria ser mudado para outras instruções. Observe que o imediato desse tipo de instrução são 5 bits, por isso a instrução de & 32’h1F antes de fazer o shift em questão. Segue as alterações do módulo alu:  

            …
            4'b0100:        // SLLI
                    ALUResult = SrcA << (SrcB & 32'h1F);
            4'b0101:        // SRLI
                    ALUResult = SrcA >> (SrcB & 32'h1F);
            …
            4'b0111:        // SRAI
                    ALUResult = $signed(SrcA) >>> (SrcB & 32'h1F);

## Instruções de Desvio Condicional
Sinais das instruções:  
| Instruções | ALUSrc | MemtoReg | RegWrite | RWSel | MemRead | MemWrite | ALUOp | Branch | JalR |
|------------|--------|----------|----------|-------|---------|----------|-------|--------|------|
| 'bne, bge, blt' | 0 | 0        | 0        | 0     | 0       | 0        | 01    | 1      | 0    |

Os sinais de Controller desse tipo de instrução já estavam implementados, pois o projeto base já incluía a instrução de beq. A primeira alteração realizada foi no módulo ALUController, pois o Operation estava igual ao beq (1000). bne gera Operation = 1001, bge gera Operation = 1010 e blt gera Operation = 1100. Segue as modificações de ALUController:  

      assign Operation[0] = … ||
          ((ALUOp == 2'b01) && (Funct3 == 3'b001));  // B-bne
    
      assign Operation[1] = … ||
          ((ALUOp == 2'b01) && (Funct3 == 3'b101));  // B-bge
    
      assign Operation[2] =  … ||
          ((ALUOp == 2'b01) && (Funct3 == 3'b100));  // B-blt

No módulo alu, foi preciso implementar novas operações. Observe que o Operation de blt é igual ao de slt ou slti, essa é uma escolha proposital pois essas operações são iguais na alu. Se ALUResult = 1, significa que o desvio irá acontecer, caso contrário ele não acontecerá. Seguem as modificações de alu:  

            …
            4'b1001:        // BNE
                    ALUResult = (SrcA != SrcB) ? 1 : 0;
            4'b1010:        // BGE
                    ALUResult = ($signed(SrcA) >= $signed(SrcB)) ? 1 : 0;
            4'b1100:        // SLT, SLTI, BLT
                    ALUResult = ($signed(SrcA) < $signed(SrcB)) ? 1 : 0;

## Instruções de Memória
Sinais das instruções:  
| Instruções | ALUSrc | MemtoReg | RegWrite | RWSel | MemRead | MemWrite | ALUOp | Branch | JalR |
|------------|--------|----------|----------|-------|---------|----------|-------|--------|------|
| 'lh, lb, lbu'| 1    | 1        | 1        | 0     | 1       | 0        | 00    | 0      | 0    |
| 'sh, sb'   | 1      | 0        | 0        | 0     | 0       | 1        | 00    | 0      | 0    |

O tipo das instruções de load e store já estavam implementadas, pois o projeto base já incluía as instruções lw e sw, logo o módulo Controller não precisou de novas alterações. Essas instruções também não precisaram de novos Operation ou novos casos de operações na alu, pois na alu essas instruções somam para calcular o endereço na memória.  

A maior modificação foi no módulo datamemory, a memória acessa as informações em words (4 bytes), como as instruções de lh, lb, lbu, sh e sb usam menos que 4 bytes (2 ou 1 byte) temos que acessar parte dessa memória. O endereço de leitura ou escrita é o resultado de alu (a) dividido por 4 (divisão inteira). Os bits não utilizados de a serão usados como o switch case para definir qual(is) byte(s) serão usados na escrita ou leitura. loads usam Dataout, rd e a, enquanto stores usam Datain, wd e Wr.  

Para lh, temos que os 2 bytes de interesse serão ou os dois primeiros ou os dois últimos de Dataout, a[1] será o dado do nosso switch case. Para lb ou lbu, o valor de a[1:0] definirá qual byte será o de rd, e, exclusivamente para lbu, não haverá extensão de sinal.  

Cada bit de Wr representa um byte que será escrito. Para sh, temos que o valor de Datain será os 2 primeiros bytes de wd e Wr = 1100 se a[1] = 1, ou os 2 últimos e Wr = 0011 se a[1] = 0. Para sb, temos que a[1:0] determinará qual será a posição de wd em Datain.  

Segue as modificações de datamemory:  

      raddress = {{22{1'b0}}, {a[8:2], {2{1'b0}}}};
      …
  
      if (MemRead) begin
        case (Funct3)
          …
  
          3'b000: begin  //LB
            case (a[1:0])
              2'b00: rd <= {{24{Dataout[7]}}, Dataout[7:0]};
              2'b01: rd <= {{24{Dataout[15]}}, Dataout[15:8]};
              2'b10: rd <= {{24{Dataout[23]}}, Dataout[23:16]};
              2'b11: rd <= {{24{Dataout[31]}}, Dataout[31:24]};
            endcase
          end
  
          3'b100: begin  //LBU
            case (a[1:0])
              2'b00: rd <= {24'b0, Dataout[7:0]};
              2'b01: rd <= {24'b0, Dataout[15:8]};
              2'b10: rd <= {24'b0, Dataout[23:16]};
              2'b11: rd <= {24'b0, Dataout[31:24]};
            endcase
          end
  
          3'b001: begin  //LH
            if (a[1] == 1'b0) rd <= {{16{Dataout[15]}}, Dataout[15:0]};
            else rd <= {{16{Dataout[31]}}, Dataout[31:16]};
          end
  
          …
        endcase
      end else if (MemWrite) begin
        case (Funct3)
          …
  
          3'b000: begin  //SB
            case (a[1:0])
              2'b00: begin
                Wr <= 4'b0001;
                Datain <= {24'h0, wd[7:0]};
              end
              2'b01: begin
                Wr <= 4'b0010;
                Datain <= {16'h0, wd[7:0], 8'h0};
              end
              2'b10: begin
                Wr <= 4'b0100;
                Datain <= {8'h0, wd[7:0], 16'h0};
              end
              2'b11: begin
                Wr <= 4'b1000;
                Datain <= {wd[7:0], 24'h0};
              end
            endcase
          end
  
          3'b001: begin  //SH
            if (a[1] == 1'b0) begin
              Wr <= 4'b0011;
              Datain <= {16'h0, wd[15:0]};
            end else begin
              Wr <= 4'b1100;
              Datain <= {wd[15:0], 16'h0};
            end
          end
  
          …
        endcase
      end

## Instruções de Chamada de Função
Sinais das instruções:  
| Instruções | ALUSrc | MemtoReg | RegWrite | RWSel | MemRead | MemWrite | ALUOp | Branch | JalR |
|------------|--------|----------|----------|-------|---------|----------|-------|--------|------|
| 'jal'      | 1      | 0        | 1        | 1     | 0       | 0        | 11    | 1      | 0    |
| 'jalr'     | 1      | 0        | 1        | 1     | 0       | 0        | 10    | 0      | 1    |
| 'halt'     | 0      | 0        | 0        | 0     | 0       | 0        | 11    | 1      | 0    |

Para essas instruções, novos sinais precisaram ser incluídos no Controller e nos estágios do pipeline. RWSel serve para definir se o valor a ser utilizado para registrar será o resultado do mux anterior (WrdSrc2) ou o PC+4, no caso de jal e jalr, PC+4 será o valor escolhido. Jalr é um sinal específico com um objetivo semelhante ao sinal Branch, ele existe pois, diferente de outras operações de alterar PC, o jalr não utiliza o valor de PC atual, mas sim o valor da soma de um valor de registradores com imediato (semelhante a um addi). Teremos Jalr e RWSel como novas saídas de Controller.  

Novos sinais no módulo Controller:  

      …
      assign JAL = 7'b1101111;  // jal
      assign JALR = 7'b1100111;  // jalr
      assign HALT = 7'b1111111;  // halt
    
      assign ALUSrc = (... || Opcode == JALR);
      …
      assign RegWrite = (... || Opcode == JAL || Opcode == JALR);
      assign RWSel = (Opcode == JAL || Opcode == JALR);
      …
      assign ALUOp[0] = (... || Opcode == JAL || Opcode == HALT);
      assign ALUOp[1] = (... || Opcode == JAL || Opcode == JALR || Opcode == HALT);
      assign Branch = (... || Opcode == JAL || Opcode == HALT);
      assign Jalr = (Opcode == JALR);

Para o módulo ALUController, colocaremos jal e halt em um novo Operation. Observe que jalr tem ALUOp = 10 e funct3 = 000, ou seja, esses dados já funcionam como soma, que era o nosso objetivo. Seguem as alterações nesse módulo:  

      …
      assign Operation[0] = … ||
          ((ALUOp == 2'b11));  // jal e halt
    
      …
    
      assign Operation[2] =  … ||
          ((ALUOp == 2'b11));  // jalr e halt
    
      assign Operation[3] = … ||
          ((ALUOp == 2'b11));  // jal e halt

Modificamos o módulo imm_Gen para incluir o imediato de jal, que ainda não existia no projeto base. Os bits desta instrução não seguem um formato regular. Segue as modificações do deste módulo:  

      …
      7'b1101111:  /*J-type*/
      Imm_out = {
        inst_code[31] ? 11'h7FF : 11'b0,
        inst_code[31],
        inst_code[19:12],
        inst_code[20],
        inst_code[30:21],
        1'b0
      };

Para o módulo da alu, a operação realizada será atribuir o resultado igual a 1, pois desvios condicionais utilizando 1 para dizer que o desvio acontece, no caso do jal e halt, o desvio sempre irá acontecer. Segue as alterações deste módulo:  

            …
            4'b1101:        // JAL, HALT
                    ALUResult = 1;

Um novo módulo a ser modificado será o BranchUnit. Ele terá uma nova entrada chamada Jalr, e poucas modificações. O halt seguirá a lógica de um beq x0,x0,0, onde o PC+Imm será PC, logo, PC não irá mais incrementar. Seguem as alterações deste módulo:  

      …
      assign BrPC = (Branch_Sel) ? PC_Imm : (Jalr) ? AluResult : 32'b0;  // Branch -> PC+Imm   // Otherwise, BrPC value is not important
      assign PcSel = (Branch_Sel || Jalr);  // 1:branch is taken; 0:branch is not taken(choose pc+4)

Por fim, outro módulo novo a ser modificado é o Datapath, os novos sinais das fases do pipeline foram incorporados (Jalr para B e RWSel para B, C e D), que também são novas entradas neste módulo. A maior alteração é incluir o novo mux2 para fazer a seleção entre a saída do mux anterior (o que escolhia entre a saída da alu e a saída de memória) e PC+4, para escrever esse valor no registrador de destino quando a instrução for jal ou jalr. Segue a alteração principal deste módulo:

      …
      mux2 #(32) resmux2 (
          WrmuxSrc2,
          D.Pc_Four,
          D.RWSel,
          WrmuxSrc
      );
