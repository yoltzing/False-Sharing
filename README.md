# False-Sharing
Estructura de Datos y Algoritmos II.  
Profesor: Daniel Vargas Sánchez.

## Integrantes
- 323201209 - Cline González Jocelyn Shirell
- 323148760 - Cortés González Alyson Sarahi
- 425071360 - Estrada Barrera Karen Aline
- 322111264 - Gonzalez Casamayor Yoltzin Ociel

---

## Objetivo  
Mostrar que un programa paralelo con resultados correctos puede ir más lento que el secuencial por cómo se acomodan los datos en la caché, sin que haya un error lógico.

---

## Contexto  
El false sharing (o falso compartimiento) ocurre cuando hilos en distintos núcleos de CPU modifican variables independientes que comparten la misma línea de caché (normalmente de 64 bytes). Aunque los datos no se comparten lógicamente, el hardware trata toda la línea como una unidad y provoca rebotes constantes de coherencia de caché que degradan el rendimiento.
---

## Teoria
Caché: La caché es una memoria pequeña pero ultrarrápida que guarda copias temporales de datos o instrucciones usadas con frecuencia para que el sistema pueda acceder a ellas mucho más rápido que si tuviera que ir hasta la fuente original (como la memoria RAM, el disco o la red). 
Cache Line: La unidad mínima de transferencia y almacenamiento de datos entre la memoria caché de la CPU (L1, L2, L3) y la memoria principal (RAM). 
Protocolo MESI: algoritmo estándar que utilizan los procesadores multinúcleo para mantener la coherencia de caché. (Modifies, Exclusive, Shared, Invalid)
PIP: (Priority Inheritance Protocol o PIP): mecanismo de concurrencia y sistemas en tiempo real que mitiga la inversión de prioridad: cuando una tarea de alta prioridad se bloquea esperando un recurso retenido por una de baja prioridad, la tarea de baja prioridad hereda temporalmente la prioridad alta para evitar ser desplazada por tareas intermedias, terminar su sección crítica y liberar el recurso más rápido, recuperando luego su prioridad original.

---

## Demo
### Código de prueba con un histograma
```c
#include <stdio.h>
#include <omp.h>

#define N 200000000
#define HILOS 4

int main() {
    double t0, t_seq, t_fs, t_pad, t_loc;

    // 1. SECUENCIAL (Referencia base)
    int seq[4] = {0};
    t0 = omp_get_wtime();
    for (int i = 0; i < N; i++) {
        unsigned int x = i;
        x = x * 1103515245 + 12345;
        int bin = (x / 65536) % 4;
        seq[bin]++;
    }
    t_seq = omp_get_wtime() - t0;

    // 2. FALSE SHARING: 4 hilos x 4 ints = 64 bytes (todos en 1 misma línea de caché)
    int fs[HILOS][4] = {0};
    t0 = omp_get_wtime();
    #pragma omp parallel num_threads(HILOS)
    {
        int id = omp_get_thread_num();
        #pragma omp for
        for (int i = 0; i < N; i++) {
            unsigned int x = i;
            x = x * 1103515245 + 12345;
            int bin = (x / 65536) % 4;
            fs[id][bin]++; // Invalidación constante de caché entre núcleos
        }
    }
    t_fs = omp_get_wtime() - t0;

    // 3. PADDING: 16 ints por fila = 64 bytes por hilo (4 útiles + 12 de relleno)
    int pad[HILOS][16] __attribute__((aligned(64))) = {0};
    t0 = omp_get_wtime();
    #pragma omp parallel num_threads(HILOS)
    {
        int id = omp_get_thread_num();
        #pragma omp for
        for (int i = 0; i < N; i++) {
            unsigned int x = i;
            x = x * 1103515245 + 12345;
            int bin = (x / 65536) % 4;
            pad[id][bin]++; // Cada hilo escribe en su propia línea de caché
        }
    }
    t_pad = omp_get_wtime() - t0;

    // 4. VARIABLES LOCALES (Privado + combinar)
    int total[4] = {0};
    t0 = omp_get_wtime();
    #pragma omp parallel num_threads(HILOS)
    {
        int local[4] = {0}; // Vive en el stack privado de cada hilo
        #pragma omp for
        for (int i = 0; i < N; i++) {
            unsigned int x = i;
            x = x * 1103515245 + 12345;
            int bin = (x / 65536) % 4;
            local[bin]++;
        }
        #pragma omp critical
        for (int b = 0; b < 4; b++) total[b] += local[b];
    }
    t_loc = omp_get_wtime() - t0;

    // Verificación de que el conteo es exacto en las 4 versiones
    printf("Conteo cubeta 0 -> Seq: %d | FS: %d | Pad: %d | Local: %d\n\n",
           seq[0],
           fs[0][0] + fs[1][0] + fs[2][0] + fs[3][0],
           pad[0][0] + pad[1][0] + pad[2][0] + pad[3][0],
           total[0]);

    printf("1. Secuencial:    %.4f s (Speedup: 1.00x)\n", t_seq);
    printf("2. False Sharing: %.4f s (Speedup: %.2fx)\n", t_fs,  t_seq / t_fs);
    printf("3. Con Padding:   %.4f s (Speedup: %.2fx)\n", t_pad, t_seq / t_pad);
    printf("4. Vars Locales:  %.4f s (Speedup: %.2fx)\n", t_loc, t_seq / t_loc);

    return 0;
}
```
**Compilar y ejecutar con ( Desde Codespaces ):**  
```
gcc -O2 -fopenmp src/demo.c -o demo && ./demo
```

## Resultados
```
Conteo cubeta 0 -> Seq: 50000002 | FS: 50000002 | Pad: 50000002 | Local: 50000002

1. Secuencial:    0.1280 s (Speedup: 1.00x)
2. False Sharing: 0.0620 s (Speedup: 2.06x)
3. Con Padding:   0.0320 s (Speedup: 4.00x)
4. Vars Locales:  0.0310 s (Speedup: 4.13x)
```

---

## Análisis de Resultados

| Implementación | Tiempo de ejecución | Resultado |
|---|---:|---|
| False Sharing | 62 ms | Mejora frente a la versión secuencial, pero pierde parte de la ventaja del paralelismo. |
| Padding | 32 ms | Separa los datos de cada hilo en distintas líneas de caché. |
| Variables locales | 31 ms | Cada hilo cuenta por separado y combina los resultados al final. |


| Implementación | Reducción del tiempo |
|---|---:|
| Padding | 48% menos |
| Variables locales | 50% menos |

---

## Conclusiones

False Sharing es un problema importante en la programación paralela, ya que puede disminuir el rendimiento de los programas cuando diferentes hilos modifican datos independientes que se encuentran dentro de una misma línea de caché.

Actualmente, esto es relevante porque muchos sistemas utilizan procesadores multinúcleo para realizar diferentes tareas simultáneamente, como el procesamiento de imágenes, el análisis de datos, las simulaciones científicas y el funcionamiento de servidores.

Por un lado, Padding conviene cuando diferentes hilos actualizan frecuentemente sus propios datos en memoria compartida, como sucede con los contadores. Su función es separar los datos mediante espacio adicional para evitar que compartan una misma línea de caché. Ejemplos de aplicaciones: contador y en estadísticas de solicitudes en servidores.

Por otro lado, las variables locales o privadas convienen cuando podemos dividir el trabajo entre diferentes hilos y cada uno puede obtener sus propios resultados de manera independiente, para posteriormente combinarlos. Un ejemplo es la creación de histogramas durante el procesamiento de imágenes.

---

## Referencias

- CodeLucky. (2026, 6 de enero). False Sharing: The Hidden Parallel Programming Bug
Exp/ained [Video]. YouTube. https://www.youtube.com/watch?v=HROHHmvOiuM

- Sutter, H. (2009). Effective Concurrency: Eliminate False Sharing. Dr. Dobb's Journal.

- Herlihy, M., Shavit, N., Luchangco, V., & Spear, M. (2020). The Art of Multiprocessor Programming (2.ª ed., Cap. 1: Introduction & Cap. 10: Concurrent Queues). Morgan Kaufmann.

- OpenMP Architecture Review Board. (2021). OpenMP Application Programming Interface
Specification (Versión 5.2).

---
