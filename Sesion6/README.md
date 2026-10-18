# Sesión 6 · AgentBot de críticas con herramientas MCP

**Lo que dice el público.** Un chatbot que decide si consulta lo que opinan los usuarios o la ficha
técnica de FilmAffinity, responde citando la fuente y sabe cuándo no sabe.

Grupo Tango: Gustavo Betancur · Mike Martinez · Pedro Jojoa · Juan Sebastian Orozco

## Entrega

| Archivo | Qué es |
|---|---|
| [`06_agentbot_mcp_criticas.ipynb`](06_agentbot_mcp_criticas.ipynb) | El notebook ejecutado en Colab (GPU T4), con todas sus salidas. |
| [`video_demostracion.mp4`](video_demostracion.mp4) | Video de la demostración (1:50). |

**[▶ Ver el video de demostración](video_demostracion.mp4)**: la interfaz de Gradio sobre los dos
servidores MCP. Se ven una respuesta con citas `[F n]` y el panel de fuentes (nota, fecha,
relevancia y enlace a FilmAffinity), una pregunta de datos resuelta con la ficha y la señal
`SIN_FUENTES` cuando no hay críticas que respalden la respuesta.

El código fuente (generador del notebook y servidores MCP) está en
[icesi-nlp-entregables/sesion6](https://github.com/juanseo/icesi-nlp-entregables/tree/main/sesion6).

## Qué construimos

La sesión 5 dejó un RAG monolítico sobre las críticas de FilmAffinity: una cadena que siempre
recupera y siempre redacta. Aquí movimos esa recuperación a **dos servidores MCP propios** y le
dimos la decisión a un agente:

- **`buscar_criticas`** (puerto 9082): E5 + BM25 + fusión RRF + *reranker*, con una señal de
  abstención calibrada. Si nada supera el umbral, devuelve `SIN_FUENTES`.
- **`ficha_pelicula`** (puerto 9083): dirección, reparto, género, nota media, votos y enlace.
- **Un agente** con llama3.2:3b (`create_react_agent` de LangGraph) decide qué herramienta llamar,
  en tres modos: **Autónomo** (decide el modelo), **Guiado** (pauta según la intención) y
  **Determinístico** (herramienta forzada).
- **Un chatbot en Gradio** que muestra cada respuesta con su traza MCP y sus fuentes.

El índice tiene 300 películas, 8.561 críticas y 24.062 documentos (e5-large y
bge-reranker-v2-m3), y comparamos dos modelos en Ollama: llama3.2:3b y qwen2.5:3b.

## Cómo leer la corrida

Durante la corrida, el servidor del buscador falló en cada búsqueda con
`'E5Consulta' object is not callable`: su clase de embeddings no heredaba de `Embeddings` de
LangChain. El error no detuvo el notebook, así que **el barrido de experimentos (§4.4 a §6.7) se
hizo con un buscador que devolvía error y cero fuentes**. Después de §7.2 se corrigió el servidor
dentro de la misma sesión, y desde ahí el sistema recupera y cita fuentes reales: esa es la
versión del video. Las celdas de verificación y corrección que se agregaron durante la corrida
están en §4.2 y después de §7.2.

Por eso separamos lo que la corrida mide de lo que no.

## Resultados

| Medida | Resultado | Hipótesis |
|---|---|---|
| Routing exacto de llama3.2:3b en Autónomo | **1,00** en preguntas de opinión y **1,00** en las de datos | H1 · se sostiene |
| Guiado contra Autónomo en preguntas mixtas | **0,63 → 0,83** (+0,20) | H2 · se sostiene |
| qwen2.5:3b contra llama3.2:3b | opinión 0,95 · datos 0,70 · mixtas 0,13, frente a 1,00 en las tres | H4 · refutada |
| Señal de abstención calibrada (§3.5) | AUC **1,000**, exactitud balanceada **1,00** | respalda H5 |
| Agente con la herramienta caída | inventó opiniones en **~2 de cada 3** preguntas de opinión | hallazgo |
| Costo de la capa MCP, abstención vía servidor, citas válidas | no medibles: el buscador devolvía error | H3 y H5 · sin medir |

El routing sí es válido porque el agente decide qué herramienta llamar **antes** de ver lo que
devuelve. Los veredictos de H3 y H5 que imprime §8.1 no deben leerse como resultados.

## Conclusiones

1. **El agente elige bien la herramienta cuando la pregunta pide una sola.** llama3.2:3b acertó
   en el 100% de las preguntas de opinión y de datos. La dificultad está en las mixtas, que
   necesitan las dos: en Autónomo solo el 63% llama a ambas.
2. **El modo Guiado arregla justo esa falla.** Las mixtas suben de 0,63 a 0,83 sin empeorar las
   demás.
3. **Contra lo que sugiere el guía, qwen2.5:3b invoca peor que llama3.2:3b** en este dominio:
   36 puntos menos en promedio, y 0,13 en las mixtas.
4. **La señal de abstención separa perfecto lo que el índice tiene de lo que no**: AUC de 1,000
   entre películas del índice, fuera de él e inventadas. En el video, la película inventada
   termina en `SIN_FUENTES` y el bot responde que no tiene información.
5. **La lección principal: un agente no se da cuenta de que su herramienta falló.** Con el
   buscador devolviendo error, el modelo siguió respondiendo con fluidez y fabricó opiniones de
   usuarios, con notas y hasta citas `[F n]`. Por eso la abstención no puede depender de cómo
   redacta el modelo: tiene que venir de la herramienta (`SIN_FUENTES`), y el sistema tiene que
   verificar que la herramienta responde. La versión corregida del notebook detiene la corrida si
   una herramienta devuelve error.
