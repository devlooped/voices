---
title: "Acusan a X de manipular el algo. El código es público. Aún no lo han leído."
summary: "Brivael Le Pogam sostiene que las acusaciones de manipulación algorítmica en X se derrumban ante el código de recomendación abierto: el pánico real es la pérdida del monopolio del relato, no la opacidad."
date: "2026-07-22"
link: "https://x.com/brivael/status/2078940564792426577"
author: "Brivael Le Pogam"
---
Acusan a X de manipular el algo. El código es público. Aún no lo han leído.
por Brivael Le Pogam, publicado el 19 de Julio de 2026

Sobre lo que el código fuente de X dice de verdad — y sobre el pánico que desencadena

El 15 de julio de 2026, tres cosas ocurrieron el mismo día.

Elon Musk comentó sondeos que situaban a Marine Le Pen a la cabeza y la designó como «la última esperanza del país». La izquierda francesa gritó de inmediato injerencia extranjera. Y, ese mismo día, Musk anunció que haría de código abierto la totalidad del código de X — «sin excepción» — invitando a auditores externos a verificar que el código publicado es exactamente el que corre en producción.

Tómese un segundo para medir la escena. Se acusa a un hombre de manipular en secreto una máquina. El mismo día, el hombre cuelga los planos de la máquina en la pared y propone que cualquiera venga a comprobar que no hay doble fondo.

No es una coincidencia incómoda. Es el resumen de todo el asunto.

Lo que hace realmente el algoritmo

Empecemos por el único terreno que no miente: el código. El repositorio xai-org/x-algorithm es público, bajo licencia Apache 2.0, escrito en un 57 por ciento en Rust y un 42 por ciento en Python, actualizado — según el compromiso asumido por X en enero de 2026 — cada cuatro semanas. He aquí, sin jerga, cómo fabrica su hilo «Para ti».

Un director de orquesta, llamado Home Mixer, recibe su petición. Va a buscar candidatos — publicaciones susceptibles de interesarle — en dos reservorios.

El primero, Thunder, contiene las publicaciones de las cuentas que usted sigue. Vive en memoria, se actualiza en tiempo real y responde en una fracción de milisegundo. Es su «in-network».

El segundo, Phoenix Retrieval, pesca en el corpus mundial de publicaciones que usted no sigue pero que podrían hablarle. Técnicamente, es un modelo «de dos torres»: una torre le transforma a usted (su historial de interacción) en una huella digital, la otra transforma cada publicación en una huella, y el sistema acerca lo que se parece. Es su «out-of-network», la puerta al descubrimiento.

Los candidatos de ambos reservorios se enriquecen después (texto, medio, autor, estado de verificación) y se pasan por un tamiz de filtros: se retiran los duplicados, las publicaciones demasiado viejas, sus propias publicaciones, las cuentas que ha bloqueado o silenciado, las palabras clave que ha silenciado, lo que ya ha visto.

Llega entonces el corazón del reactor: Phoenix, un transformer basado en Grok. Para cada publicación superviviente, no calcula una vaga «relevancia». Predice una quincena de probabilidades distintas: probabilidad de que le guste, de que responda, de que republique, de que cite, de que haga clic, de que se detenga en ella, de que siga al autor… y también la probabilidad de que la marque como «no me interesa», de que bloquee, silencie o denuncie.

La puntuación final es una simple suma ponderada de esas probabilidades. Las acciones positivas cuentan positivamente; las negativas — bloquear, silenciar, denunciar — cuentan negativamente y hacen bajar el contenido que detestaría. Se ordena por puntuación, se conserva lo de arriba, se aplica un último filtro de seguridad (contenido eliminado, spam, violencia, gore) y se le sirve el hilo.

Eso es todo. No es magia ni una conspiración. Es una máquina para predecir lo que usted va a hacer, entrenada con lo que ya ha hecho.

El punto que nadie quiere mirar de frente: «No Hand-Engineered Features»

En las decisiones de diseño del repositorio, la primera de todas, numerada 1, se resume en cuatro palabras: No Hand-Engineered Features. Ninguna característica fabricada a mano.

Hay que entender por qué es explosivo.

Históricamente, un algoritmo de recomendación es una acumulación de reglas humanas. «Impulsa las cuentas verificadas.» «Penaliza los enlaces externos.» «Degrada este tipo de contenido.» «Amplifica aquel otro.» Cada una de esas reglas es una válvula. Y cada válvula es un lugar donde una mano — la de un ingeniero, un jefe, un regulador, un ideólogo — puede apretar en silencio. Es exactamente ahí donde vive la manipulación cuando existe: en las heurísticas ocultas.

Lo que dice el repositorio, negro sobre blanco, es que han eliminado esas reglas de relevancia. La documentación es inequívoca: «hemos eliminado cualquier característica fabricada a mano y la mayoría de las heurísticas del sistema». La relevancia ya no la decide un comité. Se aprende, de extremo a extremo, a partir de su secuencia de interacciones por el transformer.

Dicho de otro modo: el lugar donde tradicionalmente se sospecha un dedo en la balanza ha sido desmantelado. No ocultado — desmantelado, y el anuncio de su desmantelamiento está publicado.

Seamos honestos del todo, porque es ahí donde el argumento se vuelve irrefutable en lugar de ingenuo: sí, quedan filtros. Seguridad, spam, violencia, un servicio de análisis de contenido llamado Grox que aplica las reglas de la plataforma. Toda plataforma los tiene; es inevitable. Pero aquí está la diferencia que lo cambia todo: esos filtros también están en el repositorio. Línea por línea. Nombrados. Legibles. Si mañana una regla política se escondiera en el código, estaría en GitHub, bifurcada 4.300 veces, señalada en treinta segundos por el primer investigador que mirara, y convertida en escándalo mundial antes del mediodía.

Es la inversión total del proceso que se le hace. Se le acusa de opacidad. Es la única plataforma del mundo cuya opacidad es, literalmente, imposible.

La asimetría que el «pánico a la injerencia» no puede sostener

Ahora planteemos la única pregunta que de verdad cuenta. ¿En qué plataforma se puede auditar una manipulación algorítmica?

¿Instagram? Caja negra. ¿YouTube? Caja negra. ¿TikTok? Caja negra. ¿Twitch? Caja negra. Meta incluso anunció reducir por defecto el alcance del contenido político, sin que nadie pueda leer una línea. En todas esas plataformas, la acusación «el algoritmo sofoca a tal bando» es rigurosamente infalsable: no se puede ni probar ni refutar, porque el código está sellado.

Y es precisamente la única plataforma que publicó su código — y luego anunció publicar todo el resto, con verificación externa de la producción — la que se arrastra ante un fiscal por «abuso de algoritmo».

Léalo despacio. Se han registrado, con el concurso de Europol, las oficinas de la plataforma más transparente de la historia de las redes sociales, por una manipulación que su propio código permite a cualquiera verificar o desmentir. Mientras las verdaderas cajas negras, las que ningún juez puede abrir, prosperan tranquilamente.

Si buscara de verdad la manipulación, empezaría por los sistemas que no puede ver. Se ensaña con el único que puede leer. No es una investigación sobre la opacidad. Es un castigo a la transparencia.

De lo que se trata realmente: el fin de un monopolio

Entonces, ¿de qué se trata, si no del algoritmo?

Se trata de distribución. Durante medio siglo, el acceso a la atención de masas transitó por un cuello de botella: unas pocas redacciones, unos pocos platós, unos pocos editorialistas. Ese cuello tenía una orientación — no hace falta negarlo; la sociología de la profesión periodística se inclina masivamente hacia un lado, y todos lo saben. No era necesariamente una conspiración; era un medio, una cultura, un entre-sí. Pero el resultado era el mismo: un filtro único, centralizado, decidía lo que merecía existir en el debate.

Ese cuello de botella saltó.

Un hilo algorítmico aprendido a partir de las señales de cientos de millones de individuos no tiene un editor jefe. Es un orden distribuido — Hayek lo habría reconocido al instante: información dispersa, agregada por un mecanismo que ningún planificador controla, produciendo un resultado que nadie decretó. Frente a ese orden distribuido, el viejo mundo ya no tiene la mano. Ya no tiene el botón.

De ahí el vocabulario. «Internacional reaccionaria», dijo el jefe de Estado. La palabra es reveladora: no se emplea «internacional» para describir un desacuerdo de opinión; se emplea para designar una conspiración coordinada. Es el reflejo de René Girard: cuando la mediación pierde su poder, no se cuestiona a sí misma — busca un chivo expiatorio, un culpable único sobre el que descargar la angustia de la pérdida. Musk, X, el algoritmo: el culpable está encontrado, el ritual puede empezar.

Salvo que ya no se puede quemar a un hechicero cuyo grimorio es público.

Esa es la verdadera naturaleza del «pánico a la injerencia». No es el miedo a una manipulación — ni siquiera pueden mostrarla, puesto que el código la desmentiría. Es el miedo, mucho más profundo y mucho más íntimo, de haber perdido el monopolio del relato. Durante décadas, ellos fueron el algoritmo. Decidían lo que subía y lo que desaparecía. Hoy un sistema abierto lo hace en su lugar, sin pedirles su opinión, y llaman a ese sistema una «injerencia» — porque en su gramática, todo lo que escapa a su curación es una anomalía a corregir.

La transparencia como punto final

Se me objetará que la investigación francesa apunta también a otros agravios — el comportamiento de Grok, contenidos problemáticos. Sea. Son cuestiones distintas, que se tratan en su propio terreno, y no las mezclo aquí. Pero sobre el agravio central, el que lanzó toda la máquina — el algoritmo como arma secreta de injerencia — la respuesta ya está escrita, en Rust y en Python, publicada, bifurcada, auditable, y pronto verificada en producción por terceros independientes.

La línea de defensa del viejo mundo descansaba sobre una premisa: que la manipulación algorítmica era una acusación que no se podía ni probar ni levantar, y por tanto se podía blandir indefinidamente. Esa premisa acaba de morir. Existe ahora una plataforma donde la acusación es verificable — y por tanto refutable. Y una acusación refutable que se sigue martilleando sin nunca verificarla deja de ser una inquietud democrática. Se convierte en lo que siempre fue: un intento de recuperar por el derecho lo que se perdió por la tecnología.

El código está abierto. Pronto, todo el código lo estará. La pregunta ya no es si el algoritmo está manipulado — la respuesta está a un git clone de distancia. La pregunta es cuánto tiempo más se pretenderá lo contrario, esperando que nadie vaya a leer.

Ya nadie se deja engañar. Y un hilo abierto no se inventaría. Se lee.
