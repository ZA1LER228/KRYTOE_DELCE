# Лабораторная 1

## Тема
Исследование компилятора GCC, язык ассемблера. Связь процесса и операционной системы. Makefile, Git.

---

## Вариант задания
Вычисление **n-го числа Фибоначчи** с использованием параллельного процесса (fork + pipe).

---

## Структура проекта

```
lab1/
├── main.c          # Главный модуль: fork(), pipe(), wait()
├── fib.c           # Модуль вычисления числа Фибоначчи
├── fib.h           # Заголовочный файл
├── Makefile        # Сборка, генерация asm, очистка
├── main.s          # Ассемблер main.c (без оптимизации, -O0)
├── fib.s           # Ассемблер fib.c (с оптимизацией, -O2)
└── README.md
```

---

## Исходный код

### fib.h
```c
#ifndef FIB_H
#define FIB_H

long long fib(int n);

#endif
```

### fib.c
```c
#include "fib.h"

// Итеративное вычисление числа Фибоначчи
long long fib(int n) {
    if (n <= 1) return n;
    long long a = 0, b = 1;
    for (int i = 2; i <= n; i++) {
        long long tmp = a + b;
        a = b;
        b = tmp;
    }
    return b;
}
```

### main.c
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>
#include "fib.h"

int main() {
    int n = 10;
    int fd[2];          // fd[0] — чтение, fd[1] — запись

    if (pipe(fd) == -1) {
        perror("pipe");
        exit(EXIT_FAILURE);
    }

    pid_t pid = fork();

    if (pid == -1) {
        perror("fork");
        exit(EXIT_FAILURE);
    }

    if (pid == 0) {
        // === Дочерний процесс ===
        close(fd[0]);                          // закрыть конец чтения
        long long result = fib(n);
        write(fd[1], &result, sizeof(result)); // передать результат родителю
        close(fd[1]);
        exit(EXIT_SUCCESS);
    } else {
        // === Родительский процесс ===
        close(fd[1]);                          // закрыть конец записи
        long long result;
        read(fd[0], &result, sizeof(result));  // получить результат от дочернего
        close(fd[0]);
        wait(NULL);                            // дождаться завершения дочернего
        printf("fib(%d) = %lld\n", n, result);
    }

    return 0;
}
```

---

## Makefile

```makefile
CC = gcc
CFLAGS = -Wall -Wextra

all: fib_program

fib_program: main.o fib.o
	$(CC) $(CFLAGS) -o fib_program main.o fib.o

main.o: main.c fib.h
	$(CC) $(CFLAGS) -c main.c

fib.o: fib.c fib.h
	$(CC) $(CFLAGS) -c fib.c

asm:
	gcc -S main.c -o main.s -O0
	gcc -S fib.c  -o fib.s  -O2

clean:
	rm -f *.o *.s fib_program
```

---

## Сборка и запуск

```bash
# Сборка
make

# Запуск
./fib_program

# Генерация ассемблера
make asm

# Очистка
make clean
```

**Ожидаемый вывод:**
```
fib(10) = 55
```

---

## Анализ ассемблерного кода

### Команды генерации

```bash
gcc -S main.c -o main.s -O0   # без оптимизации
gcc -S fib.c  -o fib.s  -O2   # с оптимизацией уровня 2
```

### Разница между -O0 и -O2

| Параметр | Описание | Применение |
|----------|----------|------------|
| `-O0` | Без оптимизации. Все переменные хранятся в памяти (стеке), каждая строка кода явно видна в asm | Отладка, анализ |
| `-O2` | Включает большинство оптимизаций: инлайн, устранение мёртвого кода, оптимизация регистров | Продакшн |

### Фрагмент fib.s с комментариями (-O2)

```asm
fib:
    ; Проверка: if (n <= 1) return n
    cmpl    $1, %edi            ; сравниваем n с 1
    jle     .L_base_case        ; если n <= 1 — переход к базовому случаю

    ; Инициализация: a = 0, b = 1
    xorl    %eax, %eax          ; a = 0 (через XOR — быстрее, чем movl $0)
    movl    $1, %edx            ; b = 1

    ; Счётчик цикла i = 2, условие i <= n
.L_loop:
    ; tmp = a + b
    movq    %rax, %rcx          ; tmp = a
    addq    %rdx, %rcx          ; tmp += b

    ; a = b; b = tmp
    movq    %rdx, %rax          ; a = b
    movq    %rcx, %rdx          ; b = tmp

    ; i++, проверка i <= n
    addl    $1, %esi            ; i++
    cmpl    %esi, %edi          ; сравниваем n и i
    jge     .L_loop             ; если n >= i — продолжаем цикл

    movq    %rdx, %rax          ; возвращаем b (результат)
    ret

.L_base_case:
    movslq  %edi, %rax          ; return n (расширение int → long long)
    ret
```

**Ключевые оптимизации -O2:**
- Переменные `a`, `b`, `tmp` хранятся в регистрах (`%rax`, `%rdx`, `%rcx`), а не в стеке — нет обращений к памяти
- `xorl %eax, %eax` вместо `mov $0, %eax` — команда короче и быстрее
- Цикл не разворачивается при -O2 (разворачивание — на -O3), но убраны лишние `push`/`pop`

---

## Используемые системные вызовы

| Вызов | Назначение |
|-------|-----------|
| `pipe(fd)` | Создаёт однонаправленный канал: `fd[0]` — чтение, `fd[1]` — запись |
| `fork()` | Создаёт дочерний процесс — копию родительского |
| `write(fd[1], &result, size)` | Дочерний записывает результат в канал |
| `read(fd[0], &result, size)` | Родительский читает результат из канала |
| `wait(NULL)` | Родительский ждёт завершения дочернего (предотвращает зомби-процесс) |
| `close(fd[...])` | Закрывает неиспользуемый конец канала |

### Схема взаимодействия процессов

```
Родительский процесс          Дочерний процесс
       |                              |
   fork() ─────────────────────────► |
       |                         fib(n) вычислить
       |                         write(fd[1], result)
   read(fd[0]) ◄─────────────────────|
   wait(NULL)                    exit()
   printf(result)
```

---

## Пример работы с Git

```bash
git init
git add .
git commit -m "lab1: fibonacci via fork+pipe with Makefile and asm"
git remote add origin https://github.com/<username>/os-labs.git
git push -u origin main
```

---

## Выводы

1. **GCC и ассемблер**: флаг `-S` транслирует C-код в ассемблер. Уровень `-O0` сохраняет читаемую структуру кода, `-O2` агрессивно использует регистры и убирает лишние операции с памятью.

2. **Модульность**: разбиение на `main.c` и `fib.c` позволяет независимо оптимизировать функцию вычисления и главный модуль.

3. **fork() + pipe()**: стандартный механизм IPC в Linux. Дочерний процесс вычисляет результат и передаёт его родителю через анонимный канал. Важно закрывать неиспользуемые концы канала во избежание дедлока.


4. **Makefile**: автоматизирует сборку, позволяет пересобирать только изменившиеся модули.


# Лабораторная 2
<img width="801" height="301" alt="image" src="https://github.com/user-attachments/assets/3b69fe29-7317-4189-8447-ce5862e9c8f9" />
<img width="796" height="315" alt="image" src="https://github.com/user-attachments/assets/da63db7c-90cc-4595-8c9b-ef408e6a7233" />
<img width="793" height="645" alt="image" src="https://github.com/user-attachments/assets/2e78e17c-c791-4661-8c6a-594c843470bc" />
<img width="878" height="673" alt="image" src="https://github.com/user-attachments/assets/9a4e213c-94de-48a8-b1aa-9bf8537adc35" />
<img width="536" height="800" alt="image" src="https://github.com/user-attachments/assets/d147b1d1-33d6-46fe-9d93-2cc6449053fb" />
<img width="839" height="524" alt="image" src="https://github.com/user-attachments/assets/21c32904-d328-405c-98dc-3c3a560fea8e" />
<img width="504" height="806" alt="image" src="https://github.com/user-attachments/assets/4e7b8ba6-2b78-46f1-8ba3-0f91e5494e40" />
<img width="579" height="192" alt="image" src="https://github.com/user-attachments/assets/9d38a66a-2d1c-4e7a-a935-a0f6b9c9d567" />
<img width="620" height="297" alt="image" src="https://github.com/user-attachments/assets/4341cf54-6bb9-4c8b-9eb4-ebb763ceed3f" />
<img width="831" height="218" alt="image" src="https://github.com/user-attachments/assets/da050abb-d8ac-4108-9fbf-5afcb4918e0b" />
<img width="1046" height="592" alt="image" src="https://github.com/user-attachments/assets/c7889a4a-f412-4cc0-b0f7-9f3bd6a09e67" />
<img width="806" height="235" alt="image" src="https://github.com/user-attachments/assets/571c9229-9ff1-4c2d-afaf-a78df83f5376" />
<img width="585" height="60" alt="image" src="https://github.com/user-attachments/assets/9c6a2856-10f9-41e1-9cd7-e4ad64ae2beb" />
<img width="1043" height="432" alt="image" src="https://github.com/user-attachments/assets/becc3794-890a-486b-a758-10502271bd10" />
<img width="567" height="150" alt="image" src="https://github.com/user-attachments/assets/ed117cc2-ebf4-4e38-9913-8126687d48d5" />
<img width="582" height="352" alt="image" src="https://github.com/user-attachments/assets/3400a6cd-c0c7-4655-b5fe-9bcc76829896" />
<img width="945" height="241" alt="image" src="https://github.com/user-attachments/assets/59d27e12-98a7-427b-9e8b-e195cf2f4579" />
<img width="1051" height="565" alt="image" src="https://github.com/user-attachments/assets/e1317bf7-df35-4c06-b85a-c29468883db5" />
<img width="787" height="468" alt="image" src="https://github.com/user-attachments/assets/a7b5123d-c9a9-46ec-ac29-6aece8710250" />
<img width="1041" height="635" alt="image" src="https://github.com/user-attachments/assets/c38cf8a1-f8fa-4358-8342-a1b4cd1932f9" />
<img width="1051" height="565" alt="image" src="https://github.com/user-attachments/assets/e1317bf7-df35-4c06-b85a-c29468883db5" />
<img width="787" height="468" alt="image" src="https://github.com/user-attachments/assets/a7b5123d-c9a9-46ec-ac29-6aece8710250" />
<img width="1041" height="635" alt="image" src="https://github.com/user-attachments/assets/c38cf8a1-f8fa-4358-8342-a1b4cd1932f9" />
<img width="983" height="224" alt="image" src="https://github.com/user-attachments/assets/a15d806d-9201-4c24-938f-f72db60774b4" />
<img width="487" height="120" alt="image" src="https://github.com/user-attachments/assets/babf7760-cd13-4e1e-b753-8660b2a5167b" />
<img width="759" height="128" alt="image" src="https://github.com/user-attachments/assets/3ec2d0ba-76a2-4f18-bd07-93d0374126fc" />
<img width="915" height="158" alt="image" src="https://github.com/user-attachments/assets/3fbfb347-0a0b-4c63-bd32-60e985d6c612" />
<img width="308" height="63" alt="image" src="https://github.com/user-attachments/assets/957c4a0b-d4db-4b23-8821-fc992f63ad88" />
