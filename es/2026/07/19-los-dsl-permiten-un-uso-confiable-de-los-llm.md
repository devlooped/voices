---
title: "Los DSL permiten un uso confiable de los LLM"
summary: "Unmesh Joshi sostiene que los lenguajes específicos de dominio y los modelos semánticos compartidos dan a los LLM límites claros para generar código de forma fiable, ilustrado con herramientas de presentaciones PlantUML y el framework Tickloom para sistemas distribuidos."
date: "2026-07-14"
link: "https://martinfowler.com/articles/llm-and-dsls.html"
author: "Unmesh Joshi"
---
Los DSL permiten un uso confiable de los LLM
por Unmesh Joshi, publicado el 14 de Julio de 2026

Los modelos de lenguaje grandes modernos poseen una capacidad increíble. Pueden generar grandes cantidades de código, y a veces sistemas enteros, a partir de una descripción de alto nivel en lenguaje natural. Una suposición importante es que la intención de lo que hay que construir está bien articulada, con palabras precisas que los LLM puedan mapear a bloques de construcción de código. Sin embargo, hay dos puntos importantes a tener en cuenta: los límites de la especificación previa y cómo el diseño se descubre mediante la implementación.

Los límites de la especificación previa

Construir sistemas grandes implica muchísimas decisiones de diseño pequeñas, y no todas pueden conocerse de antemano ni impulsarse por completo desde una especificación de alto nivel. Una especificación es, en el mejor de los casos, una hipótesis inicial: las restricciones reales, las compensaciones y los casos límite se descubren de forma iterativa, a medida que avanzamos con la implementación. Lo discutimos en detalle en un artículo anterior, al que llamamos la imposibilidad de la especificación previa. El punto no es que las especificaciones no sirvan, sino que la primera es una hipótesis a revisar, nunca un plano terminado.

La respuesta natural es iterar: refinar la especificación, generar código, revisar lo que vuelve y alimentar lo aprendido en la siguiente ronda. Ese bucle funciona bien cuando cada ronda produce un cambio pequeño y revisable.

El diseño se descubre mediante la implementación

Revisar código, en particular mientras aún estamos descubriendo el diseño, no es lo mismo que escribirlo. Al revisar el código generado, revisamos por fragmentos validando si se ajusta a nuestra intención y buscando posibles trampas. Pero la revisión rara vez nos obliga a pelear de verdad con las decisiones de diseño. Escribir código, en cambio, nos fuerza a pensar decisiones concretas—como dónde pertenece una responsabilidad o qué límites deben exponerse para poder extender el diseño. Es al tomar esas decisiones cuando un diseño se revela por completo.

El código tiene dos propósitos distintos pero entrelazados. Es un conjunto de instrucciones para una máquina, y también un modelo conceptual del dominio del problema. Una base de código bien diseñada es una representación del vocabulario de un dominio. Esas abstracciones solo se revelan a medida que los desarrolladores construyen el software. Los lenguajes de programación actúan como herramientas de pensamiento, permitiendo construir un modelo conceptual que sostiene la evolución posterior. Con los LLM, el código actúa como contexto esencial: buenas abstracciones, comportamiento ejecutable, pruebas, tipos e invariantes ayudan a acotar el modelo y a hacer su salida más útil.

El lenguaje de programación y el paradigma en el que codificamos dan forma a la visión de diseño que obtenemos. Un enfoque funcional o uno orientado a objetos revelan aspectos distintos del diseño, junto con idiomas y patrones naturales de cada paradigma.

Entonces, ¿dónde encajan los LLM? Yo veo que los LLM cumplen dos roles. Son de gran ayuda mientras damos forma al diseño y a su vocabulario, actuando como socios de brainstorming para explorar el espacio de diseño y descubrir las abstracciones correctas. Una vez establecido el vocabulario, los LLM funcionan como una excelente interfaz de lenguaje natural hacia él.

Abstracciones de dominio y DSL

Una forma útil de enmarcar esto es a través del diseño dirigido por el dominio. Su idea central es construir un modelo conceptual compartido del dominio en código y luego usar ese modelo—que el DDD llama lenguaje ubicuo—tanto para evolucionar la base de código como para dar al equipo un vocabulario con el que pensar y comunicarse. A menudo es muy efectivo construir un lenguaje específico de dominio sobre ese modelo: una sintaxis acotada para expresar los conceptos y operaciones del dominio. Visto así, la mayor parte del desarrollo es el proceso de construir un modelo de dominio y usarlo para evolucionar el sistema. El LLM juega dos roles distintos según exista o no ya el modelo de dominio. En este artículo me centraré en cómo los lenguajes específicos de dominio, o DSL, funcionan con los LLM.

Por qué los DSL funcionan tan bien con los LLM

Es una experiencia común que los DSL funcionan bien con los LLM. PlantUML, Mermaid y Graphviz son lenguajes específicos de dominio para modelado visual; SQL es un DSL para consultar bases de datos; el YAML de Kubernetes es un DSL para describir infraestructura en la nube. No son lenguajes de propósito general: están deliberadamente acotados, diseñados para expresar un conjunto estrecho de conceptos en un dominio. Y no es de sorprender que los LLM sean notablemente buenos generando diagramas Mermaid, consultas SQL o manifiestos de Kubernetes a partir de una descripción en lenguaje cotidiano.

Mi observación es que los DSL hacen más fiables a los LLM porque responden muy bien a unos pocos ejemplos en contexto. Un lenguaje de propósito general como Java ofrece muchas formas válidas de expresar la misma intención. Un DSL elimina esa variación. Darle al modelo unos pocos ejemplos basta para generar de forma fiable la sintaxis correcta. Vale la pena notar que los modelos de primera línea ya están muy expuestos a PlantUML o a interfaces fluidas de Java durante el entrenamiento, así que no parten de cero. Será curioso ver cómo se comportan modelos más pequeños y acotados cuando se les pide un DSL verdaderamente nuevo.

Para un agente—un LLM que corre en un bucle autónomo de generar y comprobar, no en una sola generación—hay un beneficio más. Un DSL casi siempre trae un validador determinista: un parser, un esquema JSON, un verificador de tipos o un compilador. El agente puede generar un candidato, pasarlo por el validador y repararlo a partir del error, todo sin un humano en el bucle. Crucialmente, los errores se formulan al nivel del dominio—por ejemplo, no puedes seleccionar una acción antes de elegir un cliente—y no como un stack trace enterrado en el código generado. El propio conjunto de herramientas del DSL actúa como un excelente arnés.

Es importante señalar que esto no es una solución única para todos. La ventaja se sostiene mientras el DSL se mantenga lo bastante pequeño y acotado para que unos pocos ejemplos en contexto transmitan su uso. También hay un costo real de diseñar y mantener el lenguaje y su modelo semántico. Por eso el retorno se concentra en DSL bien factorizados, genuinamente acotados y respaldados por un validador.

Ejemplo: usar LLM para generar presentaciones de PowerPoint ricas en diagramas

Los LLM facilitan mucho construir herramientas a medida. Al enseñar sistemas distribuidos, con frecuencia necesito crear presentaciones que en su mayoría tienen diagramas que explican operaciones distribuidas en un clúster. Los diagramas de secuencia UML han sido excelentes para eso, pero mostrar un diagrama de secuencia completo mientras se explica el flujo de mensajes no es muy útil. Necesitaba una herramienta para mostrar un diagrama de secuencia paso a paso en una presentación de PowerPoint. Con la ayuda de LLM pude construir una herramienta que procesa una descripción YAML de la estructura de la presentación con referencias a diagramas PlantUML y genera la presentación. Los diagramas PlantUML se marcan con pasos y la herramienta genera una diapositiva separada por cada paso. Eso facilitó mucho crear presentaciones ricas en diagramas.

Un prompt en lenguaje natural puede generar diagramas de secuencia PlantUML con marcadores de paso para un clúster de tres nodos—Atenas, Bizancio y Cirene—mostrando el flujo de mensajes, fallos y comprobaciones de quórum. Un segundo DSL más pequeño en YAML describe qué diagrama va en cada diapositiva. Como la herramienta y la especificación YAML que entiende se usan como contexto en el prompt, el LLM genera el YAML correcto que la herramienta puede consumir directamente.

Nótese que el LLM jugó dos papeles distintos en este único ejemplo. Primero fue un co-diseñador—ayudando a dar forma a la extensión PlantUML con pasos y al YAML de diapositivas sobre el tooling existente. Luego, una vez que existía ese pequeño DSL, se convirtió en la interfaz de lenguaje natural que transforma una petición en inglés en una especificación válida.

Construyendo el modelo semántico

En el ejemplo de presentaciones, el YAML se usó como sintaxis portadora, y proceso su árbol de sintaxis parseado de forma directa, usando en la práctica el propio árbol como mi modelo semántico—aunque eso acopla la sintaxis a la semántica de ejecución. Pero en dominios más complejos, como los sistemas distribuidos, necesitamos modelos semánticos más ricos para representar los conceptos del dominio y las decisiones de diseño del código.

Ejemplo: Tickloom—un modelo semántico para sistemas distribuidos

Implementar sistemas distribuidos como almacenes clave-valor basados en quórum o protocolos de consenso como Raft y Paxos es una tarea formidable. Incluso si la implementación se guía de forma incremental con prompts, especificaciones o archivos de skill cuidadosamente construidos, los runtimes asíncronos siguen exponiendo un espacio abrumador de posibles decisiones de implementación. Modelos de hilos, patrones de red, coordinación de almacenamiento, reintentos y semántica temporal quedan entrelazados en el código generado. El problema no es solo la complejidad de generar código, sino la de verificarlo. El espacio de estados resultante de todos los entrelazados posibles entre planificación de hilos, retrasos de red, pausas de proceso y desfase de reloj se vuelve tan grande que revisar y validar sistemáticamente la corrección es casi imposible. Por eso las pruebas Jepsen encuentran errores incluso en los sistemas distribuidos más probados en batalla.

Ahí es exactamente donde un modelo semántico es beneficioso. Tickloom es un pequeño framework que construí para construir y probar algoritmos distribuidos. Sus abstracciones no son un runtime genérico; son un conjunto de decisiones de diseño sobre cómo se comporta un proceso distribuido. Cada nodo corre en un bucle de tick de un solo hilo: cada llamada a tick avanza un reloj lógico en uno y procesa el trabajo pendiente en un orden fijo y determinista—red, luego bus de mensajes, luego proceso, luego almacenamiento. El tiempo se mide en ticks, no en milisegundos. Los mensajes son records simples de Java. La coordinación entre réplicas se expresa mediante una clase base Replica que ya conoce pares, broadcasts y quórums.

El threading, el tiempo y la entrega de red ya no son preguntas abiertas a redecidir en cada prompt. Lo que queda para el autor del algoritmo es la lógica real del protocolo. Una réplica de quórum, por ejemplo, es solo un conjunto de manejadores de mensajes expresados en el vocabulario del framework.

Como el framework aporta el vocabulario—Replica, petición de quórum, contar respuesta si, tipo de mensaje, manejador—un prompt puede quedarse al nivel del protocolo en lugar de la plomería. Por ejemplo: usando la abstracción Replica de Tickloom, implementa un almacén clave-valor basado en quórum donde un get de cliente recolecta valores de una mayoría y devuelve el de mayor timestamp, última escritura gana, y una escritura se aplica en local solo si su timestamp es más nuevo que el almacenado. El código generado rellena manejadores de protocolo contra esos tipos fijos. El propio modelo semántico actúa como contexto. El prompt nombra conceptos que existen como tipos concretos en el código, así que el LLM no inventa un modelo de hilos ni una capa de red: completa la lógica del protocolo sobre un sustrato fijo y bien comprendido.

Incluso las buenas abstracciones ayudan—sin un DSL

Un DSL es un extremo del espectro, y no es fácil de construir. Antes de lanzarse a un lenguaje propio conviene notar que un conjunto limpio de abstracciones ya es una versión más ligera de la misma idea—y, igual que el framework aportó el vocabulario del almacén de quórum, los tipos y métodos con nombre de una biblioteca son en sí un vocabulario en el que el modelo puede anclarse. El modelo semántico de Tickloom es en realidad solo cuatro costuras de ese tipo—Process y Replica para cómputo y mensajes, Network para comunicación, Storage para persistencia y un Clock lógico para el tiempo—y esa descomposición hace la mayor parte del trabajo sin sintaxis nueva.

Por eso las abstracciones, no solo los DSL, combinan bien con los LLM. Un prompt para implementar Raft como una Replica de Tickloom tiene un espacio de estados limitado que explorar. La QuorumReplica existente puede usarse en contexto como ejemplo trabajado.

Ejemplo: construir un DSL para probar escenarios de sistemas distribuidos

Implementar el algoritmo es una cosa; ejercitarlo es otra. Los bugs sutiles de los sistemas distribuidos viven en ordenamientos específicos: una escritura que se replica a un nodo antes de que el quórum de un lector cambie, una partición que se cura en el peor momento, dos coordinadores cuyos relojes se han desfasado. Escribir ese escenario directamente contra el kit de pruebas implica hacer malabares con futures y bucles manuales de tick. La intención—Bob escribe a través de Bizancio, Alice a través de Atenas, un lector ve el valor de Bob porque el reloj de Bizancio iba adelantado—queda enterrada bajo la mecánica. Ese estilo de código también es difícil de verificar: hay docenas de decisiones incidentales en las que un LLM puede equivocarse de forma sutil, y un revisor debe comprobar cada una.

Así que sobre el modelo semántico construí un DSL interno cuyo vocabulario es el del propio escenario—servidores, clientes, quién está conectado a quién, qué hace cada cliente y qué fallos están activos mientras lo hace. El mismo escenario de desfase de reloj se vuelve una descripción declarativa breve: servidores Atenas, Bizancio y Cirene; clientes Alice, Bob y un lector; tiempos de servidor distintos; Bob escribe B, Alice escribe A y el lector espera B.

El DSL es una superficie delgada y declarativa que compila a una representación intermedia pura—un Scenario hecho de Steps, donde cada paso lleva una Action, como una lectura o escritura, y eventos de clúster opcionales, como particiones y retrasos de mensajes. Los fallos también se leen como inglés: particionar Bizancio de Cirene, reconectar Bizancio, retrasar una petición interna de set de Atenas a los otros nodos cien ticks. La gramática la impone el sistema de tipos mediante interfaces progresivas—no puedes declarar un paso antes de la topología, ni una acción antes de elegir un cliente—así que clases enteras de escenarios mal formados simplemente no compilan. Como el DSL es interno en Java, el compilador anfitrión valida la gramática gratis, y una generación malformada vuelve como un error de compilación anclado exactamente al paso ilegal, no como una sorpresa en runtime.

Una vez existe el DSL, una descripción en lenguaje natural de un escenario de fallo se mapea casi directamente. Un prompt puede pedir un escenario que reproduzca una lectura de quórum no linealizable del apartado 10.6 de Designing Data-Intensive Applications: un escritor conectado a Atenas fija una clave, luego la actualiza mientras la replicación se retrasa; Alice leyendo a través de Bizancio se ve forzada a un quórum que incluye Atenas y ve el valor nuevo; Bob leyendo después a través de un quórum de Bizancio y Cirene aún ve el valor viejo. El LLM produce un escenario que se queda por completo dentro del vocabulario acotado del DSL.

Como la superficie es tan pequeña—y el espacio de código válido que puede generar es mucho más pequeño que el de programas Java válidos—el LLM tiene muy poco margen para alucinar, y un revisor puede leer el resultado como la descripción de un experimento y no como código a auditar línea a línea. Incluso si el LLM alucina, el DSL interno fallará al compilar, permitiendo que el LLM corrija los errores.

Dos fases de trabajo con LLM

De los ejemplos anteriores emerge un patrón: en todos ellos el LLM fue útil de dos maneras bastante distintas.

La primera fase es diseñar la abstracción o el propio DSL. Aquí al LLM conviene tratarlo como socio de brainstorming más que como generador de código. Como se argumentó al inicio, las decisiones de diseño de un modelo semántico no pueden especificarse todas de antemano: descubrimos restricciones, compensaciones y casos límite al implementarlas. Así que esta fase es inherentemente iterativa y guiada por feedback: propones una estructura, la pruebas contra un caso real, ves dónde resulta incómoda y devuelves lo aprendido a la siguiente ronda. El LLM acelera ese bucle—esboza alternativas, critica un diseño, porta una idea de un lenguaje a otro—pero tú te mantienes firmemente al volante, porque esas son exactamente las decisiones que necesitas entender y poseer. Las estructuras que hacen agradable un DSL, como interfaces progresivas que hacen fallar al compilar un escenario ilegal o un modelo semántico separado del builder que lo produce, se alcanzan iterando, no escribiendo una especificación y generando código.

La segunda fase empieza una vez la abstracción o el DSL está en su lugar. Entonces lo que hace el LLM cambia: se convierte en una interfaz de lenguaje natural hacia lo que has construido. Los prompts de este artículo son ejemplos—implementa un almacén de quórum como Replica de Tickloom, escribe un escenario que reproduzca la lectura no linealizable, crea un YAML de diapositiva para este diagrama. En cada caso la descripción en inglés se mapea casi directamente al vocabulario que definiste, y el LLM es un generador confiable precisamente porque la abstracción aporta tanto el contexto que ancla el prompt como el arnés que comprueba el resultado.

El DSL como fuente de verdad

Hay una tendencia creciente a tratar los prompts como la fuente primaria de verdad. Un DSL bien diseñado cambia fundamentalmente esa dinámica. Una de las ventajas clave que observo al trabajar con DSL es que el propio programa generado a menudo se convierte en el artefacto que los humanos mantienen. Como un DSL es denso, expresivo y en gran medida libre de boilerplate incidental, captura la intención esencial de la solución de una forma que sigue siendo legible mucho después de la generación. Si un LLM genera un escenario de fallo de Tickloom a partir de una petición en lenguaje natural, el escenario resultante ya está expresado en el vocabulario del dominio. Si el mes que viene hay que cambiarlo, no hace falta recuperar el prompt original y regenerarlo todo. El DSL tiene suficiente contexto para que el LLM conozca la intención y trabaje con él. El activo perdurable no es el prompt, sino el DSL y el modelo semántico.

Agradecimientos

Quiero agradecer a Martin Fowler y a Rebecca Parsons por sus valiosos comentarios y sugerencias.

