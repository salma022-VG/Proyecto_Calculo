# Cálculo diferencial e integral con el tráfico de Medellín

Página web interactiva que enseña cálculo diferencial e integral usando datos reales de tráfico
de Medellín. Cada concepto —derivada, integral definida, sumas de Riemann, área entre curvas,
valor medio— se explica sobre una pregunta concreta que los datos pueden responder: ¿cuántos
vehículos pasan por la Avenida Las Vegas entre las 6 y las 9 de la mañana?

La idea central es que **la intensidad vehicular q(t), medida en vehículos por hora, es una tasa**.
Por eso:

- su **derivada** q′(t) dice qué tan rápido crece o cae el tráfico (veh/h por cada hora);
- su **integral** ∫q(t)dt cuenta los **vehículos acumulados** en un intervalo.

Todo lo que calcula la página se muestra con el procedimiento completo, paso a paso, con las
fórmulas y los números sustituidos. No es una calculadora: es un desarrollo escrito que se
recalcula cada vez que cambias un parámetro.

Son **diez pestañas**: seis de teoría aplicada (integral definida, sumas de Riemann, derivadas,
área entre curvas, velocidad media y un formulario), dos talleres resueltos, una que **rehace
todos los cálculos en Python** para contrastarlos, y una que lleva los resultados a un **mapa de
Medellín**.

---

## Cómo abrirlo

**En línea:** https://salma022-vg.github.io/Proyecto_Calculo/

**En tu computador:** descarga `index.html` y haz doble clic. Se abre en cualquier navegador
(Edge, Chrome, Firefox). No hay que instalar nada ni montar un servidor.

**Las ocho primeras pestañas funcionan sin conexión a internet.** Tres funciones sí la necesitan,
porque descargan una librería la primera vez que las usas en cada sesión:

| Función | Qué descarga |
|---|---|
| ⬇ Descargar Excel de metodologías | SheetJS, que arma el archivo `.xlsx` |
| 9 · Verificación | Pyodide y numpy, para ejecutar Python en el navegador |
| 10 · Mapa | Leaflet, los mosaicos del mapa y, si lo pides, los trazados de OpenStreetMap |

Si no hay conexión, cada una avisa con un mensaje y el resto de la página sigue funcionando
igual.

El archivo es **autocontenido**: los datos, el código y los estilos están dentro del mismo HTML.
Se puede enviar por correo o copiar en una memoria USB tal cual.

---

## Los datos

**Fuente:** velocidad e intensidad vehicular por corredor en Medellín, del **1 de julio al 31 de
agosto de 2020**.

El archivo trae **44 corredores** (Autopista Norte, Avenida 80, Avenida El Poblado, Calle 10,
San Juan, Vía al Mar…). Para cada corredor hay **11 agrupaciones de día**:

| Grupo | Qué incluye |
|---|---|
| Todos los días | todo el periodo disponible para ese corredor |
| Lunes a viernes | días hábiles |
| Sábados / Domingos | fin de semana separado |
| Lunes, Martes, … Domingo | cada día de la semana por separado |

**La cobertura no es igual en todos los corredores.** El periodo completo son 62 días, pero cada
corredor tiene los días que su sensor alcanzó a registrar: van de 21 a 62 días en el grupo "todos
los días", y de 15 a 44 en "lunes a viernes". Lo mismo con los puntos de medición, que van de 1 a
55 según el corredor. Por eso la página muestra siempre, debajo de los controles, cuántos días y
cuántos carriles hay detrás del perfil que estás viendo: conviene mirarlo antes de sacar
conclusiones, porque un corredor con 21 días y 1 carril no da la misma confianza que uno con 62
días y 55 carriles.

Y para cada combinación corredor × grupo se guardan cuatro cosas:

- `q` — 24 valores: intensidad promedio de cada hora, en **veh/h**
- `v` — 24 valores: velocidad promedio de cada hora, en **km/h**
- `dias` — cuántos días entraron en el promedio
- `carriles` — cuántos carriles o puntos de medición se sumaron

### Cómo se construyó el perfil horario

1. Para cada carril o punto de medición se promedia la intensidad de cada hora sobre todos los
   días del grupo elegido.
2. Se **suman** los carriles del corredor, usando solo los que tienen las 24 horas completas.
3. La velocidad es el **promedio ponderado por intensidad**, no el promedio simple: las horas con
   más vehículos pesan más, que es lo que corresponde si se quiere la velocidad que experimenta
   un vehículo típico.

Consecuencia importante de este método: **un vehículo que pasa por varios puntos de medición se
cuenta en cada uno**. Las cifras son conteos de paso por sensor, no vehículos distintos.

---

## El modelo matemático

Los datos son 24 números por día (uno por hora), pero derivar e integrar requiere una función
continua. La página construye dos modelos distintos según lo que necesite.

### Modelo principal: serie de Fourier de periodo 24 h

Como el tráfico es un fenómeno que se repite cada día, el modelo natural es una suma de senos y
cosenos con periodo 24 horas:

```
q(t) = a₀ + Σ [ aₖ·cos(kωt) + bₖ·sin(kωt) ],   k = 1 … K,   ω = 2π/24 = π/12
```

Los coeficientes salen por mínimos cuadrados sobre los 24 datos, colocados en el **centro de cada
hora** (tₕ = h + 0,5):

```
a₀ = (1/24) Σ qₕ          (el promedio del día)
aₖ = (2/24) Σ qₕ·cos(kωtₕ)
bₖ = (2/24) Σ qₕ·sin(kωtₕ)
```

**K, el número de armónicos, se controla con un deslizador** (de 1 a 11). Con K = 1 el modelo es
una sola onda suave; con K = 11 sigue los datos casi punto por punto. Esto permite ver en vivo el
compromiso entre suavidad y fidelidad.

Con este modelo las tres operaciones del curso salen **analíticamente**, término a término:

```
q′(t)  = Σ kω·[ −aₖ·sin(kωt) + bₖ·cos(kωt) ]
q″(t)  = −Σ (kω)²·[ aₖ·cos(kωt) + bₖ·sin(kωt) ]
Q(t)   = a₀t + Σ (1/kω)·[ aₖ·sin(kωt) − bₖ·cos(kωt) ] + C
```

Hay una propiedad que hace bonito el ajuste: como a₀ es el promedio de los datos y los términos
oscilantes se cancelan sobre un periodo completo, se cumple exactamente

```
∫₀²⁴ q(t) dt = 24·a₀ = Σ qₕ·1h = suma de los conteos horarios
```

Es decir, **el modelo continuo y los datos discretos dan el mismo total diario**, sea cual sea K.
Sobre un subintervalo [a, b] ya no coinciden, y esa diferencia es justamente la que la página
muestra como error de modelado.

### Modelo secundario: polinomio por mínimos cuadrados

En las pestañas de taller (7 y 8) se usa además un **polinomio de grado 2 o 3** ajustado solo a
los datos que caen dentro de la ventana [a, b]:

```
P(t) = c₀ + c₁t + c₂t² + c₃t³
```

Se resuelven las ecuaciones normales por eliminación gaussiana con pivoteo parcial (escalando t
para no perder precisión) y se reporta el **R²**. Este modelo es el que se puede reproducir con
fórmulas de Excel, y por eso es el que viaja al libro descargable.

### Métodos numéricos que usa por dentro

- **Raíces** (puntos críticos, inflexiones, cortes entre curvas, v(t) = v̄): barrido de 2400
  subintervalos buscando cambios de signo, y 60 iteraciones de bisección en cada uno.
- **Máximos de |q′| y |q″|**: muestreo de 600 puntos en el intervalo.
- **Verificación de ∫t·q(t)dt**: regla de Simpson con 2000 subintervalos, para contrastar el
  resultado analítico de la integración por partes.

---

## Las diez pestañas

### 1 · Integral definida

Calcula **cuántos vehículos pasan entre las horas a y b** y desarrolla el Teorema Fundamental del
Cálculo completo.

Muestra en tarjetas: los vehículos según la integral, los vehículos según los conteos, el flujo
medio del intervalo, el porcentaje que representa del día y el total diario.

El paso a paso incluye:

1. **Planteamiento** — si N(t) son los vehículos acumulados, N′(t) = q(t), luego
   N(b) − N(a) = ∫ₐᵇ q dt.
2. **Modelo continuo** — la serie de Fourier con los coeficientes calculados.
3. **Reglas de integración** — linealidad, constante, ∫cos(ωt)dt = sin(ωt)/ω y
   ∫sin(ωt)dt = −cos(ωt)/ω, justificando la sustitución u = ωt.
4. **Antiderivada completa** Q(t).
5. **Tabla de evaluación término a término**: para cada armónico k se muestra el término, su
   antiderivada Qₖ(t), Qₖ(b), Qₖ(a) y la diferencia.
6. **Regla de Barrow** con los números sustituidos.
7. **Comprobación derivando** — deriva un término de Q para verificar que devuelve q.
8. **Validación con los datos** — compara contra Σqₕ·Δt y reporta el error porcentual.

### 2 · Sumas de Riemann

Aproxima la misma integral con **cuatro métodos** (izquierda, derecha, punto medio, trapecios) y
un número de subintervalos n ajustable de 1 a 96. Hay un botón **▶ Animar** que recorre n de 1 a
60 para ver la convergencia.

Lo interesante no es la aproximación sino el análisis del error:

- Tabla de los n rectángulos: intervalo, punto muestra, altura, área y acumulado.
- **Comparación de los cuatro métodos** con el mismo n: valor, error con signo, si sobreestima o
  subestima, cota teórica, y el n que haría falta para bajar el error de 100 vehículos.
- **Explicación del sesgo a partir de las derivadas**, detectando automáticamente si la función
  es creciente, decreciente o mixta, y si es cóncava hacia arriba o hacia abajo en [a, b]:

  ```
  q′ > 0 ⇒ Lₙ ≤ ∫ ≤ Rₙ        q′ < 0 ⇒ Rₙ ≤ ∫ ≤ Lₙ
  q″ < 0 ⇒ Tₙ ≤ ∫ ≤ Mₙ        q″ > 0 ⇒ Mₙ ≤ ∫ ≤ Tₙ
  ```

- **Cotas del error** con M₁ = máx|q′| y M₂ = máx|q″| medidos en el intervalo:

  ```
  |E_L|, |E_R| ≤ M₁(b−a)²/(2n)      |E_M| ≤ M₂(b−a)³/(24n²)      |E_T| ≤ M₂(b−a)³/(12n²)
  ```

  Con la observación de que izquierda y derecha mejoran como 1/n, mientras punto medio y
  trapecios mejoran como 1/n².

Cierra con una idea que conecta todo: **los datos originales ya son una suma de Riemann** con
Δt = 1 h, porque cada conteo horario es el rectángulo de esa hora.

### 3 · Derivadas

Un deslizable mueve el instante t₀ por el día y la página muestra dos gráficas apiladas: arriba
q(t) con su **recta tangente** en t₀, abajo q′(t) coloreada según su signo.

Calcula y explica:

- q(t₀), q′(t₀) y q″(t₀), con la lectura de la concavidad.
- **Cociente de diferencias** con h = 1; 0,1; 0,01; 0,001, para ver numéricamente cómo se acerca
  a la derivada.
- Reglas de derivación usadas, incluida la cadena para cos(ωt) y sin(ωt).
- **Derivada término a término** en tabla, armónico por armónico.
- **Recta tangente** y = q(t₀) + q′(t₀)(t − t₀), con la aproximación lineal a 15 minutos
  contrastada contra el valor real del modelo.
- **Puntos críticos** q′(t) = 0 clasificados con el criterio de la segunda derivada: hora pico
  (máximo) y hora valle (mínimo).
- **Puntos de inflexión** q″(t) = 0, que son los momentos en que el tráfico deja de acelerarse y
  empieza a frenarse: la subida más rápida y la bajada más rápida del día.
- La relación con la integral por el TFC parte 1: la hora pico es donde el acumulado N(t) sube
  con mayor pendiente.

### 4 · Área entre curvas

Compara **dos corredores** (o el mismo corredor en dos tipos de día). Dibuja ambas curvas y
sombrea la región entre ellas con distinto color según cuál va arriba.

- Encuentra los **puntos de corte** resolviendo A(t) − B(t) = 0 por bisección.
- Permite tres modos de límites: entre los puntos de corte, día completo [0, 24], o manuales.
- **Separa la región en subintervalos** por cada cruce, determina con un punto de prueba qué
  curva va arriba en cada uno, y suma los valores absolutos.
- Distingue explícitamente el **área** (∫|A − B|) de la **diferencia neta** (∫(A − B)), y avisa
  cuando esta última no representa el área porque las regiones se cancelan.
- Tiene una casilla de **normalización**: convierte cada curva a "% del tráfico diario por hora",
  lo que permite comparar la *forma* del día entre un corredor grande y uno pequeño, en vez del
  volumen.

### 5 · Velocidad media

Aquí la función modelada es la **velocidad** v(t) en km/h, no la intensidad.

- Calcula el **valor medio** v̄ = (1/(b−a))·∫ₐᵇ v(t)dt.
- Dibuja el **rectángulo de altura v̄** sobre el intervalo, que tiene exactamente la misma área
  que la región bajo v(t): la interpretación geométrica del valor medio.
- Aplica el **Teorema del Valor Medio para integrales**: como v es continua, existe al menos un
  c ∈ [a, b] con v(c) = v̄, y lo resuelve numéricamente marcando esos instantes en la gráfica.
- Interpreta ∫ₐᵇ v(t)dt como la distancia (en km) que recorrería un vehículo que circulara todo
  el intervalo a esa velocidad.
- Reporta el mínimo y el máximo del intervalo con la hora en que ocurren.

### 6 · Formulario

Una hoja de consulta con 13 tarjetas, sin gráficas ni controles: definición de derivada, reglas
de derivación, derivadas de funciones usuales, aplicaciones de la derivada, integrales
inmediatas, propiedades de la integral definida, Teorema Fundamental del Cálculo, técnicas de
integración, sumas de Riemann, cotas de error, área entre curvas, valor medio y el resumen del
modelo de Fourier que usa la propia página.

### 7 · Metodología 1 — De la tasa a la acumulación

Taller completo con **10 secciones**, cada una organizada en cinco pasos:
*Identificar · Plantear · Resolver · Interpretar · Verificar*.

1. Del cambio a la cantidad
2. Datos del Excel y modelo de la tasa (polinomio por mínimos cuadrados, con R²)
3. Derivada o antiderivada: cuál de las dos resuelve la pregunta
4. Integral indefinida y linealidad
5. **Condición inicial**: se elige la constante C tal que N(a) = 0, es decir, "vehículos contados
   desde la hora a"
6. **Cambio acumulado** entre dos instantes t₁ y t₂ ajustables, N(t₂) − N(t₁)
7. Sumas de Riemann con n subintervalos y el método elegido
8. TFC y cálculo del error frente a la aproximación
9. ¿Aproximación o valor exacto?: cuándo conviene cada uno
10. Punto de control y reflexión

### 8 · Metodología 2 — Técnicas y decisiones

Taller de **7 secciones** centrado en elegir la técnica de integración correcta:

- **R1 · Diagnóstico** — dónde q′ es positiva o negativa: en qué franjas el flujo sube y en
  cuáles baja.
- **R2 · Caso integrador** — comparar dos corredores como si fueran ingreso contra costo: área
  entre curvas, cortes, y en qué tramos gana cada uno.
- **R3 · Sustitución** — integrar el modelo de Fourier del día completo con u = kπt/12.
- **R4 · Identidad trigonométrica** — modela la subida hacia la hora pico como una rampa
  r(t) = q₁ + Δq·sin²(π(t−t₁)/2L) y la integra con **sin²θ = (1 − cos2θ)/2**. El resultado
  exacto es q₁L + ΔqL/2, que después contrasta con los datos.
- **R5 · Integración por partes** — calcula ∫t·q(t)dt con u = t, y de ahí la **hora media del
  tráfico**:

  ```
  t̄ = ∫ₐᵇ t·q(t) dt / ∫ₐᵇ q(t) dt
  ```

  el "centro de masa" temporal de la franja. Se verifica con Simpson.
- **R6 · Decidir antes de calcular** — tabla que asocia cada forma de integrando con su técnica y
  con la señal estructural que la delata (función interna lineal → sustitución; potencia par de
  seno → identidad; polinomio × trigonométrica → por partes; solo datos discretos → Riemann).
- **R7 · Recomendación ejecutiva** — genera un borrador redactado con los resultados numéricos
  del corredor y la franja elegidos, para completar con el análisis del equipo.

### 9 · Verificación

Rehace los cálculos **en Python, dentro del navegador**, y los compara con los que hizo
JavaScript. Es un control cruzado: dos implementaciones independientes del mismo modelo que
tienen que coincidir.

Al pulsar **▶ Ejecutar** se descarga [Pyodide](https://pyodide.org) (Python compilado a
WebAssembly) junto con **numpy**, y se ejecuta una función `analizar(q, K, a, b, n)` escrita en
Python que reconstruye todo desde cero con álgebra matricial:

- los coeficientes de Fourier, con productos matriciales en vez de bucles;
- la integral exacta por el Teorema Fundamental;
- las aproximaciones por trapecios y punto medio;
- **la regla de Simpson**, que no aparece en ninguna otra pestaña (ajusta n al par siguiente si
  hace falta, porque Simpson lo exige);
- el conteo real de los datos, con el solape de cada hora contra el intervalo [a, b];
- el **R²** del modelo y los puntos críticos de q′(t) = 0, localizados por interpolación lineal
  entre cambios de signo sobre una malla de 4.801 puntos.

El resultado es una tabla de cuatro columnas —concepto, valor en JavaScript, valor en Python y
diferencia— más el R², la hora pico y la lista de máximos y mínimos del día. Las diferencias
deben salir prácticamente en cero; si alguna no lo hace, hay un error en alguna de las dos
implementaciones.

Una vez cargado Python, la pestaña **recalcula sola** cada vez que cambias un control, sin volver
a descargar nada.

### 10 · Mapa

Sitúa los corredores sobre un mapa de Medellín con [Leaflet](https://leafletjs.com) y los
colorea según su tráfico, con un deslizador que recorre las 24 horas del día.

- **Variable a mapear**: intensidad q (veh/h) o velocidad v (km/h).
- **Escala de color fija** para todo el día —de verde a rojo pasando por ámbar—, calculada sobre
  el mínimo y el máximo de *todos* los corredores en *todas* las horas. Al mover el deslizador,
  el color cambia porque cambia el tráfico, no porque se haya reajustado la escala. Esto es lo
  que permite comparar horas entre sí.
- **Clic en un corredor**: abre un globo con su intensidad y su velocidad a esa hora, y carga su
  curva del día en la gráfica de al lado.
- **Mapa base** elegible entre Esri Calles y OpenTopoMap. Esri va primero porque OpenStreetMap
  bloquea sus mosaicos cuando la página se abre como archivo local.
- **🗺 Cargar trazado real**: por defecto cada corredor es un punto aproximado. Este botón
  consulta la API **Overpass** de OpenStreetMap y reemplaza los puntos por el trazado real de
  las vías, probando variantes del nombre (*Calle 30*, *Avenida 30*…) y con un segundo servidor
  de reserva si el primero falla.

Al lado del mapa, las tarjetas muestran el valor del corredor a esa hora, **su posición en el
ranking** de todos los corredores en ese momento, y el promedio general.

---

## El Excel que genera

En las pestañas 7 y 8 aparece el botón **⬇ Descargar Excel de metodologías**. Produce un archivo
llamado `metodologias_resueltas_<corredor>_<grupo>.xlsx` con **13 hojas**.

Lo importante: las celdas no traen números pegados, traen **fórmulas vivas de Excel**. Si cambias
un dato, todo se recalcula. Y la primera hoja compara el valor que mostró la página contra el que
recalcula Excel, para que la diferencia se vea y sea ≈ 0.

| Hoja | Contenido |
|---|---|
| **Instrucciones** | Portada: fuente, corredores, ventana, parámetros e índice de hojas |
| **Resumen comparativo** | Valor de la página vs. valor recalculado en Excel, con la columna Diferencia |
| **Datos del Excel** | Perfil horario: q en veh/h y v en km/h de ambos corredores |
| **Mínimos cuadrados** | Coeficientes del polinomio, residuos y R² con fórmula `DEVSQ` |
| **Antiderivada y cond. inicial** | F(t), la constante C con N(a) = 0, tabla de N(t) y cambio acumulado |
| **Sumas de Riemann** | Partición, tabla de subintervalos y las cuatro sumas con fórmulas; error contra el TFC |
| **Riemann con datos** | Suma de los conteos horarios dentro de [a, b], que es Riemann con Δt = 1 h |
| **Criterio de la derivada** | q′(t), discriminante, ceros, clasificación con q″ y signo hora por hora |
| **Área entre curvas** | Integrales de A y B, diferencia neta y área total |
| **Sustitución** | Coeficientes de Fourier con `SUMPRODUCT` y aporte de cada término |
| **Identidad trigonométrica** | La integral de la rampa con sin²θ = (1 − cos2θ)/2 |
| **Integración por partes** | ∫t·q(t)dt desarrollado y la hora media |
| **Perfiles de corredores** | Perfiles horarios de los 44 corredores, para comparar |

---

## Cómo está hecho por dentro

Un solo archivo HTML de unas 1.350 líneas, sin frameworks y sin nada que instalar:

- **Líneas 7–94** — estilos CSS, con variables de color y soporte automático de **modo claro y
  oscuro** según la configuración del sistema.
- **Líneas 96–212** — estructura de la página: cabecera, las diez pestañas, el panel de controles
  (que se muestra u oculta según la pestaña activa), el lienzo de la gráfica y el pie con la nota
  metodológica.
- **Línea 214** — la constante `DATA` con los 44 corredores. Es la línea larguísima del archivo:
  ahí están los 44 × 11 perfiles horarios.
- **Líneas 215–1346** — el JavaScript, organizado en bloques comentados: modelo de Fourier,
  formato de números, interfaz, dibujo en canvas, una función por pestaña, ajuste polinómico, el
  cálculo común de los talleres, el generador del Excel, el código Python y el mapa.

Las cuatro librerías externas (SheetJS, Pyodide, numpy y Leaflet) **no están incrustadas**: se
descargan solo cuando pulsas el botón que las necesita. Por eso el archivo pesa 274 KB y no
varios megas, y por eso las ocho primeras pestañas funcionan sin conexión.

Las gráficas se dibujan a mano sobre un elemento `<canvas>`, con reescalado según la densidad de
pantalla. Al pasar el cursor por encima aparece un recuadro con los valores en ese instante, y lo
que muestra depende de la pestaña: en la 1, q(t) y el acumulado ∫₀ᵗq; en la 3, q, q′ y q″; en la
4, las dos curvas comparadas.

Los números se formatean en convención colombiana (`es-CO`): punto para los miles y coma para los
decimales.

### Editar el proyecto

- **Cambiar los datos**: reemplazar la constante `DATA` de la línea 214, respetando la estructura
  `{ corredor: { grupo: { q: [24], v: [24], dias, carriles } } }`.
- **Cambiar el periodo que se muestra**: la constante `PERIODO`, justo debajo.
- **Cambiar los valores iniciales**: el objeto `AB` fija la ventana [a, b] por defecto de cada
  pestaña, y unas líneas más arriba se fijan el corredor y el tipo de día iniciales.
- **Añadir un corredor al mapa**: agregar su nombre y sus coordenadas a `GEO_APROX`, con el
  nombre escrito exactamente igual que en `DATA`.
- **Cambiar los cálculos de Python**: la constante `PY_CODE` contiene el código fuente completo
  como texto; se edita ahí mismo, sin archivos aparte.

---

## Detalles conocidos

- **Un nombre de corredor sale mal escrito.** En la lista aparece `Vï¿½ï¿½ï¿½a al Tï¿½ï¿½ï¿½nel
  de Occidente`; debería decir *Vía al Túnel de Occidente*. Es un problema de codificación que
  viene arrastrado del archivo de datos original, donde las tildes se perdieron antes de generar
  el HTML. No afecta a los cálculos, solo al texto del menú.
- **Los conteos no son vehículos únicos.** Como se suman todos los puntos de medición del
  corredor, un mismo vehículo que recorre varios puntos aparece varias veces. Las cifras sirven
  para comparar franjas horarias y corredores entre sí, no como censo de vehículos distintos.
- **Los datos son de julio y agosto de 2020**, en plena pandemia. Los volúmenes son más bajos que
  los de un año normal, aunque la forma del día (los picos de la mañana y la tarde) se conserva.
- **El mapa muestra 21 de los 44 corredores.** Solo esos tienen coordenadas en la tabla
  `GEO_APROX`; el resto tiene datos y aparece en las demás pestañas, pero no en el mapa. Para
  añadir uno basta con agregar su nombre y su par latitud/longitud a esa tabla.
- **Las ubicaciones del mapa son aproximadas**: un punto por corredor, no su trazado. El botón
  *Cargar trazado real* las sustituye por las líneas de OpenStreetMap, pero depende de un
  servicio público (Overpass) que a veces va lento o rechaza la consulta. Si falla, el mapa se
  queda con los puntos aproximados y lo avisa.
- **Tres funciones necesitan internet la primera vez** de cada sesión: el Excel, la pestaña de
  Verificación y la del Mapa. Las librerías se descargan en ese momento en vez de venir
  incrustadas, que es lo que mantiene el archivo en 274 KB.
