# Week 3 Exercise Plan: Synthetic Data Generator

## Goal

You will build a tool. The tool takes a business problem as input. It creates a fake dataset for that problem. For example, a user might type "patient records for a small clinic." Your tool then makes sample data that fits this problem.

Your tool will use a large language model to design the data. It will offer more than one output format. It will run in a Gradio app, so a user can interact with it in a browser.

---

## Step 1: Set up your environment

**Action:** Create a new notebook in the `week3` folder. Name it something like `synthetic_data_generator.ipynb`. In the first cell, add these imports:

```python
import os
import re
import json
from dotenv import load_dotenv
from openai import OpenAI
import anthropic
import gradio as gr
import pandas as pd
```

Then load your keys:

```python
load_dotenv()
openai_api_key = os.getenv("OPENAI_API_KEY")
anthropic_api_key = os.getenv("ANTHROPIC_API_KEY")
openai = OpenAI()
claude = anthropic.Anthropic()
```

**Why this step matters:** You will call more than one model in this project. You need the right library for each model. You need your keys loaded before you make any API call. Your `pyproject.toml` file already has `openai`, `anthropic`, `gradio`, and `pandas` listed. So you do not need to install anything new.

---

## Step 2: Add your local model too

You also have Ollama running on your machine. This gives you a free, open-source option next to the paid APIs.

**Action:** Add a client for your local model:

```python
ollama_client = OpenAI(base_url="http://localhost:11434/v1", api_key="ollama")
```

**Why this step matters:** The exercise wants you to compare models. A local open-source model costs nothing to run. A frontier model like GPT or Claude often gives higher quality. Your tool should let the user pick.

---

## Step 3: Write your system prompt

**Action:** Write a system message that sets the model's job. Keep it narrow and clear:

```python
system_message = (
    "You are an assistant that creates synthetic datasets for a given business problem. "
    "You respond only with the dataset and, when asked, the Python code to build it. "
    "You do not add extra chat or explanation."
)
```

**Why this step matters:** A narrow system prompt stops the model from adding chatty text around your data. This keeps the output clean and easy to parse later.

---

## Step 4: Write your user prompt builder

**Action:** Write a function that turns user choices into one clear prompt:

```python
def build_user_prompt(business_problem, data_type, num_rows, file_format):
    return f"""
The business problem is: {business_problem}
Create a synthetic dataset of type: {data_type}
Number of rows: {num_rows}
Output format: {file_format}

Also write Python code that builds this dataset and saves it to a file
in the {file_format} format. Use only pandas and the standard library.
Wrap the code in a single Python code block.
"""
```

**Why this step matters:** A clear, structured prompt gives you a consistent, structured answer. This makes your next steps much easier.

---

## Step 5: Pick your generation strategy

You have two ways to get your dataset. Learn both. Pick one as your main path.

**Direct generation:** Ask the model to write out the rows of data itself, in the chat reply. This is simple. It breaks down for large datasets, because the model can only write so much text at once.

**Code generation:** Ask the model to write a Python script. The script builds the dataset. You then run that script yourself. This scales to large datasets. It needs one more safety step: you must check the code before you run it.

**Action:** Use code generation as your main path. Use direct generation as a simple fallback for very small datasets.

**Why this step matters:** Real business datasets often need thousands of rows. A model cannot type out thousands of rows in one reply. Code generation solves this problem, because your computer runs the loop, not the model.

---

## Step 6: Call the model and stream the reply

**Action:** Write one function per model type. Reuse the streaming pattern from your week 2 work. For example, for OpenAI:

```python
def generate_with_openai(business_problem, data_type, num_rows, file_format):
    messages = [
        {"role": "system", "content": system_message},
        {"role": "user", "content": build_user_prompt(business_problem, data_type, num_rows, file_format)},
    ]
    stream = openai.chat.completions.create(model="gpt-4o-mini", messages=messages, stream=True)
    result = ""
    for chunk in stream:
        result += chunk.choices[0].delta.content or ""
        yield result
```

Write a matching function for Claude, and one for your local Ollama model.

**Why this step matters:** Streaming shows partial output right away. This makes your Gradio app feel fast and alive, instead of frozen while it waits.

---

## Step 7: Pull the code out of the reply

The model wraps its Python code in a fenced block, like this:  ```python ... ```

**Action:** Write a function that extracts just the code:

```python
def extract_code(response_text):
    match = re.search(r"```python(.*?)```", response_text, re.DOTALL)
    return match.group(1).strip() if match else None
```

**Why this step matters:** You cannot run a whole chat reply as Python. You must pull out only the code block, and leave the rest of the text for your Markdown preview.

---

## Step 8: Run the code safely

**Action:** Show the extracted code to the user before you run it. Only run it after the user agrees. Run it in its own namespace:

```python
def run_generated_code(code):
    namespace = {}
    exec(code, namespace)
```

**Why this step matters:** You are about to run code that a model wrote, not code that you wrote. Always let a human check it first. This is a normal safety rule for any tool that runs generated code.

---

## Step 9: Build your Gradio UI

**Action:** Add these input pieces:
- A text box for the business problem.
- A dropdown for data type (tabular, text, time-series).
- A dropdown for file format (CSV, JSON, Markdown).
- A number box for row count.
- A dropdown for model choice (GPT, Claude, local Ollama model).
- A "Generate" button.

Add these output pieces:
- A Markdown box, to show the model's full reply as it streams in.
- A code box, to show the extracted Python code.
- A "Run code" button, separate from "Generate."

**Why this step matters:** Splitting "Generate" from "Run code" gives the user a chance to review the code first. This matches the safety rule from Step 8.

---

## Step 10: Wire it all together

**Action:** Connect your Gradio components with `.click()` handlers. Route the model choice to the right function from Step 6. Route the "Run code" button to your Step 8 function.

**Why this step matters:** This is the step where all your separate pieces become one working app.

---

## Step 11: Test your tool

**Action:** Try at least three different business problems. For example:
- "Customer orders for a small online bakery"
- "Patient visit records for a dental clinic"
- "Daily temperature readings for five cities"

Try each one with a different file format. Check that the output file is correct and makes sense.

**Why this step matters:** Testing with different inputs shows you where your prompt or your code needs more work. One test is never enough.

---

## Step 12 (Stretch goal): Compare models

**Action:** Add a mode that runs the same request through two models at once. Show both outputs side by side. Note differences in speed, cost, and quality.

**Why this step matters:** This is good practice for later projects. Many real tools must choose between a fast local model and a stronger paid model.

---

## Order of Work Summary

1. Set up imports and load your keys.
2. Add your local Ollama model as a free option.
3. Write a narrow system prompt.
4. Write a function that builds the user prompt.
5. Choose code generation as your main strategy.
6. Write a streaming call function for each model.
7. Extract the Python code block from the reply.
8. Run the code only after a safety check, in its own namespace.
9. Build the Gradio inputs and outputs.
10. Wire the buttons to your functions.
11. Test with at least three business problems.
12. (Stretch) Compare two models side by side.
