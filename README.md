# My To-Do List App

A simple to-do list web app built with Python (Flask) and SQLite. Add tasks, mark them done, delete them — all saved to a local database file so nothing is lost when you close the server.

## Project structure

```
sample own proj/
├── app.py              # Flask backend — routes and database logic
├── requirements.txt    # Python packages needed (just Flask)
├── templates/
│   └── index.html      # The page layout (HTML + Jinja)
├── static/
│   └── style.css       # All the styling
└── tasks.db            # SQLite database (created automatically on first run)
```

## Setup (first time only)

1. Make sure Python is installed.
2. Install Flask:
   ```
   pip install -r requirements.txt
   ```

## How to run

```
cd "sample own proj"
python app.py
```

Then open a browser and go to:

```
http://127.0.0.1:5000
```

The server keeps running in that terminal window. Close the terminal (or press `Ctrl+C`) to stop it. You need to run `python app.py` again each time you want to use the app.

## How to use it

- Type a task in the input box and click **Add**.
- Click **Done** to mark a task complete (it gets a strikethrough). Click it again (**Undo**) to un-complete it.
- Click **Delete** to remove a task permanently.

## How it works (short version)

- Flask handles web requests and decides what HTML to send back.
- SQLite (`tasks.db`) stores the tasks so they persist between restarts.
- Every button on the page is its own small form that submits to a specific route (`/add`, `/complete/<id>`, `/delete/<id>`), which is why the page reloads after every click.

For a full line-by-line walkthrough of the code, see [CODE_WALKTHROUGH.md](CODE_WALKTHROUGH.md).
