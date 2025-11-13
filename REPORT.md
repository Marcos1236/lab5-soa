# Lab 5 Integration and SOA - Project Report

## 1. EIP Diagram (Before)

![Before Diagram](diagrams/before.png)

Esta aplicación pretendía demostrar patrones EIP básicos. Encontramos un source que genera enteros secuenciales, un flujo principal que transforma y analiza los números, un router que envía cada número a canales "evenChannel" u "oddChannel", handlers para cada rama y un gateway programado que inyecta números negativos aleatorios en el sistema. El objetivo de este código es mostrar el uso de Router, Transformer, Filter, Channels, Service Activator y Message Gateway.

Sin embargo, observamos varios errores que provocan que el código inicial difiera del diagrama target, correspondiéndose el código inicial con el diagrama before.png en su lugar. En primer lugar he detectado inconsistencias en el enrutamiento, concretamente, faltaba el canal llamado "numbersChannel". Al faltar este canal, los números inyectados por el "Integer Source" llegaban directamente al "Even or Odd" mientras que los inyectados por el "Message Gateway" llegaban directamente al "evenChannel". Esto rompe completamente con lo que el diagrama representa ya que, todos los números negativos están siendo gestionados como pares independientemente de si lo son o no.

Por otra parte, la lógica de filtrado en el 'Filter' que hay antes del "Odd Handler" estaba invertida por lo que se estaban rechazando todos los números impares. De esta manera, todos los números impares llegaban al "oddChannel" pero ninguno llegaba al "Odd Handler" ya que el filtro los rechazaba.

Además de esto, los canales estaban mal definidos, siendo el canal "evenChannel" uno de tipo pub/sub cuando tenía que ser de tipo directo y el canal "oddChannel", el cual sí que debía de ser pub/sub, era de tipo directo. Esto provocaba que los dos suscriptores del canal "oddChannel" estaban compitiendo por las publicaciones. Esto se puede ver en la ejecución ya que a veces veíamos cómo un número impar estaba siendo gestionado por "Some Service" y otras veces por el "Filter" que va antes de "Odd Handler".

---

## 2. What Was Wrong

Los bugs encontrados en el código inicial fueron:

- **Falta del canal de entrada "numberChannel"**: 
El "Message Gateway" estaba configurado para enviar los números directamente a "evenChannel". Al mismo tiempo el "Integer Source" enviaba sus números a la ruta del router. El resultado era que los números generados por el "Integer Source" pasaban por el router pero los inyectados por el gateway se saltaban el router.

Esto pasó debido a que no existía un canal de entrada común que unificara los mensajes antes del router. El gateway apuntaba al canal "evenChannel" en vez de a un canal de entrada general.

Para arreglarlo he creado un "numberChannel" (canal directo) que actúa como punto de entrada único al "Even or Odd". Tanto el gateway como el "Integer Source" publican ahora en "numberChannel". Se ha añadido el flujo numberChannelFlow (igual que "Even or Odd" en el diagrama objetivo) que toma mensajes de "numberChannel" y los enruta a "evenChannel" o "oddChannel". 

- **Filter invertido**: 
En el oddFlow, el cual procesaba los mensajes de "oddChannel", existía un filtro que aceptaba pares y rechazaba impares. En la práctica, los números impares eran enviados a "oddChannel" pero eran filtrados antes de llegar al "Odd Handler", por lo tanto, no llegaban a procesarse correctamente. 

Esto pasó debido a un error en la lógica del filtro: p % 2 == 0 en vez de p % 2 != 0. Además de esto, el error también ocurrió por redundancia funcional ya que el router ya decide la paridad y por lo tanto el filter tampoco era necesario. Además de esto, en el diagrama objetivo no existe ese filter.

Para arreglarlo he eliminado el filtro ya que era redundante y no aparecía en el diagrama objetivo. De esta manera el error queda solucionado y el código corresponde con el diagrama objetivo.

- **Tipos de canales intercambiados**:
El canal "evenChannel" estaba definido como pub/sub cuando debía ser directo, mientras que "oddChannel" estaba definido como directo cuando en realidad debía ser pub/sub. Debido a esto, los dos suscriptores de "oddChannel" competían por los mensajes en lugar de recibir ambos la publicación. A veces era el servicio el que procesaba el mensaje y otras veces lo hacía el handler.

Esto pasó debido a la confusión entre el Direct Channel (punto a punto) y el Publish-Subscribe Channel (fan-out). Teníamos una definición incorrecta que no reflejaba el diagrama objetivo.

Para arreglarlo he redefinido los canales con el tipo apropiado. Con esto, "oddChannel" será pub/sub y ambos consumidores recibirán el mensaje, tal y como indica el diagrama.

- **Paridad con números negativos**:
El gateway inyectaba los números negativos en "oddChannel" (error solucionado anteriormente) y, por lo tanto, la comprobación de paridad se realizaba sin considerar el signo ya que los negativos iban siempre al canal de los impares. La comparación se realizaba con la expresión 'p % 2 == 0', lo cual puede devolver valores negativos en Kotlin si p es negativo.

Esto pasó debido a que no se tuvo en cuenta que la entrada podía ser negativa y que la comprobación de paridad debe basarse en valor absoluto.

Para arreglarlo, en el router he usado 'abs(p) % 2 == 0' para realizar la comprobación de paridad. 

---

## 3. What You Learned

- Sobre Enterprise Integration Patterns (EIP): 

He aprendido la importancia de un punto de entrada único y de un router central, los cuales separan la producción de mensajes de la decisión de enrutamiento. 

Además, he entendido la diferencia práctica entre Direct Channel y Publish-Subscribe Channel y cuando utilizar cada uno dependiendo de lo que queremos.

Por último, he aprendido que los patrones se complementan. Esto lo he visto sobre todo en el diagrama objetivo, en el cual podemos observar claramente que estos patrones van siguiendo una secuencia lógica: Gateway -> Channel -> Router -> Transformer -> Handler / Service Activator.

- Spring Integration:

He aprendido el uso de 'integrationFlow' en Spring, el cual nos permite componer pipelines fácilmente (transform, handle, channel). Además de esto, he visto como se utilizan Pollers, los cuales permiten integrar fuentes de mensajes periódicas y el Message Gateway, el cual convierte llamadas de método en mensajes sobre un channel.

Por último, hemos podido comprobar la importancia de una buena elección de tipos de Message Channels dependiendo de las necesidades que tengamos. En nuestro caso, la salida obtenida cambiaba mucho ya que queríamos que tanto el Handler como el Service Activator recibieran el mensaje pero solo uno de ellos lo estaba haciendo (y además de forma no determinista).

- Desafíos y cómo resolverlos:

En un primer momento fue complicado entender lo que estaba ocurriendo en la aplicación, principalmente porque aún no habíamos dado en clase de teoría nada sobre los patrones EIP. Luego, tras estudiar detenidamente el diagrama, fue más sencillo comprobar lo que hacía el programa al ir reconociendo las diferencias y similitudes del código con este. Finalmente, una vez sabía cómo funcionaba el código a la perfección, ha sido bastante fácil realizar el diagrama del código inicial.

Por otra parte y ligado a lo anterior, inicialmente no conocía la diferencia entre un Direct Channel y un Publish-Subscribe Channel. No fue tras un poco de investigación que descubrí sus diferencias y entendí a la perfección porque a veces recibía los mensajes en Odd Handler y otras veces el Service Activator.

---

## 4. AI Disclosure

Para la realización de esta práctica he utilizado ChatGPT.

Esta herramienta me ha ayudado principalmente a entender el código inicial y los patrones EIP ya que, como he comentado anteriormente, antes de realizar la práctica desconocía su existencia. 

El código completo lo he realizado yo ya que, una vez teniendo el diagrama del código inicial y entendiendo a la perfección las diferencias entre este y el diagrama objetivo, los cambios a realizar eran bastante simples. 

Gracias a esto he conseguido una buena comprensión del código y de los patrones utilizados. He comprendido que el Messaging Gateway actúa como una fachada para convertir llamadas de método en mensajes, los cuales deben ser publicados en un canal de entrada común (numberChannel) para que todos los mensajes sigan la misma ruta lógica. Por otra parte el Integer Source produce enteros secuenciales y el integrationFlow (con Polling) los extrae periódicamente. El router central comprueba la paridad y enruta en consecuencia, manejando correctamente los valores negativos. Además de esto, he comprendido los diferentes tipos de canales que podemos tener dependiendo de las necesidades que tengamos. Cada rama transforma los mensajes y luego los maneja. Además, el Service Activator demuestra como un componente fuera del flujo DSL puede subscribirse a canales para actuar como consumidores.

---
