# Laboratorio 3 — Parte I: Productor/Consumidor

## Integrantes
- *(Agrega tu nombre aquí)*

## ¿De qué va esto?
Básicamente tenemos un programa de Productor/Consumidor donde probamos dos formas de manejar la sincronización entre hilos: una que usa busy-wait (espera activa) y otra que usa monitores de Java (`wait()`/`notifyAll()`). La idea es ver cómo afecta cada enfoque al consumo de CPU.

## Estructura del proyecto

- `PCApp.java` — Es el main, desde ahí se configura todo: el modo (spin o monitor), cuántos productores y consumidores, la capacidad de la cola, los delays, etc.
- `BusySpinQueue.java` — La cola que usa busy-wait. Básicamente se queda dando vueltas en un `while(true)` hasta que pueda meter o sacar un elemento.
- `BoundedBuffer.java` — La cola con monitores. Usa `synchronized`, `wait()` y `notifyAll()` para no gastar CPU de más.
- `Producer.java` — El hilo que produce elementos y los mete a la cola.
- `Consumer.java` — El hilo que saca elementos de la cola.

## Comandos que usé para las pruebas

### Escenario 1: Productor lento / Consumidor rápido

Con busy-wait:
```powershell
mvn -q -DskipTests "exec:java" '-Dexec.mainClass=edu.eci.arsw.pc.PCApp' '-Dmode=spin' '-Dproducers=1' '-Dconsumers=1' '-Dcapacity=8' '-DprodDelayMs=50' '-DconsDelayMs=1' '-DdurationSec=30'
```

Con monitores:
```powershell
mvn -q -DskipTests "exec:java" '-Dexec.mainClass=edu.eci.arsw.pc.PCApp' '-Dmode=monitor' '-Dproducers=1' '-Dconsumers=1' '-Dcapacity=8' '-DprodDelayMs=50' '-DconsDelayMs=1' '-DdurationSec=30'
```

### Escenario 2: Productor rápido / Consumidor lento (cola chiquita)

Con busy-wait:
```powershell
mvn -q -DskipTests "exec:java" '-Dexec.mainClass=edu.eci.arsw.pc.PCApp' '-Dmode=spin' '-Dproducers=1' '-Dconsumers=1' '-Dcapacity=4' '-DprodDelayMs=1' '-DconsDelayMs=50' '-DdurationSec=30'
```

Con monitores:
```powershell
mvn -q -DskipTests "exec:java" '-Dexec.mainClass=edu.eci.arsw.pc.PCApp' '-Dmode=monitor' '-Dproducers=1' '-Dconsumers=1' '-Dcapacity=4' '-DprodDelayMs=1' '-DconsDelayMs=50' '-DdurationSec=30'
```

## Evidencias de VisualVM

### Escenario 1: Productor lento / Consumidor rápido

*(Insertar pantallazos aquí)*

### Escenario 2: Productor rápido / Consumidor lento

*(Insertar pantallazos aquí)*

## Análisis

### ¿Por qué el CPU se pone tan alto con busy-wait?

El problema está en `BusySpinQueue`. Si uno revisa los métodos `take()` y `put()`, ambos tienen un `while(true)` que se queda preguntando una y otra vez "¿ya hay espacio?" o "¿ya hay elementos?". Aunque usa `Thread.onSpinWait()`, eso realmente no ayuda mucho porque el hilo nunca se duerme de verdad, sigue corriendo y gastando CPU sin hacer nada útil.

### ¿Cómo se arregla?

Con `BoundedBuffer` se soluciona usando monitores. La diferencia principal es que en vez de quedarse girando en un ciclo, el hilo llama a `wait()` y se duerme de verdad. Cuando el otro hilo hace su parte (meter o sacar un elemento), llama a `notifyAll()` para despertar al que estaba esperando. Esto se puede ver en los métodos `put()` y `take()` de `BoundedBuffer`.

Lo importante es que `wait()` libera el monitor y suspende el hilo, así que no gasta CPU mientras espera. Solo se despierta cuando tiene sentido hacerlo.

### Sobre la cola acotada (escenario 2)

Cuando el productor es rápido y el consumidor lento, la cola se llena rapidísimo (más con capacity=4). En `BoundedBuffer`, cuando la cola está llena, el productor entra al `while` y hace `wait()`, así que se queda dormido hasta que el consumidor saque algo y lo despierte con `notifyAll()`. De esta forma se respeta el límite de la cola sin gastar CPU de más.

En cambio con `BusySpinQueue`, el productor se queda dando vueltas sin parar esperando que haya espacio, lo cual se nota bastante en VisualVM.

## Conclusiones

Después de probar ambos modos, queda bastante claro que el busy-wait no es viable cuando un hilo tiene que esperar mucho. Se come el CPU sin razón. Con los monitores de Java el programa se comporta mucho mejor porque los hilos realmente se duermen cuando no tienen nada que hacer, y solo se despiertan cuando les toca. Además, `BusySpinQueue` ni siquiera usa sincronización, así que aparte de gastar CPU podría tener problemas de concurrencia. En cambio `BoundedBuffer` con `synchronized` es thread-safe y eficiente.
