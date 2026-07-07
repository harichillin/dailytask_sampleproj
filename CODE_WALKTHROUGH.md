# Code Walkthrough

This explains every part of the project, file by file, so you can answer questions about any line.

---

## 1. `app.py` — the backend

### Imports and setup

```python
from flask import Flask, render_template, request, redirect, url_for
import sqlite3

app = Flask(__name__)
DB_NAME = "tasks.db"
```

- `Flask` is the web framework — it listens for browser requests and runs Python code in response.
- `sqlite3` is Python's built-in library for talking to a SQLite database (a database that lives in a single file, no server needed).
- `app = Flask(__name__)` creates the actual web application object. `__name__` just tells Flask where the current file lives, so it can find `templates/` and `static/`.
- `DB_NAME` is the filename of the database. Kept as a constant so it's only written in one place.

### Database connection helper

```python
def get_db_connection():
    conn = sqlite3.connect(DB_NAME)
    conn.row_factory = sqlite3.Row
    return conn
```

- `sqlite3.connect(DB_NAME)` opens (or creates) `tasks.db`.
- `conn.row_factory = sqlite3.Row` makes query results behave like dictionaries, so you can write `task["title"]` instead of `task[0]`. Without this line you'd only get plain tuples.
- Every route that touches the database calls this function to get a fresh connection.

### Creating the table

```python
def init_db():
    conn = get_db_connection()
    conn.execute("""
        CREATE TABLE IF NOT EXISTS tasks (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            title TEXT NOT NULL,
            done INTEGER NOT NULL DEFAULT 0
        )
    """)
    conn.commit()
    conn.close()
```

- `CREATE TABLE IF NOT EXISTS` means: only create the table the first time; if it already exists, do nothing (so restarting the app doesn't wipe your tasks).
- Table columns:
  - `id` — auto-incrementing unique number for each task (this is the "primary key").
  - `title` — the text of the task, required (`NOT NULL`).
  - `done` — `0` (not done) or `1` (done). Defaults to `0` when a task is first created.
- `conn.commit()` saves the change to disk. `conn.close()` releases the connection.
- This function is only called once, at the bottom of the file, when the app starts.

### Route: show the page (`/`)

```python
@app.route("/")
def index():
    conn = get_db_connection()
    tasks = conn.execute("SELECT * FROM tasks ORDER BY id DESC").fetchall()
    conn.close()
    return render_template("index.html", tasks=tasks)
```

- `@app.route("/")` means: "when someone visits the homepage, run this function."
- `SELECT * FROM tasks ORDER BY id DESC` fetches every task, newest first (`DESC` = descending order by id).
- `.fetchall()` turns the query result into a Python list of rows.
- `render_template("index.html", tasks=tasks)` loads `templates/index.html` and hands it the `tasks` list, so the HTML file can loop over it and display each one.

### Route: add a task (`/add`)

```python
@app.route("/add", methods=["POST"])
def add_task():
    title = request.form.get("title", "").strip()
    if title:
        conn = get_db_connection()
        conn.execute("INSERT INTO tasks (title, done) VALUES (?, 0)", (title,))
        conn.commit()
        conn.close()
    return redirect(url_for("index"))
```

- `methods=["POST"]` means this route only accepts form submissions (POST requests), not regular page visits (GET).
- `request.form.get("title", "")` reads the text typed into the form's input box (the input is named `title` in `index.html`). `.strip()` removes accidental leading/trailing spaces.
- `if title:` skips inserting if the box was left empty.
- The `?` in the SQL is a placeholder — the actual value (`title`) is passed in separately as a tuple `(title,)`. This is done specifically to avoid SQL injection (never insert user input directly into a SQL string).
- `redirect(url_for("index"))` sends the browser back to the homepage (`/`) after adding, so the new task shows up in the list.

### Route: mark a task done/undone (`/complete/<id>`)

```python
@app.route("/complete/<int:task_id>", methods=["POST"])
def complete_task(task_id):
    conn = get_db_connection()
    task = conn.execute("SELECT done FROM tasks WHERE id = ?", (task_id,)).fetchone()
    if task is not None:
        new_status = 0 if task["done"] else 1
        conn.execute("UPDATE tasks SET done = ? WHERE id = ?", (new_status, task_id))
        conn.commit()
    conn.close()
    return redirect(url_for("index"))
```

- `<int:task_id>` in the route path means Flask expects a number in the URL (e.g. `/complete/3`) and passes it into the function as `task_id`.
- It first looks up the task's current `done` value.
- `new_status = 0 if task["done"] else 1` flips the value: if it's currently done (`1`), set it to not-done (`0`), and vice versa. This is why the button can say either "Done" or "Undo".
- `UPDATE ... WHERE id = ?` changes only that one task.

### Route: delete a task (`/delete/<id>`)

```python
@app.route("/delete/<int:task_id>", methods=["POST"])
def delete_task(task_id):
    conn = get_db_connection()
    conn.execute("DELETE FROM tasks WHERE id = ?", (task_id,))
    conn.commit()
    conn.close()
    return redirect(url_for("index"))
```

- Straightforward: deletes the row matching that `id`, then redirects back to the homepage.

### Starting the app

```python
if __name__ == "__main__":
    init_db()
    app.run(debug=True)
```

- `if __name__ == "__main__":` means this code only runs when you execute `python app.py` directly (not if the file is imported elsewhere).
- `init_db()` makes sure the table exists before the app starts handling requests.
- `app.run(debug=True)` starts the local web server on `http://127.0.0.1:5000`. `debug=True` auto-reloads the server when you edit code, and shows detailed errors in the browser — useful while developing, not something you'd leave on for a real public app.

---

## 2. `templates/index.html` — the page

```html
<form action="{{ url_for('add_task') }}" method="POST" class="add-form">
    <input type="text" name="title" placeholder="What do you need to do?" required>
    <button type="submit">Add</button>
</form>
```

- This form submits to the `/add` route when you click "Add" (or press Enter).
- `name="title"` is what makes `request.form.get("title", ...)` work in `app.py` — the names have to match.
- `{{ url_for('add_task') }}` is Jinja templating syntax — instead of hardcoding `/add`, it asks Flask "what's the URL for the `add_task` function?" This means if you ever renamed the route, the link would still work automatically.

```html
{% if tasks %}
<ul class="task-list">
    {% for task in tasks %}
    <li class="{{ 'done' if task['done'] else '' }}">
        <span class="task-title">{{ task['title'] }}</span>
        ...
    </li>
    {% endfor %}
</ul>
{% else %}
<p class="empty-msg">No tasks yet. Add one above!</p>
{% endif %}
```

- `{% if tasks %} ... {% else %} ... {% endif %}` — Jinja's version of an if/else statement, used here to show a friendly message when the list is empty instead of a blank page.
- `{% for task in tasks %} ... {% endfor %}` loops over every task passed in from `index()` in `app.py` and prints one `<li>` per task.
- `class="{{ 'done' if task['done'] else '' }}"` adds the CSS class `done` to the `<li>` only if that task is marked complete — this is what triggers the strikethrough style.
- Each task has two more tiny forms (Done/Undo and Delete), each pointing at `/complete/<id>` or `/delete/<id>` with that specific task's `id` filled in.

---

## 3. `static/style.css` — the look

Nothing logically complex here — just visual styling:

- `body` has a purple-blue gradient background (`linear-gradient`) and centers the white card on the page.
- `.container` is that white card: rounded corners (`border-radius`), a soft shadow (`box-shadow`), and a max width so it doesn't stretch too wide on big screens.
- `.task-list li.done .task-title` is what actually applies `text-decoration: line-through` — but only to tasks that have the `done` class, which (as shown above) is only added when `task['done']` is true.
- `.btn-complete` (green) and `.btn-delete` (red) are just colored differently so the two actions are visually easy to tell apart.

---

## Big picture: what happens when you click a button?

1. Browser sends a POST request to a route (e.g. `/complete/3`).
2. Flask runs the matching Python function, which reads/writes `tasks.db`.
3. That function ends with `redirect(url_for("index"))`, sending the browser back to `/`.
4. The `index()` function re-reads *all* tasks from the database (now updated) and re-renders the page.
5. You see the updated list.

This "do the action, then redirect and re-render everything" pattern is the simplest way to build a web app — no JavaScript needed, the page just reloads with fresh data every time.
