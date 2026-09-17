# Concurrencia vs. paralelismo

- **Concurrencia:** estructurar un programa para *manejar* varias tareas cuyos periodos de ejecución se solapan. Funciona con un solo núcleo, alternando entre tareas.
- **Paralelismo:** *ejecutar* varias tareas en el mismo instante, en núcleos distintos. Requiere varios núcleos.
- Rob Pike: *"concurrencia es lidiar con muchas cosas a la vez; paralelismo es hacer muchas cosas a la vez"*.
- **Ejemplo:** un cajero que atiende dos filas alternando es concurrencia; dos cajeros atendiendo al mismo tiempo es paralelismo.
- **Por qué importa:** define qué herramienta sirve. Si el programa espera *(red, disco)*, basta con concurrencia. Si calcula, hace falta paralelismo.

## Proceso vs. hilo

| | Proceso | Hilo |
| --- | --- | --- |
| **Memoria** | Espacio propio, aislado | Comparte la memoria del proceso |
| **Costo de creación** | Alto | Bajo |
| **Comunicación** | Mecanismos explícitos: pipes, colas, sockets | Variables compartidas |
| **Fallo** | Si cae, los demás procesos siguen | Si el proceso cae, caen todos sus hilos |
| **En Python** | `multiprocessing` | `threading` |

### Ciclo de vida de un hilo

| Estado | Qué pasa | `is_alive()` |
| --- | --- | --- |
| **Nuevo** | Objeto `Thread` creado, sin `start()` | `False` |
| **Listo** | Tras `start()`, espera turno para ejecutar | `True` |
| **Ejecución** | Ejecuta su función *(con el GIL tomado)* | `True` |
| **Bloqueado** | Espera I/O, `sleep()`, `join()` o un lock | `True` |
| **Terminado** | Su función terminó, normalmente o por excepción | `False` |

- Un hilo **no se reinicia**: llamar `start()` dos veces lanza `RuntimeError: threads can only be started once`.

## El GIL — Global Interpreter Lock

Bloque no negociable del curso.

- En **CPython** solo un hilo ejecuta bytecode de Python a la vez. El GIL protege el conteo de referencias de los objetos, que no es seguro entre hilos.
- El intérprete obliga a soltar el GIL cada cierto intervalo *(`sys.getswitchinterval()`, 5 ms por defecto)* y siempre que un hilo se bloquea esperando I/O.

### Consecuencia

- **I/O-bound** *(red, disco, esperas)*: mientras un hilo espera, suelta el GIL y otro avanza → **los hilos sí ayudan**.
- **CPU-bound** *(cálculo puro en Python)*: los hilos se turnan el GIL → **los hilos no aceleran**, y el cambio de turno puede hacerlo algo más lento.
- Las bibliotecas escritas en C *(NumPy, `hashlib`)* sueltan el GIL durante el cálculo pesado; por eso ahí los hilos sí rinden.
- **El GIL no evita condiciones de carrera.** Protege al intérprete, no a los datos del programa: `contador += 1` sigue sin ser atómico *(Clase 3)*.

> **Nota al pie:** existe un build *free-threaded* de CPython sin GIL *(PEP 703)*, experimental en 3.13 y con soporte oficial desde 3.14. No es el build por defecto; el curso trabaja con el estándar.

## Laboratorio

### 1. Hilos

```python
import threading
import time


def task(name: str, seconds: float) -> None:
    print(f"{name}: Inicia")
    time.sleep(seconds)
    print(f"{name}: Termina")


thread = threading.Thread(target=task, args=("descarga", 2))
print("Antes de Start:", thread.is_alive())
thread.start()
print("Después de Start:", thread.is_alive())
thread.join()
print("Después de join:", thread.is_alive())
```

- `args` es una **tupla**: con un solo argumento lleva coma, `args=("descarga",)`.
- **Quitar el `join()`** y ejecutar: el mensaje final aparece antes de que termine la tarea.
- **Llamar `thread.run()` en vez de `thread.start()`** ejecuta la función en el hilo principal: funciona, pero no hay concurrencia.

### 2. Demo del GIL

```python
import time
from concurrent.futures import ThreadPoolExecutor


def cpu_bound(n):
    return sum(i * i for i in range(n))


def io_bound(_):
    time.sleep(1)


def medir(func, tareas, hilos):
    inicio = time.perf_counter()
    with ThreadPoolExecutor(max_workers=hilos) as pool:
        list(pool.map(func, tareas))
    return time.perf_counter() - inicio


print(f"CPU 1 hilo:  {medir(cpu_bound, [10_000_000]*4, 1):.2f}s")
print(f"CPU 4 hilos: {medir(cpu_bound, [10_000_000]*4, 4):.2f}s")
print(f"I/O 1 hilo:  {medir(io_bound, range(4), 1):.2f}s")
print(f"I/O 4 hilos: {medir(io_bound, range(4), 4):.2f}s")
```

## Documentación

<https://github.com/orgs/programming-language-i/repositories>
