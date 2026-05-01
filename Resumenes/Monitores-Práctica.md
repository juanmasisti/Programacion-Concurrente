# 🧠 Guía Definitiva para Arquitectura de Monitores

Esta guía establece el paso a paso para desarmar cualquier enunciado de concurrencia y transformarlo en una arquitectura de procesos y monitores eficiente, evitando cuellos de botella y deadlocks.

---

## Paso 1: Identificar los Entes Activos (Los Procesos)
El primer paso es definir quiénes ejecutan código. Todo lo que "tiene vida propia" en el enunciado es un proceso.

*   **Los protagonistas físicos:** ¿Quiénes se mueven, llegan, piden o trabajan solos? (Ej: `Process Jugador`, `Process Cliente`, `Process Alumno`).
*   **Los coordinadores de tiempo (¡Clave!):** Si el enunciado menciona acciones que toman tiempo de forma independiente que bloquean el flujo, como *"juegan 50 minutos"*, *"viajan"* o *"simular el paso del tiempo"*, **necesitás un proceso extra coordinador** (Ej: `Process Partido`, `Process Reloj`). Nunca hagas un `delay()` largo adentro de un monitor.
*   **Identificar acciones de "Solo Lectura":** Si el proceso obtiene un dato estático usando una función (Ej: "conoce su equipo llamando a `DarEquipo()`"), esa función va **afuera del monitor** en el código del proceso, ya que no requiere exclusión mutua y maximiza la concurrencia.

---

## Paso 2: Identificar los "Puntos de Choque" (Los Monitores)
Un monitor es un "cuello de botella controlado" donde los procesos chocan para tomar una decisión, evaluar un estado o esperarse. Marcá en el enunciado cada vez que los procesos deban sincronizarse:

*   **Barreras de grupo:** *"Cuando un equipo está listo (han llegado los 5)..."*. Se necesita un lugar que cuente hasta 5. -> `Monitor Equipo`
*   **Reparto de recursos globales:** *"Asignar la cancha 1 o 2..."*. Se necesita un lugar que lleve la cuenta total de equipos formados. -> `Monitor Administrador`
*   **Sincronización de eventos (Rendezvous):** *"Cuando los 10 llegaron, comienza el partido"*. Se necesita un lugar que junte a los jugadores con el árbitro/reloj. -> `Monitor Cancha`

**REGLA DE ORO (Peligro de Deadlock):** Evitá el anidamiento de monitores. Un proceso no debería entrar a un monitor (ej. `Equipo`) y desde adentro de sus procedures llamar a otro monitor (ej. `Administrador`). Hacé que el proceso salga del primer monitor y luego llame al segundo.

---

## Paso 3: Definir la Granularidad (¿Un Monitor Único o un Arreglo?)
Para saber el alcance del monitor, preguntate: *¿Qué variable estoy protegiendo y a quiénes impacta?*

*   **Variables de recursos independientes -> ARREGLO DE MONITORES (`Monitor [1..N]`):**
    *   *Pregunta:* ¿Si el Equipo 1 se está armando, interfiere con el Equipo 2 que se está armando al mismo tiempo?
    *   *Respuesta:* No, son independientes. Usar `Monitor Equipo [1..4]`. Maximiza la concurrencia.
*   **Variables de estado global -> UN SOLO MONITOR CENTRAL:**
    *   *Pregunta:* ¿Asignar si toca la cancha 1 o 2 interfiere con el resto?
    *   *Respuesta:* Sí, porque depende de un contador global estricto (*"los primeros dos a la 1, los otros a la 2"*). Usar un único `Monitor Organizador/Administrador`.

---

## Paso 4: Diseñar las "Camas" (Variables Condición)
Las variables condición (`cond`) son las salas de espera. Elegí la estructura correcta según a quién necesites despertar:

*   **Despertar a todos (ej: El Pitazo Final del partido):** 
    *   *Cuándo:* Todos esperan el mismo evento y se liberan al mismo tiempo.
    *   *Cómo:* Una sola variable `cond espera;` y se despierta con `signal_all(espera);`.
*   **Despertar ordenado por prioridad o ID (Atención Personalizada):** 
    *   *Cuándo:* Tenés que despertar a un proceso específico (ej. "el de mayor edad", o "el ID que le toca").
    *   *Cómo:* Un arreglo de variables `cond espera[N];` y se despierta específicamente con `signal(espera[idAMiEleccion]);`.
*   **Despertar a uno solo, al azar (Orden FIFO ciego):**
    *   *Cuándo:* Hay un recurso libre y pasa el que sigue en la fila (ej. fotocopiadora básica).
    *   *Cómo:* Una sola variable `cond fila;` y se despierta con `signal(fila);`.
