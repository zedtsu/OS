## 1. Описание задачи
В рамках лабораторной работы реализована программа для численного интегрирования функции $f(x) = x^2$ на заданном отрезке $[a, b]$ методом трапеций. 

Работа разделена на следующие этапы:
1. Создание базовой последовательной версии на языке C.
2. Генерация ассемблерного кода с различными уровнями оптимизации (`-O0`, `-O2`) и его подробный анализ.
3. Разделение программы на логические модули и автоматизация сборки с помощью `Makefile`.
4. Модернизация программы: распараллеливание вычислений на два независимых процесса (`fork()`) с синхронизацией через разделяемую память POSIX (`mmap`) и семафоры (`sem_t`).

---

## 2. Ассемблерный анализ и оптимизация

Для генерации ассемблерного кода использовались команды:
```bash
gcc -S -O0 main.c -o main_O0.s
gcc -S -O2 main.c -o main_O2.s
```

### Подробный разбор ассемблерного кода функций (вариант `-O0`)

Ниже представлен участок сгенерированного компилятором GCC кода x86-64 с подробными комментариями архитектуры цикла и работы с памятью:

```assembly
.file	"main.c"
	.text
	.globl	f
	.type	f, @function
f:
	pushq	%rbp
	movq	%rsp, %rbp
	movsd	%xmm0, -8(%rbp)     # Входной аргумент x (из xmm0) сохраняем в стек
	movsd	-8(%rbp), %xmm0     # Загружаем x обратно в xmm0
	mulsd	-8(%rbp), %xmm0     # xmm0 = x * x (вычисление x^2)
	popq	%rbp
	ret                         # Возврат значения через регистр xmm0

	.globl	integrate
	.type	integrate, @function
integrate:
	pushq	%rbp
	movq	%rsp, %rbp
	subq	\$48, %rsp           # Выделяем 48 байт в стеке под локальные переменные
	movsd	%xmm0, -32(%rbp)    # Сохраняем в стек переменную 'a'
	movsd	%xmm1, -40(%rbp)    # Сохраняем в стек переменную 'b'
	movl	%edi, -44(%rbp)     # Сохраняем в стек переменную 'n' (int)
	
	# Вычисление h = (b - a) / n
	movsd	-40(%rbp), %xmm0
	subsd	-32(%rbp), %xmm0    # xmm0 = b - a
	cvtsi2sdl	-44(%rbp), %xmm1 # Конвертируем 'n' из int в double
	divsd	%xmm1, %xmm0        # xmm0 = (b - a) / n
	movsd	%xmm0, -8(%rbp)     # Сохраняем шаг 'h' в стеке

	# Вычисление стартовой суммы: sum = 0.5 * (f(a) + f(b))
	movq	-32(%rbp), %rax
	movq	%rax, %xmm0         # Передаем 'a' в f()
	call	f
	movsd	%xmm0, -56(%rbp)    # Временное сохранение f(a) в стек
	movq	-40(%rbp), %rax
	movq	%rax, %xmm0         # Передаем 'b' в f()
	call	f                   # Вызов f(b)
	addsd	-56(%rbp), %xmm0    # xmm0 = f(a) + f(b)
	movsd	.LC0(%rip), %xmm1   # Загружаем константу 0.5 из секции данных
	mulsd	%xmm1, %xmm0        # xmm0 = 0.5 * (f(a) + f(b))
	movsd	%xmm0, -16(%rbp)    # Сохраняем результат в переменную 'sum' в стеке

	# Инициализация цикла: i = 1
	movl	\$1, -20(%rbp)       # Локальная переменная 'i' = 1 в стеке
	jmp	.L4                 # Безусловный переход к проверке условия цикла

.L5:    # ТЕЛО ЦИКЛА
	movl	-20(%rbp), %eax     # Загружаем 'i' из стека в регистр общего назначения
	cvtsi2sdl	%eax, %xmm0     # Переводим счетчик 'i' во float-формат (double)
	mulsd	-8(%rbp), %xmm0     # xmm0 = i * h
	movsd	-32(%rbp), %xmm1    # Загружаем 'a' из стека
	addsd	%xmm1, %xmm0        # xmm0 = a + i * h
	call	f                   # Вызов f(a + i * h). Результат возвращается в xmm0
	movsd	-16(%rbp), %xmm1    # Загружаем текущее значение 'sum' из стека
	addsd	%xmm1, %xmm0        # sum += f(...)
	movsd	%xmm0, -16(%rbp)    # Записываем обновленную 'sum' обратно в стек
	addl	\$1, -20(%rbp)       # Инкремент счетчика: i++

.L4:    # УСЛОВИЕ ЦИКЛА
	movl	-20(%rbp), %eax     # Загружаем 'i'
	cmpl	-44(%rbp), %eax     # Сравниваем i и n (инструкция сравнения)
	jl	.L5                 # Если i < n (Jump if Less), прыгаем в тело цикла на .L5

	# Финал: return sum * h
	movsd	-16(%rbp), %xmm0    # Загружаем 'sum'
	mulsd	-8(%rbp), %xmm0     # xmm0 = sum * h
	leave                       # Восстановление указателей стека и кадра rbp
	ret                         # Выход из функции
```

### Анализ флагов оптимизации GCC
1. **Режим `-O0` (Без оптимизации):** Программа работает напрямую со стеком ОЗУ. Любое изменение переменной (например, `i++` или накопление `sum`) вызывает принудительную запись в память и чтение из неё. Это сильно замедляет выполнение, но делает код линейным и понятным для отладки.
2. **Режим `-O2` (Высокая оптимизация):** Компилятор полностью убирает работу со стеком внутри критических участков. Все переменные цикла (`i`, `sum`, `h`) постоянно удерживаются в сверхбыстрых регистрах процессора (`%xmm` и `%eax`). Более того, вызовы функции `f(x)` инлайнятся (встраиваются непосредственно в цикл), устраняя накладные расходы на команды `call` и `ret`.

---

## 3. Модульная структура проекта

Проект разделен на три логических файла: интерфейс вычислений, реализация математики и управляющий модуль ядра ОС.

### Файл `math_functions.h`
```c
#ifndef MATH_FUNCTIONS_H
#define MATH_FUNCTIONS_H

double f(double x);
double integrate_part(double a, double b, int n);

#endif
```

### Файл `math_functions.c`
```c
#include "math_functions.h"

double f(double x) {
    return x * x;
}

double integrate_part(double a, double b, int n) {
    double h = (b - a) / n;
    double sum = 0.5 * (f(a) + f(b));
    for (int i = 1; i < n; i++) {
        sum += f(a + i * h);
    }
    return sum * h;
}
```

---

## 4. Параллельное программирование и IPC в Linux

Для параллельного выполнения расчетов отрезок интегрирования разбивается пополам. Системный вызов `fork()` ветвит текущий процесс на родительский и дочерний.

Для организации межпроцессного взаимодействия (IPC) используется **Shared Memory** посредством `mmap` с флагами `MAP_SHARED | MAP_ANONYMOUS`. Синхронизация доступа к переменной в памяти реализована через **именованные семафоры POSIX** (`sem_t`), что гарантирует атомарность операции сложения и защищает от состояния гонки (Race Condition).

### Файл `main.c`
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/mman.h>
#include <sys/wait.h>
#include <fcntl.h>
#include <semaphore.h>
#include "math_functions.h"

#define SEM_NAME "/integral_sem"

int main() {
    double a = 0.0, b = 10.0;
    int n = 1000000;
    double mid = a + (b - a) / 2.0;

    // 1. Создание разделяемой памяти (Shared Memory)
    double *shared_result = mmap(NULL, sizeof(double), PROT_READ | PROT_WRITE, 
                                 MAP_SHARED | MAP_ANONYMOUS, -1, 0);
    if (shared_result == MAP_FAILED) {
        perror("mmap failed");
        return 1;
    }
    *shared_result = 0.0; 

    // 2. Инициализация семафора синхронизации
    sem_t *sem = sem_open(SEM_NAME, O_CREAT, 0644, 1);
    if (sem == SEM_FAILED) {
        perror("sem_open failed");
        return 1;
    }

    printf("[Main] Starting parallel computation...\n");

    pid_t pid = fork();

    if (pid < 0) {
        perror("fork failed");
        return 1;
    }

    if (pid == 0) {
        // --- ДОЧЕРНИЙ ПРОЦЕСС ---
        printf("[Child] Computing interval [%.1f, %.1f]\n", a, mid);
        double local_res = integrate_part(a, mid, n / 2);

        // Критическая секция
        sem_wait(sem);
        *shared_result += local_res;
        sem_post(sem);

        printf("[Child] Job done.\n");
        exit(0);
    } else {
        // --- РОДИТЕЛЬСКИЙ ПРОЦЕСС ---
        printf("[Parent] Computing interval [%.1f, %.1f]\n", mid, b);
        double local_res = integrate_part(mid, b, n / 2);

        // Критическая секция
        sem_wait(sem);
        *shared_result += local_res;
        sem_post(sem);

        // Синхронизация: ожидание завершения дочернего процесса
        wait(NULL);

        // Вывод итогового значения
        printf("[Parent] Final Integrated Result: %f\n", *shared_result);

        // Освобождение системных ресурсов IPC
        sem_close(sem);
        sem_unlink(SEM_NAME);
        munmap(shared_result, sizeof(double));
    }

    return 0;
}
```

---

## 5. Автоматизация сборки (Makefile)

Сборка проекта автоматизирована с помощью утилиты `make`. Флаги компиляции включают строгий контроль предупреждений (`-Wall -Wextra`) и оптимизацию `-O2`.

```makefile
CC = gcc
CFLAGS = -Wall -Wextra -O2
TARGET = integral_prog

all: \$(TARGET)

\$(TARGET): main.o math_functions.o
	\$(CC) \((CFLAGS) -o\)(TARGET) main.o math_functions.o

main.o: main.c math_functions.h
	\((CC)\)(CFLAGS) -c main.c

math_functions.o: math_functions.c math_functions.h
	\((CC)\)(CFLAGS) -c math_functions.c

clean:
	rm -f *.o \$(TARGET)
```

---

## 6. Фиксация версий в Git

Управление версиями проекта осуществлялось локально при помощи Git:

```bash
git init
git add main.c math_functions.c math_functions.h Makefile
git commit -m "Initial commit: Modular architecture with process-level parallelism"
```

---

## Вывод
В ходе выполнения лабораторной работы были изучены низкоуровневые принципы работы транслятора GCC. Анализ ассемблерного кода наглядно показал разницу между неоптимизированным кодом и кодом с флагом `-O2`, где операции со стеком ОЗУ заменяются регистрами процессора. Успешно освоены механизмы параллельного программирования Linux (`fork`), работа с разделяемой памятью и примитивами синхронизации для предотвращения race condition.
