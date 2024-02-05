# VM
#include <stdio.h>

#define MEM_SIZE 65536

typedef enum {
    OP_BR,  // условный переход
    OP_ADD, // сложение
    OP_LD,  // загрузка
    OP_ST,  // сохранение
    OP_JSR, // вызов подпрограммы
    OP_AND, // логическое И
    OP_LDR, // загрузка с относительным адресом
    OP_STR, // сохранение с относительным адресом
    OP_RTI, // возврат из прерывания
    OP_NOT, // логическое отрицание
    OP_LDI, // загрузка по адресу с относительным адресом
    OP_STI, // сохранение по адресу с относительным адресом
    OP_JMP, // безусловный переход
    OP_RES, // зарезервировано (не используется)
    OP_LEA, // загрузка адреса
    OP_TRAP // вызов прерывания
} Opcode;

typedef enum {
    FL_POS = 1 << 0, // флаг положительного результата
    FL_ZRO = 1 << 1, // флаг нулевого результата
    FL_NEG = 1 << 2  // флаг отрицательного результата
} ConditionFlag;

typedef struct {
    int reg[8];          // регистры
    unsigned short pc;   // счетчик команд
    unsigned short ir;   // регистр команды
    unsigned short psr;  // регистр состояния программы
    unsigned short mar;  // регистр адреса памяти
    unsigned short mdr;  // регистр данных памяти
    unsigned short cc;   // регистр флагов
    unsigned short mem[MEM_SIZE]; // память
} LC3VM;

void update_flags(LC3VM* vm, unsigned short result) {
    if (result == 0) {
        vm->cc = FL_ZRO;
    } else if (result >> 15) {
        vm->cc = FL_NEG;
    } else {
        vm->cc = FL_POS;
    }
}

unsigned short sign_extend(unsigned short x, int bit_count) {
    if ((x >> (bit_count - 1)) & 1) {
        x |= (0xFFFF << bit_count);
    }
    return x;
}

int sext(int x, int bit_count) {
    if ((x >> (bit_count - 1)) & 1) {
        x |= (0xFFFFFFFF << bit_count);
    }
    return x;
}

void lc3_vm_execute(LC3VM* vm) {
    while (1) {
        unsigned short instr = vm->mem[vm->pc++];
        Opcode opcode = instr >> 12;

        switch (opcode) {
            case OP_ADD: {
                unsigned short dr = (instr >> 9) & 0x7;
                unsigned short sr1 = (instr >> 6) & 0x7;
                unsigned short imm_flag = (instr >> 5) & 0x1;

                if (imm_flag) {
                    unsigned short imm5 = sign_extend(instr & 0x1F, 5);
                    vm->reg[dr] = vm->reg[sr1] + imm5;
                } else {
                    unsigned short sr2 = instr & 0x7;
                    vm->reg[dr] = vm->reg[sr1] + vm->reg[sr2];
                }

                update_flags(vm, vm->reg[dr]);
                break;
            }

            case OP_AND: {
                unsigned short dr = (instr >> 9) & 0x7;
                unsigned short sr1 = (instr >> 6) & 0x7;
                unsigned short imm_flag = (instr >> 5) & 0x1;

                if (imm_flag) {
                    unsigned short imm5 = sign_extend(instr & 0x1F, 5);
                    vm->reg[dr] = vm->reg[sr1] & imm5;
                } else {
                    unsigned short sr2 = instr & 0x7;
                    vm->reg[dr] = vm->reg[sr1] & vm->reg[sr2];
                }

                update_flags(vm, vm->reg[dr]);
                break;
            }

            case OP_NOT: {
                unsigned short dr = (instr >> 9) & 0x7;
                unsigned short sr = (instr >> 6) & 0x7;
                vm->reg[dr] = ~vm->reg[sr];
                update_flags(vm, vm->reg[dr]);
                break;
            }

            case OP_LD: {
                unsigned short dr = (instr >> 9) & 0x7;
                unsigned short pc_offset = sign_extend(instr & 0x1FF, 9);
                vm->reg[dr] = vm->mem[vm->pc + pc_offset];
                update_flags(vm, vm->reg[dr]);
                break;
            }

            case OP_LDI: {
                unsigned short dr = (instr >> 9) & 0x7;
                unsigned short pc_offset = sign_extend(instr & 0x1FF, 9);
                unsigned short addr = vm->mem[vm->pc + pc_offset];
                vm->reg[dr] = vm->mem[addr];
                update_flags(vm, vm->reg[dr]);
                break;
            }

            case OP_LEA: {
                unsigned short dr = (instr >> 9) & 0x7;
                unsigned short pc_offset = sign_extend(instr & 0x1FF, 9);
                vm->reg[dr] = vm->pc + pc_offset;
                update_flags(vm, vm->reg[dr]);
                break;
            }

            case OP_ST: {
                unsigned short sr = (instr >> 9) & 0x7;
                unsigned short pc_offset = sign_extend(instr & 0x1FF, 9);
                vm->mem[vm->pc + pc_offset] = vm->reg[sr];
                break;
            }

            case OP_STI: {
                unsigned short sr = (instr >> 9) & 0x7;
                unsigned short pc_offset = sign_extend(instr & 0x1FF, 9);
                unsigned short addr = vm->mem[vm->pc + pc_offset];
                vm->mem[addr] = vm->reg[sr];
                break;
            }

            case OP_JMP: {
                unsigned short base_r = (instr >> 6) & 0x7;
                vm->pc = vm->reg[base_r];
                break;
            }

            case OP_JSR: {
                unsigned short long_flag = (instr >> 11) & 0x1;
                vm->reg[7] = vm->pc;

                if (long_flag) {
                    unsigned short pc_offset = sign_extend(instr & 0x7FF, 11);
                    vm->pc += pc_offset;
                } else {
                    unsigned short base_r = (instr >> 6) & 0x7;
                    vm->pc = vm->reg[base_r];
                }
                break;
            }

            case OP_TRAP: {
                unsigned short trap_vect = instr & 0xFF;

                switch (trap_vect) {
                    case 0x20: // GETC
                        // Ввод символа
                        vm->reg[0] = getchar();
                        break;
                    case 0x21: // OUT
                        // Вывод символа
                        putchar(vm->reg[0]);
                        fflush(stdout);
                        break;
                    case 0x22: // PUTS
                        // Вывод строки
                        for (unsigned short* c = vm->mem + vm->reg[0]; *c; ++c) {
                            putchar(*c);
                        }
                        fflush(stdout);
                        break;
                    case 0x23: // IN
                        // Ввод строки
                        break;
                    case 0x25: // HALT
                        // Остановка программы
                        return;
                }
                break;
            }

            default:
                // Неизвестная операция
                break;
        }
    }
}

void load_program(LC3VM* vm, unsigned short* program, int program_size) {
    for (int i = 0; i < program_size; ++i) {
        vm->mem[i] = program[i];
    }
}

int main() {
    LC3VM vm;
    // Инициализация виртуальной машины

    // Пример программы для сложения чисел
    unsigned short program[] = {
        0xF022, // GETC
        0xA004, // OUT
        0xF022, // GETC
        0xA005, // OUT
        0x3001, // LD R0, num1
        0x3002, // LD R1, num2
        0x1002, // ADD R0, R0, R1
        0xA006  // OUT
    };

    int program_size = sizeof(program) / sizeof(program[0]);

    load_program(&vm, program, program_size);

    lc3_vm_execute(&vm);

    return 0;
}
