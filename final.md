# ✅ PDF Chat App with RAG (Flask + pgvector + Ollama)

## 📦 Prerequisites (Debian 12+)
```bash
sudo apt update
sudo apt install -y curl ca-certificates gnupg lsb-release wget python3-venv python3-pip postgresql postgresql-client
```

## 🐘 Install PostgreSQL pgvector
```bash
# Add PostgreSQL APT repo
echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" \
  | sudo tee /etc/apt/sources.list.d/pgdg.list
curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | \
  sudo tee /etc/apt/trusted.gpg.d/postgres.asc > /dev/null

sudo apt update
sudo apt install -y lsof
sudo apt install -y postgresql-15-pgvector
```

## 🛠️ Create PostgreSQL Database
```bash
sudo -u postgres psql
# Inside psql:
CREATE DATABASE vector_db;
\c vector_db
CREATE EXTENSION vector;
CREATE TABLE documents (
  id SERIAL PRIMARY KEY,
  content TEXT,
  embedding vector(384),
  source TEXT,
  chunk_index INT
);
\q
```

## 🧠 Install Ollama + Mistral
```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama run mistral
```

## 🐍 Backend (Flask)
```bash
python3 -m venv ~/rag-env
source ~/rag-env/bin/activate
pip install --upgrade pip
pip install flask flask-cors psycopg2-binary pgvector sentence-transformers pymupdf requests
```

## app.py

```
from flask import Flask, request, jsonify, render_template, session
from flask_cors import CORS
from sentence_transformers import SentenceTransformer
import fitz  # PyMuPDF
import psycopg2
from pgvector.psycopg2 import register_vector
import os
import requests

app = Flask(__name__)
app.secret_key = 'supersecretkey'  # required for session
CORS(app)
model = SentenceTransformer("all-MiniLM-L6-v2")
conn = psycopg2.connect(dbname="postgres", user="postgres", password="postgres", host="localhost")
register_vector(conn)
cur = conn.cursor()

UPLOAD_FOLDER = 'uploads'
os.makedirs(UPLOAD_FOLDER, exist_ok=True)

def chunk_text(text, max_len=300, overlap=50):
    words = text.split()
    return [" ".join(words[i:i+max_len]) for i in range(0, len(words), max_len - overlap)]

def get_embedding(text):
    return model.encode([text])[0].tolist()

@app.route("/")
def index():
    return render_template("index.html")

@app.route("/add", methods=["POST"])
def add():
    content = request.form.get("content")
    emb = get_embedding(content)
    cur.execute("INSERT INTO documents (content, embedding, source, chunk_index) VALUES (%s, %s, %s, %s)",
                (content, emb, "manual", 0))
    conn.commit()
    return "OK"

@app.route("/upload", methods=["POST"])
def upload():
    file = request.files['file']
    filepath = os.path.join(UPLOAD_FOLDER, file.filename)
    file.save(filepath)
    text = "".join([page.get_text() for page in fitz.open(filepath)])
    for i, chunk in enumerate(chunk_text(text)):
        emb = get_embedding(chunk)
        cur.execute("INSERT INTO documents (content, embedding, source, chunk_index) VALUES (%s, %s, %s, %s)",
                    (chunk, emb, file.filename, i))
    conn.commit()
    return "OK"

@app.route("/ask", methods=["POST"])
def ask():
    data = request.get_json()
    query = data["query"]
    emb = get_embedding(query)
    cur.execute("SELECT content FROM documents ORDER BY embedding <-> %s::vector LIMIT 3", (emb,))
    context = "\n".join(r[0] for r in cur.fetchall())
    prompt = f"""Answer the question using the context below:\n\nContext:\n{context}\n\nQuestion: {query}\nAnswer:"""
    res = requests.post("http://localhost:11434/api/generate", json={"model": "mistral", "prompt": prompt, "stream": False})
    ans = res.json().get("response", "[LLM Error]")
    if "chat" not in session:
        session["chat"] = []
    session["chat"].append({"q": query, "a": ans})
    session.modified = True
    return jsonify({"answer": ans})

@app.route("/chat")
def chat():
    return jsonify(session.get("chat", []))

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=True)
```

### /templates/index.html

```
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>RAG Chat App</title>
  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
</head>
<body class="bg-light">
<div class="container py-5">
  <h2 class="mb-4">📄 Upload PDF</h2>
  <form id="uploadForm" class="mb-4">
    <input type="file" id="pdfFile" name="file" accept="application/pdf" class="form-control mb-2">
    <button type="submit" class="btn btn-primary">Upload PDF</button>
  </form>

  <h2 class="mb-4">✍️ Manual Text Entry</h2>
  <form id="addForm" class="mb-4">
    <textarea id="content" name="content" class="form-control mb-2" placeholder="Enter text here..." rows="3"></textarea>
    <button type="submit" class="btn btn-warning">Add Text</button>
  </form>
  <div id="lastAdded" class="alert alert-secondary d-none"></div>

  <h2 class="mb-3">💬 Ask a Question</h2>
  <form id="askForm" class="mb-4">
    <div class="input-group">
      <input type="text" id="query" class="form-control" placeholder="Ask something about your data...">
      <button class="btn btn-success" type="submit">Ask</button>
    </div>
  </form>
  <div id="answerContainer" class="alert alert-info d-none">
    <strong>Answer:</strong>
    <pre id="answer" class="mb-0"></pre>
  </div>

  <h4 class="mt-5">🕒 Chat History</h4>
  <ul id="chatLog" class="list-group mb-5"></ul>
</div>
<script>
  const chatLog = document.getElementById("chatLog");
  async function loadChatHistory() {
    const res = await fetch("/chat");
    const history = await res.json();
    chatLog.innerHTML = "";
    history.forEach((item) => {
      const li = document.createElement("li");
      li.className = "list-group-item";
      li.innerHTML = `<strong>Q:</strong> ${item.q}<br><strong>A:</strong> ${item.a}`;
      chatLog.appendChild(li);
    });
  }

  document.getElementById('uploadForm').onsubmit = async (e) => {
    e.preventDefault();
    const formData = new FormData();
    formData.append("file", document.getElementById("pdfFile").files[0]);
    const res = await fetch("/upload", { method: "POST", body: formData });
    alert(await res.text());
  };

  document.getElementById('addForm').onsubmit = async (e) => {
    e.preventDefault();
    const content = document.getElementById("content").value;
    const form = new FormData();
    form.append("content", content);
    const res = await fetch("/add", { method: "POST", body: form });
    if ((await res.text()) === "OK") {
      document.getElementById("lastAdded").textContent = `✅ Added: ${content}`;
      document.getElementById("lastAdded").classList.remove("d-none");
      document.getElementById("content").value = "";
    }
  };

  document.getElementById('askForm').onsubmit = async (e) => {
    e.preventDefault();
    const query = document.getElementById("query").value;
    const answerEl = document.getElementById("answer");
    const container = document.getElementById("answerContainer");
    answerEl.textContent = "Thinking...";
    container.classList.remove("d-none");
    const res = await fetch("/ask", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ query })
    });
    const data = await res.json();
    answerEl.textContent = data.answer;
    await loadChatHistory();
  };

  window.onload = loadChatHistory;
</script>
</body>
</html>
```



---

You can now:
- Run: source ~/rag-env/bin/activate && python3 app.py
- Open index.html in browser or serve via nginx/static
- Ask questions from uploaded PDFs!
