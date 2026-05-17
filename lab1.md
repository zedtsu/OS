# Лабораторная работа №1
## Исследование процесса компиляции и оптимизации программ на C++

**Студент:** Шугалей Александр  
**Группа:** 2271

---

## 1. Реализация программы на C++

### 1.1 Заголовочный файл `factorial.h`

```cpp
#ifndef FACTORIAL_H
#define FACTORIAL_H

unsigned long long factorial(int n);

#endif
Описание:
В заголовочном файле содержится объявление функции вычисления факториала.
Конструкция #ifndef / #define / #endif используется для защиты от повторного подключения файла.

1.2 Реализация функции factorial.cpp
cpp
#include "factorial.h"

unsigned long long factorial(int n) {
    if (n < 0)
        return 0;
    
    unsigned long long result = 1;
    
    for (int i = 1; i <= n; i++) {
        result *= i;
    }
    
    return result;
}
Описание:
Функция реализует итеративное вычисление факториала.
В отличие от рекурсивного подхода, цикл использует меньше памяти и работает быстрее для небольших значений n.
Проверка if (n < 0) не допускает вычисление факториала отрицательных чисел.
Основное вычисление происходит в цикле: result *= i, что эквивалентно result = result * i.

1.3 Главная программа main.cpp
cpp
#include <iostream>
#include <fstream>
#include <thread>
#include "factorial.h"

using namespace std;

void computeFactorial(int n) {
    unsigned long long result = factorial(n);
    
    ofstream file("result.txt");
    if (file.is_open()) {
        file << result;
        file.close();
    }
}

int main() {
    int n = 10;
    
    cout << "Calculating factorial of " << n << "..." << endl;
    
    thread calculator(computeFactorial, n);
    
    cout << "Parallel thread started..." << endl;
    cout << "Main thread continues working..." << endl;
    
    calculator.join();
    
    cout << "Thread finished." << endl;
    
    unsigned long long result;
    ifstream file("result.txt");
    if (file.is_open()) {
        file >> result;
        file.close();
    }
    
    cout << n << "! = " << result << endl;
    
    return 0;
}
2. Компиляция программы
Команда компиляции:
bash
g++ main.cpp factorial.cpp -o main.exe
Запуск программы:
bash
main.exe
Пример выполнения:
text
Calculating factorial of 10...
Parallel thread started...
Main thread continues working...
Thread finished.
10! = 3628800
3. Получение ассемблерного листинга
Без оптимизации (-O0):
bash
g++ factorial.cpp -O0 -S -o factorial_O0.asm
С оптимизацией (-O3):
bash
g++ factorial.cpp -O3 -S -o factorial_O3.asm
4. Анализ ассемблерного кода (без оптимизации, -O0)
Ниже приведён фрагмент ассемблерного кода функции вычисления факториала с комментариями.

asm
.file   "factorial.cpp"
.text
.globl  _Z9factoriali              // объявление глобальной функции
.def    _Z9factoriali; .scl 2; .type 32; .endef
.seh_proc   _Z9factoriali

_Z9factoriali:
.LFB0:
    // ===== ПРОЛОГ ФУНКЦИИ =====
    pushq  %rbp                    // сохраняем старый базовый указатель
    movq   %rsp, %rbp              // создаем новый стековый фрейм
    subq   $16, %rsp               // выделяем память под локальные переменные
    
    // сохраняем аргумент функции n
    movl   %ecx, 16(%rbp)          // n -> стек
    
    // ===== ПРОВЕРКА: if (n < 0) return 0 =====
    cmpl   $0, 16(%rbp)            // сравниваем n и 0
    jns    .L2                     // если n >= 0 → продолжаем
    movl   $0, %eax                // eax = 0 (результат)
    jmp    .L3                     // переход к выходу

.L2:
    // ===== ИНИЦИАЛИЗАЦИЯ =====
    movq   $1, -8(%rbp)            // result = 1
    movl   $1, -12(%rbp)           // i = 1
    jmp    .L4                     // переход к проверке условия цикла

.L5:
    // ===== ТЕЛО ЦИКЛА =====
    movl   -12(%rbp), %eax         // eax = i
    cltq                            // преобразование int -> long long
    movq   -8(%rbp), %rdx          // rdx = result
    imulq  %rdx, %rax              // rax = result * i
    movq   %rax, -8(%rbp)          // result = result * i
    addl   $1, -12(%rbp)           // i++

.L4:
    // ===== ПРОВЕРКА УСЛОВИЯ ЦИКЛА =====
    movl   -12(%rbp), %eax         // eax = i
    cmpl   16(%rbp), %eax          // сравниваем i и n
    jle    .L5                     // если i <= n → повторяем цикл

    // ===== ВОЗВРАТ РЕЗУЛЬТАТА =====
    movq   -8(%rbp), %rax          // загружаем result в регистр возврата

.L3:
    // ===== ЭПИЛОГ ФУНКЦИИ =====
    addq   $16, %rsp               // освобождаем стек
    popq   %rbp                    // восстанавливаем rbp
    ret                            // выход из функции
Соответствие переменных:
Переменная C++	Адрес в ASM
n	16(%rbp)
result	-8(%rbp)
i	-12(%rbp)
Анализ цикла:
Цикл реализован с помощью меток и переходов:

Метка	Назначение
.L4	проверка условия
.L5	тело цикла
jle	переход при i <= n
5. Особенности оптимизированного кода (-O3)
При использовании флага -O3 компилятор GCC выполняет максимальную оптимизацию программы.

В оптимизированной версии:

уменьшается количество инструкций;

сокращается число обращений к памяти;

локальные переменные чаще хранятся в регистрах процессора;

цикл вычисления факториала выполняется эффективнее.

По сравнению с вариантом -O0 оптимизированный код:

содержит меньше инструкций;

реже использует стек;

активнее применяет регистры процессора;

выполняется быстрее.

Однако такой ASM-код становится менее наглядным для изучения.

6. Создание Makefile
Файл Makefile
makefile
CXX = g++
CXXFLAGS = -Wall -std=c++11

all: main.exe

main.exe: main.o factorial.o
	$(CXX) main.o factorial.o -o main.exe

main.o: main.cpp factorial.h
	$(CXX) $(CXXFLAGS) -c main.cpp

factorial.o: factorial.cpp factorial.h
	$(CXX) $(CXXFLAGS) -c factorial.cpp

asm:
	$(CXX) factorial.cpp -O0 -S -o factorial_O0.asm
	$(CXX) factorial.cpp -O3 -S -o factorial_O3.asm

clean:
	del *.o main.exe *.asm result.txt

run: main.exe
	main.exe

.PHONY: all asm clean run
Использование Makefile:
Команда	Описание
mingw32-make	Сборка проекта
mingw32-make asm	Генерация ASM листингов
mingw32-make run	Запуск программы
mingw32-make clean	Очистка проекта
7. Реализация параллельного выполнения
Для демонстрации многопоточности в программе используется отдельный поток выполнения с использованием библиотеки:

cpp
#include <thread>
Создание потока:

cpp
thread calculator(computeFactorial, n);
Синхронизация потоков:

cpp
calculator.join();
Метод join() заставляет главный поток ожидать завершения вычислений.

8. Выводы
В ходе выполнения лабораторной работы были получены следующие результаты:

Разработана программа для вычисления факториала с использованием многопоточности.

Изучены этапы компиляции программы на C++.

Получены и проанализированы ассемблерные листинги с различными уровнями оптимизации (-O0 и -O3).

Выявлены различия между неоптимизированным и оптимизированным кодом.

Создан Makefile для автоматизации сборки проекта и генерации ассемблерного кода.

Изучена работа с потоками на примере параллельного вычисления факториала.
