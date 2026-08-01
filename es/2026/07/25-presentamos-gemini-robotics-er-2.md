---
title: "Presentamos Gemini Robotics ER 2"
summary: "Google AI Studio presenta Gemini Robotics ER 2, un salto en comprensión de video, orquestación de tareas y colaboración multi-robot que acerca un razonamiento espacial más rápido y robots más capaces en el mundo físico."
date: "2026-07-31"
link: "https://x.com/GoogleAIStudio/status/2082846204262531310"
author: "Google AI Studio"
---
Presentamos Gemini Robotics ER 2
por Google AI Studio, publicado el 30 de Julio de 2026

Gemini Robotics ER 2 representa un salto cualitativo en la capacidad de potenciar robots con comprensión de video, orquestación de tareas y colaboración multi-robot, haciendo posible que los robots sean más útiles en el mundo físico.
Para que los robots ayuden a los humanos en entornos cotidianos, el razonamiento espacial preciso no basta. Los robots también deben pensar rápido, sincronizando sus decisiones y su razonamiento con la velocidad en tiempo real del mundo físico.
Por eso hoy lanzamos Gemini Robotics ER 2, nuestro modelo de "razonamiento incorporado" más capaz para robótica. Piensen en Gemini Robotics ER 2 como un cerebro de alto nivel para robots. Permite a los robots conversar con humanos, entender el mundo físico y planificar tareas de múltiples pasos. Luego delega la ejecución motora a cualquier modelo de visión-lenguaje-acción (VLA) de nivel inferior. Gemini Robotics ER 2 también puede invocar de forma nativa herramientas como Google Search para buscar información, o cualquier otra función definida por el usuario. El diseño de Gemini Robotics ER 2 permite que el robot "piense" en lo que viene después mientras realiza sus acciones de forma simultánea.
Gemini Robotics ER 2 representa una mejora significativa respecto a Gemini Robotics ER 1.6. Al observar flujos de video continuos, los robots pueden ahora seguir su propio progreso, adaptarse si algo sale mal y saber exactamente cuándo pasar al siguiente paso. También presentamos la colaboración multi-robot, que permite a los robots trabajar juntos en espacios compartidos y completar flujos de trabajo complejos que un solo robot no podría hacer solo.
Gemini Robotics ER 2 ya está disponible públicamente para desarrolladores a través de la API de Gemini, Google AI Studio y en vista previa privada en Gemini Enterprise Agent Platform. Para ayudarles a empezar, compartimos ejemplos de cómo configurar el modelo y solicitarle que potencie tareas de IA física más útiles.
Avances en capacidades agénticas físicas
La mayoría de las tareas en el mundo físico son complejas y requieren múltiples pasos para completarse. Gemini Robotics ER 2 es un agente físico que orquesta los pasos del robot y le permite autocorregirse y generalizar a situaciones más novedosas. Para construir un montaje agéntico, los desarrolladores pueden declarar interfaces de control de bajo nivel —como modelos de visión-lenguaje-acción (VLA) o APIs de navegación— como herramientas, y transmitir video multimodal, audio o texto directamente al modelo.
Gemini Robotics ER 2 mejora este flujo de orquestación de herramientas. Podemos evaluar su rendimiento con robots en simulación, usando control de robots en el mundo real e incluso emparejarlo con un humano que controla el robot de forma remota.
 
En robótica, el razonamiento de alto nivel depende de la velocidad de ejecución. Gemini Robotics ER 2 se integra en la API Gemini Live, usando un endpoint de transmisión bidireccional optimizado para tareas sensibles a la latencia. El resultado es una orquestación fluida: Gemini Robotics ER 2 comanda modelos de acción y APIs de robótica para completar tareas de múltiples pasos sin las bruscas pausas de "parar y pensar".

Para ilustrarlo, hemos construido una demo con Spot de nuestros socios en Boston Dynamics. Usamos Gemini Robotics ER 2 para orquestar las APIs de Spot, como navegación y movimiento del manipulador, creando un robot interactivo que busca objetos por ti.
 
El código está disponible en Github junto con otros ejemplos.
Desbloqueando la inteligencia temporal para una finalización robusta de tareas
Uno de los mayores desafíos de la robótica es saber cuándo una tarea está hecha. Gemini Robotics ER 2 aporta un salto en comprensión de video y seguimiento del progreso para verificar que tareas complejas —como apretar una bombilla o atar una bolsa de basura— se completen según las especificaciones antes de pasar a la siguiente tarea.
En esta actualización, hemos avanzado en dos capacidades fundamentales para la comprensión del progreso de las tareas: clasificación del progreso y localización de momentos.
Clasificación continua del progreso
La clasificación del progreso se refiere a la capacidad de un robot de seguir el avance hacia la finalización de una tarea. En nuestras evaluaciones, asignamos cada fotograma de un flujo de video a cinco niveles de progreso (0-20%, 20-40%, 40-60%, 60-80%, 80-100%). Al cuantificar el progreso de la tarea, Gemini Robotics ER 2 proporciona a los robots conciencia situacional en tiempo real y les permite ajustar acciones sobre la marcha o reintentar pasos fallidos sin reiniciar todo el flujo de trabajo.
 
Localización precisa de momentos
La localización de momentos mide la capacidad de un modelo de identificar el fotograma exacto del video donde ocurre un evento crítico (es decir, cuándo dejar de verter café en una taza). Gemini Robotics ER 2 logra avances significativos en rendimiento en localización de momentos, permitiendo a los robots cambiar de tarea con precisión, verificar el éxito y sugerir correcciones.
 
Colaboración multi-robot
Ningún robot individual se adapta a todas las tareas: un rover con ruedas destaca en interiores, mientras que un robot humanoide puede sobresalir en terreno irregular. Gemini Robotics ER 2 habilita la colaboración multi-robot, permitiendo que máquinas diversas se comuniquen mediante una comprensión semántica compartida para transferir y completar tareas complejas. Vean cómo Gemini Robotics ER 2 permite que Apollo 2 de Apptronik y Franka F3 Duo colaboren aquí.
Mejorando la inteligencia espacial general
Gemini Robotics ER 2 avanza nuestra capacidad central de razonamiento espacial, medida por tres puntos de referencia:
Detección de éxito o fracaso: ahora opera sobre flujos de video en bruto en lugar de capturas estáticas para detectar fallos a mitad de ejecución como derrames, resbalones o desalineaciones.
Lectura general de instrumentos: se extiende más allá de diales circulares y visores para incluir pantallas digitales, escalas lineales, reglas y termómetros de líquido. Lo probamos en 10 tipos diferentes de instrumentos.
VQA espacial mejorada: mejora la respuesta visual a preguntas mediante los avances de Gemini en comprensión multimodal.
 
Avanzando en seguridad para la inteligencia incorporada
Gemini Robotics ER 2 es nuestro modelo más seguro, con avances significativos en los puntos de referencia de Seguimiento de Instrucciones de Seguridad y Proximidad Humana, que evalúan cómo un modelo se adhiere a restricciones físicas durante tareas de razonamiento y la conciencia espacial para detectar humanos. Descubrimos que Gemini Robotics ER 2 detiene con éxito un robot humanoide cuando hay una persona cerca y reanuda el trabajo de forma autónoma solo cuando la zona está despejada. Para avanzar en la seguridad de los agentes físicos, presentamos un punto de referencia que evalúa la capacidad de un modelo fundacional de actuar como orquestador VLA seguro, poniendo a prueba su capacidad de imponer restricciones de seguridad, monitorear el entorno, evaluar la viabilidad física y solicitar aclaraciones humanas. Para más detalles, consulten nuestro informe técnico de seguridad.
 
De cara al futuro, nuestros planes son impulsar estos modelos hacia tareas aún más complejas para acelerar el desarrollo de robots útiles y apoyar a la comunidad de robótica.
