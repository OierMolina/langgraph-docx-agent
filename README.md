# Agente LangGraph DOCX (DEV-108)

Este proyecto es la plantilla base para un agente que utiliza LangGraph y `python-docx` para generar documentos de Microsoft Word (`.docx`) a partir de instrucciones en lenguaje natural.

## Instalación

1. Crear un entorno virtual.
2. Instalar las dependencias del proyecto:

```bash
pip install -r requirements.txt
```

3. Copiar `.env.example` a `.env` y configurar la clave necesaria:

```env
GROQ_API_KEY=tu_clave_aqui
```

## Ejecución

```bash
python main.py
```
