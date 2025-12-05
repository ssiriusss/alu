module CYBERcobra (
  input  logic         clk_i,
  input  logic         rst_i,
  input  logic [15:0]  sw_i,
  output logic [31:0]  out_o
);

  // 1. Создание счётчика команд и вспомогательных проводов
  logic [31:0] pc;                 // Программный счётчик
  logic [31:0] instruction;        // Текущая инструкция
  logic [31:0] pc_next;            // Следующее значение PC
  logic [31:0] pc_inc;             // PC + 4
  logic [31:0] pc_offset;          // PC со смещением
  logic [31:0] pc_sum;             // Результат сложения для PC
  
  // Поля инструкции
  logic        J;                 
  logic        B;              
  logic [1:0]  WS;               
  logic [4:0]  ALUop;             
  logic [4:0]  RA1;               
  logic [4:0]  RA2;               
  logic [4:0]  WA;              
  logic [22:0] RF_const;         
  logic [7:0]  offset_const;      
  
  // Сигналы для ALU
  logic [31:0] operand1;        
  logic [31:0] operand2;          
  logic [31:0] ALUresult;         
  logic        ALUflag;           
  
  // Сигналы для регистрового файла
  logic [31:0] write_data;         // Данные для записи в RF
  logic        write_enable;       // Разрешение записи в RF
  
  // Сигналы для мультиплексоров
  logic [31:0] pc_mux_out;         // Выход мультиплексора PC
  logic        pc_mux_sel;         // Сигнал выбора для мультиплексора PC
  
  // 2. Декодирование полей инструкции
  assign J = instruction[31];
  assign B = instruction[30];
  assign WS = instruction[29:28];
  assign ALUop = instruction[27:23];
  assign RA1 = instruction[22:18];
  assign RA2 = instruction[17:13];
  assign WA = instruction[4:0];
  assign RF_const = instruction[27:5];
  assign offset_const = instruction[12:5];
  
  // 3. Экземпляры модулей
  
  // Память инструкций (imem)
  instr_mem imem (
    .read_addr_i (pc),
    .read_data_o (instruction)
  );
  
  // АЛУ
  alu aluUnit (
    .a_i       (operand1),
    .b_i       (operand2),
    .result_o  (ALUresult),
    .alu_op_i  (ALUop),
    .flag_o    (ALUflag)
  );
  
  // Регистровый файл
  register_file regFile (
    .clk_i          (clk_i),
    .write_enable_i (write_enable),
    .write_addr_i   (WA),
    .read_addr1_i   (RA1),
    .read_addr2_i   (RA2),
    .write_data_i   (write_data),
    .read_data1_o   (operand1),
    .read_data2_o   (operand2)
  );
  
  // Сумматор для вычисления PC + смещение
  // Создаём 32-битный сумматор с явным подключением бита переноса
  logic [31:0] adder_a;
  logic [31:0] adder_b;
  logic        adder_cin;
  logic [31:0] adder_sum;
  logic        adder_cout;  // Не используется, но декларируем
  
  // Присваиваем входы сумматора
  assign adder_a = pc;
  assign adder_b = {{24{offset_const[7]}}, offset_const};  // Знаковое расширение
  assign adder_cin = 1'b0;  // Обязательно подаём нулевое значение на входной бит переноса
  
  // Экземпляр сумматора
  adder_32bit pc_adder (
    .a_i    (adder_a),
    .b_i    (adder_b),
    .cin_i  (adder_cin),
    .sum_o  (adder_sum),
    .cout_o (adder_cout)  // Выходной бит переноса не подключаем
  );
  
  assign pc_offset = adder_sum;
  
  // 4. Логика программного счётчика
  assign pc_inc = pc + 32'd4;  // PC + 4 для обычного инкремента
  
  // Программный счётчик с синхронным сбросом
  always_ff @(posedge clk_i) begin
    if (rst_i) begin
      pc <= 32'b0;
    end else begin
      pc <= pc_next;
    end
  end
  
  // 5. Сигнал управления мультиплексором, выбирающим слагаемое для PC
  // Выбираем PC + смещение если (branch AND flag) OR jump
  assign pc_mux_sel = (B && ALUflag) || J;
  
  // 6. Мультиплексор, выбирающий слагаемое для программного счётчика
  always_comb begin
    if (pc_mux_sel) begin
      pc_mux_out = pc_offset;  // PC + смещение
    end else begin
    assign pc_next = pc_mux_out;  // Следующее значение PC
  
  // 7. Сигнал разрешения записи в регистровый файл
  // Запись разрешена когда НЕ (branch OR jump)
  assign write_enable = !(B || J);
  
  // 8. Мультиплексор, выбирающий источник записи в регистровый файл
  always_comb begin
    case (WS)
      2'b00: begin  // Константа из инструкции
        write_data = {{9{RF_const[22]}}, RF_const};
      end
      2'b01: begin  // Результат ALU
        write_data = ALUresult;
      end
      2'b10: begin  // Внешний источник (переключатели)
        write_data = {{16{sw_i[15]}}, sw_i};  // Знаковое расширение
      end
      2'b11: begin  // Резерв/ноль
        write_data = 32'b0;
      end
      default: begin
        write_data = 32'b0;
      end
    endcase
  end
  
  // 9. Выходной сигнал
  assign out_o = operand1;
  
endmodule
      pc_mux_out = pc_inc;     // PC + 4
    end
  end
