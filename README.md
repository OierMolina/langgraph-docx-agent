# Agente LangGraph DOCX (DEV-108)

Este proyecto implementa un agente con LangGraph capaz de generar documentos de Microsoft Word (`.docx`) a partir de instrucciones en lenguaje natural. La solución divide el trabajo en dos pasos: primero diseña la estructura y el contenido del documento en Markdown, y después compila ese contenido a un archivo Word utilizando `python-docx`.

## Objetivo

El objetivo del agente es transformar un prompt de usuario en un documento Word listo para usar mediante un flujo de dos nodos:

1. `writer_node`: utiliza un modelo LLM para redactar el contenido en Markdown estricto.
2. `docx_generator_node`: interpreta ese Markdown y genera el archivo `salida.docx`.

Este enfoque separa claramente la fase de generación de contenido de la fase de renderizado del documento, lo que hace la solución más sencilla de mantener, probar y extender.

## Arquitectura

La aplicación utiliza un estado compartido llamado `AgentState`, definido como `TypedDict`, para transportar la información entre nodos del grafo.

Campos principales del estado:

- `messages`: historial de mensajes del grafo, gestionado con `add_messages`.
- `markdown_content`: contenido generado por el LLM en formato Markdown.
- `docx_path`: ruta del archivo `.docx` generado al final del flujo.

### Nodo `writer_node`

Este nodo instancia `ChatGroq` con el modelo `llama-3.1-8b-instant` y temperatura `0.4`. Su responsabilidad es actuar como redactor experto y devolver un documento en Markdown limpio:

- `#` para título principal
- `##` para subtítulos
- texto normal para párrafos
- listas con `-` o `*` cuando sea necesario

El resultado se guarda en `markdown_content`.

### Nodo `docx_generator_node`

Este nodo crea un documento con `Document()` y procesa el contenido Markdown línea a línea:

- `# ` se convierte en heading de nivel 1
- `## ` se convierte en heading de nivel 2
- `- ` o `* ` se convierten en elementos de lista
- cualquier línea no vacía restante se añade como párrafo normal

Finalmente, el documento se guarda como `salida.docx` y la ruta se almacena en `docx_path`.

## Flujo del Grafo

El flujo definido con LangGraph es:

`START -> writer -> generator -> END`

Esto garantiza que el documento Word solo se genere después de que el contenido Markdown haya sido redactado correctamente.

## Código Final

```python
from typing import Annotated, TypedDict

from dotenv import load_dotenv
from docx import Document
from langchain_core.messages import HumanMessage, SystemMessage
from langgraph.graph import END, START, StateGraph
from langgraph.graph.message import add_messages
from langchain_groq import ChatGroq


class AgentState(TypedDict):
    messages: Annotated[list, add_messages]
    markdown_content: str
    docx_path: str


def writer_node(state: AgentState) -> AgentState:
    llm = ChatGroq(model="llama-3.1-8b-instant", temperature=0.4)

    system_prompt = (
        "Actua como un redactor experto. Devuelve el contenido en Markdown estricto, "
        "usando '# ' para el titulo principal, '## ' para subtitulos y texto normal "
        "para los parrafos. Puedes usar listas con '- ' o '* ' si ayudan. No uses "
        "bloques de codigo ni fences como ```markdown. Devuelve solo el texto crudo."
    )

    user_prompt = state["messages"][-1].content
    response = llm.invoke(
        [
            SystemMessage(content=system_prompt),
            HumanMessage(content=user_prompt),
        ]
    )

    return {
        "messages": [response],
        "markdown_content": response.content,
    }


def docx_generator_node(state: AgentState) -> AgentState:
    doc = Document()

    for raw_line in state["markdown_content"].splitlines():
        line = raw_line.strip()

        if line.startswith("# "):
            doc.add_heading(line[2:].strip(), level=1)
        elif line.startswith("## "):
            doc.add_heading(line[3:].strip(), level=2)
        elif line.startswith("- ") or line.startswith("* "):
            doc.add_paragraph(line[2:].strip(), style="List Bullet")
        elif line:
            doc.add_paragraph(line)

    output_path = "salida.docx"
    doc.save(output_path)

    return {"docx_path": output_path}


graph_builder = StateGraph(AgentState)
graph_builder.add_node("writer", writer_node)
graph_builder.add_node("generator", docx_generator_node)
graph_builder.add_edge(START, "writer")
graph_builder.add_edge("writer", "generator")
graph_builder.add_edge("generator", END)
graph = graph_builder.compile()


if __name__ == "__main__":
    load_dotenv()

    prompt = (
        "Genera una propuesta comercial breve para un servicio de consultoria IA, "
        "con introduccion, servicios y precios"
    )

    result = graph.invoke(
        {
            "messages": [HumanMessage(content=prompt)],
            "markdown_content": "",
            "docx_path": "",
        }
    )

    print(f"Documento generado en: {result['docx_path']}")
```

## Instrucciones de Uso

1. Instala las dependencias del proyecto:

```bash
pip install -r requirements.txt
```

2. Configura el archivo `.env` a partir de `.env.example` e incluye tu clave:

```env
GROQ_API_KEY=tu_clave_aqui
```

3. Ejecuta el agente:

```bash
python main.py
```

Si la clave es válida y las dependencias están instaladas, el flujo generará un archivo llamado `salida.docx` en la raíz del proyecto.
