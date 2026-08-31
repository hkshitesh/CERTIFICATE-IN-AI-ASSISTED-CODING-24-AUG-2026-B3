## Google Drive Link

https://drive.google.com/drive/folders/13e_ZrLS8XVh_MnUU0Y-8Z9zXaLbOCaVG?usp=sharing


## Important Commands

python3 -m venv .venv

source .venv/bin/activate

python -c "from app.db import init_db; init_db(); print('tables created')"

python -c "import sqlite3,os; c=sqlite3.connect('expenseflow.db'); print(c.execute(\"SELECT sql FROM sqlite_master WHERE name='expenses'\").fetchone()[0])"

## Important Git Commands

  git init
  git add .
  git config --global user.name "Your Name"
  git config --global user.email "your.email@example.com"
  git commit -m "final project"


