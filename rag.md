✅ Check EC2 IP

```
TOKEN=$(curl -sX PUT "http://169.254.169.254/latest/api/token" \
       -H "X-aws-ec2-metadata-token-ttl-seconds: 60")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
     http://169.254.169.254/latest/meta-data/public-ipv4

# 🧠 Retrieval-Augmented Generation (RAG) with pgvector + Mistral

This app allows storing documents and asking questions via semantic search + LLM (Mistral via Ollama).

---

## 🚧 Prerequisites

- Python 3.8+
- PostgreSQL 17 with `pgvector`
- Ollama installed and running `mistral`

---

## 📥 Ollama Setup

```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama run mistral

This runs a local LLM API at http://localhost:11434

sudo apt install python3-venv python3-full -y
python3 -m venv ~/pgvector-env
source ~/pgvector-env/bin/activate
pip install flask psycopg2-binary pgvector sentence-transformers requests


app.py

from flask import Flask, request, render_template
from sentence_transformers import SentenceTransformer
import psycopg2
from pgvector.psycopg2 import register_vector
import requests

app = Flask(__name__)
model = SentenceTransformer("all-MiniLM-L6-v2")

conn = psycopg2.connect(
    dbname="vector_db",
    user="postgres",
    password="postgres",
    host="localhost"
)
register_vector(conn)
cur = conn.cursor()

def get_embedding(text):
    return model.encode([text])[0].tolist()

@app.route("/", methods=["GET", "POST"])
def index():
    results = []
    message = ""
    if request.method == "POST":
        content = request.form.get("content")
        if content:
            try:
                emb = get_embedding(content)
                cur.execute("INSERT INTO documents (content, embedding) VALUES (%s, %s)", (content, emb))
                conn.commit()
                message = "✅ Document added!"
            except Exception as e:
                conn.rollback()
                message = f"❌ Error: {str(e)}"
    return render_template("index.html", results=results, answer=None, message=message)

@app.route("/ask", methods=["POST"])
def ask():
    query = request.form["query"]
    query_vec = get_embedding(query)

    cur.execute("""
        SELECT content FROM documents
        ORDER BY embedding <-> %s::vector
        LIMIT 3
    """, (query_vec,))
    results = cur.fetchall()
    context = "\n".join([row[0] for row in results])

    prompt = f"""Answer the question based on the context below.
Context:
{context}

Question: {query}
Answer:"""

    try:
        response = requests.post(
            "http://localhost:11434/api/generate",
            json={"model": "mistral", "prompt": prompt, "stream": False}
        )
        answer = response.json().get("response", "[No response]")
    except Exception as e:
        answer = f"LLM error: {e}"

    return render_template("index.html", results=results, answer=answer, message=None)

templates/index.html

<!DOCTYPE html>
<html>
<head>
    <title>pgvector RAG App</title>
</head>
<body>
    <h2>Add Document</h2>
    <form action="/" method="post">
        <textarea name="content" rows="3" cols="60" placeholder="Enter content to store..."></textarea><br>
        <input type="submit" value="Add Document">
    </form>
    {% if message %}
        <p>{{ message }}</p>
    {% endif %}

    <hr>

    <h2>Ask a Question</h2>
    <form action="/ask" method="post">
        <input type="text" name="query" size="60" placeholder="Ask a question...">
        <input type="submit" value="Ask">
    </form>

    {% if answer %}
        <h3>Answer:</h3>
        <p>{{ answer }}</p>
    {% endif %}
</body>
</html>

### run the app

ollama run mistral

export FLASK_APP=app.py
export FLASK_ENV=development
flask run --host=0.0.0.0 --port=5000



```
