# 🧠 Semantic Search App with PostgreSQL + pgvector

A simple Flask app for storing documents and performing semantic similarity search using pgvector and SentenceTransformers.

---

## 🐘 Prerequisites

- Ubuntu VM or EC2
- Python 3.8+
- PostgreSQL 17
- pgvector extension
- Internet access (to download transformer model)

---

## 📦 Install PostgreSQL + pgvector

```bash
sudo apt install curl ca-certificates -y
sudo install -d /usr/share/postgresql-common/pgdg
sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc
. /etc/os-release
echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt $VERSION_CODENAME-pgdg main" | sudo tee /etc/apt/sources.list.d/pgdg.list
sudo apt update
sudo apt install -y postgresql postgresql-17-pgvector


## PostgreSQL set up

sudo -u postgres psql

CREATE DATABASE vector_db;
\c vector_db
CREATE EXTENSION vector;

CREATE TABLE documents (
  id SERIAL PRIMARY KEY,
  content TEXT,
  embedding vector(384)
);
\q

## Project Structure

mkdir ~/pgvector-app && cd ~/pgvector-app

## app.py


from flask import Flask, request, render_template
from sentence_transformers import SentenceTransformer
import psycopg2
from pgvector.psycopg2 import register_vector

app = Flask(__name__)
model = SentenceTransformer("all-MiniLM-L6-v2")

conn = psycopg2.connect(
    dbname="vector_db",
    user="postgres",
    password="postgres",  # change if needed
    host="localhost"
)
register_vector(conn)
cur = conn.cursor()

def get_embedding(text):
    return model.encode([text])[0].tolist()

@app.route("/", methods=["GET", "POST"])
def index():
    results = []
    if request.method == "POST":
        query = request.form["query"]
        query_vec = get_embedding(query)
        cur.execute("""
            SELECT content, embedding <-> %s::vector AS distance
            FROM documents
            ORDER BY distance ASC
            LIMIT 5
        """, (query_vec,))
        results = cur.fetchall()
    return render_template("index.html", results=results)

@app.route("/add", methods=["POST"])
def add():
    content = request.form["content"]
    try:
        emb = get_embedding(content)
        cur.execute("INSERT INTO documents (content, embedding) VALUES (%s, %s)", (content, emb))
        conn.commit()
        return "Document added!"
    except Exception as e:
        conn.rollback()
        return f"Error: {str(e)}"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=True)


## templates/index.html


<!doctype html>
<html lang="en">
<head>
  <title>Semantic Search</title>
</head>
<body class="p-5">
  <h2>🔍 Semantic Search</h2>
  <form method="post">
    <input name="query" placeholder="Search..." required>
    <button>Search</button>
  </form>

  {% if results %}
  <h4>Top Results:</h4>
  <ul>
    {% for row in results %}
      <li>{{ row[0] }} (Distance: {{ row[1] }})</li>
    {% endfor %}
  </ul>
  {% endif %}

  <hr>
  <h4>➕ Add Document</h4>
  <form action="/add" method="post">
    <textarea name="content" placeholder="Enter content..." required></textarea>
    <button>Add</button>
  </form>
</body>
</html>


## Run the App

cd ~/pgvector-app
source ~/pgvector-env/bin/activate
python3 app.py
```

