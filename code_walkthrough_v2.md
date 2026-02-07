# Code Walkthrough: Recipe App (with Authentication)

This guide explains your entire application code line-by-line. It is written for a beginner and includes analogies to help you understand how everything connects.

---

## 1. The Big Picture: How It All Works

Imagine your app is a **private club with a library**:

1.  **The Bouncer (Authentication)**: You can't enter without a membership card (Login). If you don't have one, you must sign up (Register).
2.  **The Librarian (Backend/Flask)**: Once inside, you ask the librarian for a book (Search for "Pasta").
3.  **The Archive (Database)**: The librarian keeps a list of all members here.
4.  **The Partner Library (Spoonacular API)**: The librarian doesn't have the books themselves, so they call a partner library to get the recipe details.
5.  **The Display (Frontend)**: The librarian hands you the book formatted nicely on a page (HTML).

---

## 2. The Backend (`app.py`) - The Brain

This file controls everything. It runs on the server.

### Essential Imports
```python
1: from flask import Flask, render_template, request, redirect, url_for, flash
2: import requests
3: from urllib.parse import unquote
4: from flask_login import LoginManager, login_user, logout_user, login_required, current_user
5: from models import db, User
```
*   **Flask tools**:
    *   `render_template`: Loads HTML files to show to the user.
    *   `request`: Contains data sent *by* the user (e.g., what they typed in a form).
    *   `redirect`, `url_for`: Send the user to a different page (e.g., sending them to login).
    *   `flash`: Sends a temporary message to the next page (e.g., "Login Successful").
*   **flask_login tools**: These manage the "membership card" (session).
    *   `login_user`: Gives the user a card.
    *   `logout_user`: Takes the card away.
    *   `login_required`: The "Bouncer" that stops non-members.
    *   `current_user`: The person holding the card right now.
*   **models**: Imports your Database setup (`db`) and the `User` definition.

### Configuration
```python
10: app.config['SECRET_KEY'] = 'your-secret-key...'
11: app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///users.db'
```
*   **Line 10**: A secret password the app uses to sign the membership cards (cookies) so they can't be faked.
*   **Line 11**: Tells the app where to find the file that stores all the user accounts (`users.db`).

### Setup & The "User Loader"
```python
23: @login_manager.user_loader
24: def load_user(user_id):
29:     return User.query.get(int(user_id))
```
*   **What is this?** When a user visits a page, their browser shows a "membership card" (cookie) with their ID number. Flask needs to know *who* that ID belongs to.
*   **Line 29**: It looks up that ID in the database and returns the User object so we can say `current_user.username`.

### Registration Route (`/register`)
```python
33: @app.route('/register', methods=['GET', 'POST'])
34: def register():
```
This function handles signing up.

*   **GET** request: The user just *visited* the page. We show them the empty form (Line 82).
*   **POST** request: The user filled out the form and clicked "Submit".

```python
73:         new_user = User(username=username, email=email)
74:         new_user.set_password(password)
76:         db.session.add(new_user)
77:         db.session.commit()
```
*   **Line 73**: We create a new User in memory.
*   **Line 74**: **Crucial Security Step!** We don't save "password123". We scramble it into "pbkdf2:sha256:..." so even if hackers steal the database, they can't read the passwords.
*   **Line 76-77**: We save it to the database file permanently.

### Login Route (`/login`)
```python
84: @app.route('/login', methods=['GET', 'POST'])
```
This checks if the user is who they say they are.

```python
104:         user = User.query.filter_by(email=email).first()
106:         if user and user.check_password(password):
108:             login_user(user)
```
*   **Line 104**: Checks if that email exists in our "Archive" (Database).
*   **Line 106**: Checks if the password matches the scrambled one we saved.
*   **Line 108**: `login_user(user)` issues the "membership card" (session cookie) effectively logging them in.

### Protected Routes (The "Bouncer")
```python
133: @login_required
134: def home():
```
*   **Line 133**: `@login_required` is a **decorator**. It sits atop the function like a guard. 
*   **How it works**: If you try to visit `/home`, this guard checks "Do you have a membership card?". 
    *   **Yes**: Run the `home()` function.
    *   **No**: Kick you out to the login page immediately.

---

## 3. The Database Model (`models.py`)

This file defines what a "User" actually looks like in our database.

```python
7: class User(UserMixin, db.Model):
14:     id = db.Column(db.Integer, primary_key=True)
15:     username = db.Column(db.String(80), unique=True, nullable=False)
...
17:     password_hash = db.Column(db.String(200), nullable=False)
```
*   **Class**: Think of this as a blueprint. Every user will have these exact fields.
*   **Columns**:
    *   `id`: A unique number (1, 2, 3...). The database manages this automatically.
    *   `username`: Must be unique (no two people can be "john_doe").
    *   `password_hash`: The scrambled password. Note we **do not** have a `password` column!

---

## 4. The Frontend (`templates/`)

This is what the user sees.

### Logic in HTML? (`index.html`)
We use a template language called **Jinja2**. It allows us to put Python-like logic inside HTML.

**Showing User Info:**
```html
87:         {% if current_user.is_authenticated %}
88:         <span>Welcome, {{ current_user.username }}!</span>
90:         {% endif %}
```
*   **Line 87**: Checks if the user is logged in. 
*   If they are, it prints their username. This is dynamic—it changes based on who is viewing the page!

**Showing Messages (Flash):**
```html
94:     {% with messages = get_flashed_messages(with_categories=true) %}
```
*   Remember when we did `flash('Invalid email!')` in Python? This block catches that message and displays it as a nice red or green box on the screen.

---

## 5. How Frontend & Backend Connect (The "If This, Then That")

### Scenario A: You try to login
1.  **Frontend**: You type your email/password in `login.html` and click "Login".
2.  **Browser**: Sends a **POST** request to the server at `/login` with your data inside.
3.  **Backend (`app.py`)**:
    *   Catches the request at `@app.route('/login')`.
    *   Reads `request.form['email']` and `request.form['password']`.
    *   Asks the Database (`User.query...`) "Do we have this guy?".
    *   Says "Yes, password matches".
    *   Calls `login_user()`.
    *   Sends a response: "Redirect to `/home`".
4.  **Browser**: Receives "Redirect". It immediately loads `/home`.
5.  **Frontend**: `index.html` loads. It sees you are `current_user` and displays "Welcome, [You]!".

### Scenario B: You search for "Pizza"
1.  **Frontend**: You type "Pizza" in `index.html` and hit enter.
2.  **Browser**: Sends a **POST** request to `/`.
3.  **Backend**:
    *   First, `@login_required` checks your session. You're logged in, so it lets you pass.
    *   `search_recipes("Pizza")` is called.
    *   **The Backend becomes a Client**: Python calls Spoonacular's API (another server).
    *   Spoonacular sends back a JSON list of pizza recipes.
    *   Python passes this list to `render_template('index.html', recipes=...)`.
4.  **Frontend**: The Flask template engine takes the `recipes` list and loops through it (`{% for recipe in recipes %}`), creating one `<li>` (list item) for every pizza found.
5.  **Result**: You see 10 pizza recipes on your screen.

### Scenario C: You try to enter without logging in
1.  **Browser**: You type `http://localhost:5000/home`.
2.  **Backend**:
    *   `@app.route('/home')` is triggered.
    *   `@login_required` jumps in: "Check for session cookie".
    *   **Result**: No cookie found.
    *   **Action**: Stop everything. Do not run `home()`. Return "Redirect to /login".
3.  **Frontend**: You suddenly see the login page.
