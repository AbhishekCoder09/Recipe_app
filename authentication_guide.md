# Authentication System Documentation

This document provides a comprehensive guide to the authentication system implemented in the Recipe App, designed for beginners.

## Table of Contents
1. [Overview](#overview)
2. [Installation & Setup](#installation--setup)
3. [How Authentication Works](#how-authentication-works)
4. [Code Explanation](#code-explanation)
5. [User Guide](#user-guide)
6. [Security Features](#security-features)
7. [Troubleshooting](#troubleshooting)

---

## Overview

The Recipe App now includes a complete user authentication system that allows users to:
- **Register** a new account with username, email, and password
- **Login** to access the recipe search functionality
- **Logout** to end their session
- **Protected Routes**: Only logged-in users can search and view recipes

### Technologies Used
- **Flask-Login**: Manages user sessions (keeps track of who is logged in)
- **Flask-SQLAlchemy**: Database management (stores user information)
- **Werkzeug**: Password hashing (encrypts passwords for security)
- **SQLite**: Database file (stores all user accounts)

---

## Installation & Setup

### Step 1: Install Dependencies
Open your terminal in the project folder and run:

```bash
pip install -r requirements.txt
```

This installs:
- `Flask-Login==0.6.2` - Session management
- `Flask-SQLAlchemy==3.0.5` - Database ORM
- `email-validator==2.0.0` - Email validation

### Step 2: Run the Application
```bash
python app.py
```

The first time you run the app:
1. A file called `users.db` will be created automatically in your project folder
2. This database file stores all user accounts
3. The app will start at `http://127.0.0.1:5000`

### Step 3: Access the App
Open your browser and go to:
```
http://127.0.0.1:5000
```

You'll be redirected to the login page since you're not logged in yet.

---

## How Authentication Works

### The Big Picture

Think of authentication like a **members-only club**:

1. **Registration** = Signing up for membership
2. **Login** = Showing your membership card at the door
3. **Session** = The wristband they give you so you don't have to show your card every time
4. **Logout** = Returning your wristband and leaving

### The Technical Flow

#### Registration Flow
```
User fills form → Server validates data → Password gets hashed → 
User saved to database → Redirect to login page
```

#### Login Flow
```
User enters credentials → Server finds user in database → 
Password checked → Session created → User redirected to home page
```

#### Protected Routes
```
User tries to access /home → Flask-Login checks session → 
If logged in: Show page | If not logged in: Redirect to login
```

---

## Code Explanation

### File Structure
```
recipe_app/
├── app.py                 # Main application with authentication routes
├── models.py              # Database models (User table)
├── requirements.txt       # Dependencies
├── users.db              # SQLite database (auto-generated)
└── templates/
    ├── login.html        # Login page
    ├── register.html     # Registration page
    ├── index.html        # Recipe search (updated with auth UI)
    └── view_recipe.html  # Recipe details (updated with auth UI)
```

### models.py - Database Model

```python
from flask_sqlalchemy import SQLAlchemy
from flask_login import UserMixin
from werkzeug.security import generate_password_hash, check_password_hash

db = SQLAlchemy()

class User(UserMixin, db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    password_hash = db.Column(db.String(200), nullable=False)
```

**What this does:**
- `UserMixin`: Provides default methods Flask-Login needs (like `is_authenticated`, `is_active`)
- `db.Model`: Makes this class a database table
- `id`: Unique identifier for each user (auto-increments)
- `username`: User's chosen name (must be unique)
- `email`: User's email (must be unique)
- `password_hash`: Encrypted password (NEVER store plain passwords!)

**Password Methods:**

```python
def set_password(self, password):
    self.password_hash = generate_password_hash(password)
```
- Takes a plain text password (e.g., "mypassword123")
- Converts it to a hash (e.g., "pbkdf2:sha256:...")
- Stores the hash in the database

```python
def check_password(self, password):
    return check_password_hash(self.password_hash, password)
```
- Takes a plain text password attempt
- Compares it with the stored hash
- Returns `True` if they match, `False` otherwise

### app.py - Authentication Routes

#### Configuration
```python
app.config['SECRET_KEY'] = 'your-secret-key-change-this-in-production'
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///users.db'
```

- `SECRET_KEY`: Used to encrypt session cookies (change this for production!)
- `SQLALCHEMY_DATABASE_URI`: Location of the database file

#### User Loader
```python
@login_manager.user_loader
def load_user(user_id):
    return User.query.get(int(user_id))
```

**What this does:**
- Flask-Login stores the user ID in the session cookie
- Every time a user makes a request, this function loads their full User object
- This is how `current_user` knows who you are

#### Register Route (`/register`)

```python
@app.route('/register', methods=['GET', 'POST'])
def register():
    if current_user.is_authenticated:
        return redirect(url_for('index'))
```

**Step-by-step:**
1. If user is already logged in, redirect them to home
2. If GET request: Show the registration form
3. If POST request:
   - Get form data (username, email, password)
   - Validate:
     - All fields filled?
     - Passwords match?
     - Password at least 6 characters?
     - Username/email not already taken?
   - Create new User object
   - Hash the password with `set_password()`
   - Save to database
   - Redirect to login page

#### Login Route (`/login`)

```python
@app.route('/login', methods=['GET', 'POST'])
def login():
    if current_user.is_authenticated:
        return redirect(url_for('index'))
```

**Step-by-step:**
1. If user is already logged in, redirect them to home
2. If GET request: Show the login form
3. If POST request:
   - Get email and password from form
   - Find user in database by email
   - Check if password is correct using `check_password()`
   - If valid:
     - Call `login_user(user)` - creates session
     - Redirect to home page
   - If invalid:
     - Show error message

#### Logout Route (`/logout`)

```python
@app.route('/logout')
@login_required
def logout():
    logout_user()
    flash('You have been logged out.', 'success')
    return redirect(url_for('login'))
```

**What this does:**
- `@login_required`: Only logged-in users can access this
- `logout_user()`: Destroys the session
- Redirects to login page

#### Protected Routes

```python
@app.route('/')
@login_required
def index():
    # ... recipe search code
```

**The `@login_required` decorator:**
- Checks if user is logged in
- If yes: Execute the function
- If no: Redirect to login page (set by `login_manager.login_view = 'login'`)

### Frontend Templates

#### Flash Messages
```html
{% with messages = get_flashed_messages(with_categories=true) %}
    {% if messages %}
        {% for category, message in messages %}
            <div class="message {{ category }}">{{ message }}</div>
        {% endfor %}
    {% endif %}
{% endwith %}
```

**What this does:**
- `flash()` in Python sends a message to the next page
- This template code displays those messages
- Categories: `'error'` (red) or `'success'` (green)

#### Current User Display
```html
{% if current_user.is_authenticated %}
    <span>Welcome, {{ current_user.username }}!</span>
    <a href="{{ url_for('logout') }}">Logout</a>
{% endif %}
```

**What this does:**
- `current_user` is available in all templates
- `is_authenticated` checks if someone is logged in
- Shows username and logout link if logged in

---

## User Guide

### How to Register
1. Go to `http://127.0.0.1:5000`
2. Click "Register here"
3. Fill in:
   - **Username**: Your display name (must be unique)
   - **Email**: Your email address (must be unique)
   - **Password**: At least 6 characters
   - **Confirm Password**: Must match password
4. Click "Register"
5. You'll be redirected to login page

### How to Login
1. Enter your email and password
2. Click "Login"
3. You'll see "Welcome, [username]!" at the top
4. Now you can search for recipes

### How to Logout
1. Click the "Logout" link at the top of any page
2. You'll be logged out and redirected to login page

---

## Security Features

### 1. Password Hashing
**What it is:** Passwords are encrypted before storing in the database.

**Why it matters:** If someone steals the database, they can't see actual passwords.

**Example:**
- You enter: `mypassword123`
- Database stores: `pbkdf2:sha256:260000$abc123...`

### 2. Session Management
**What it is:** Flask-Login uses encrypted cookies to remember who you are.

**Why it matters:** You don't have to login on every page.

**How it works:**
- When you login, a cookie is created
- Every request sends this cookie
- Server verifies the cookie and loads your user data

### 3. Protected Routes
**What it is:** `@login_required` decorator blocks unauthenticated access.

**Why it matters:** Only registered users can use the app.

### 4. Input Validation
**What it is:** Server checks all form data before processing.

**Checks include:**
- All fields filled?
- Email format valid?
- Password long enough?
- Username/email unique?

---

## Troubleshooting

### Problem: "ModuleNotFoundError: No module named 'flask_login'"
**Solution:** Install dependencies:
```bash
pip install -r requirements.txt
```

### Problem: "sqlite3.OperationalError: no such table: user"
**Solution:** Delete `users.db` and restart the app. The database will be recreated.

### Problem: Can't login with correct password
**Solution:** 
1. Make sure you registered the account first
2. Check if you're using the correct email (not username)
3. Password is case-sensitive

### Problem: "Please log in to access this page"
**Solution:** This is normal! The app requires login. Go to `/login` and sign in.

### Problem: Database file location
**Location:** The `users.db` file is in your project root:
```
c:\Users\priya\OneDrive\Desktop\recipe_app\users.db
```

You can delete this file to reset all users (useful for testing).

---

## What Happens When...

### Scenario 1: User Registers
1. **Frontend**: User fills registration form, clicks "Register"
2. **Browser**: Sends POST request to `/register` with form data
3. **Backend (`app.py`)**:
   - Receives data
   - Validates all fields
   - Checks if username/email already exists
   - Creates new User object
   - Hashes password with `set_password()`
   - Saves to database: `db.session.add(new_user)` then `db.session.commit()`
4. **Database**: New row added to `user` table
5. **Response**: Redirect to login page with success message

### Scenario 2: User Logs In
1. **Frontend**: User enters email/password, clicks "Login"
2. **Browser**: Sends POST request to `/login`
3. **Backend**:
   - Queries database: `User.query.filter_by(email=email).first()`
   - Checks password: `user.check_password(password)`
   - If valid: `login_user(user)` creates session
4. **Session**: Cookie created with encrypted user ID
5. **Response**: Redirect to home page

### Scenario 3: User Accesses Protected Page
1. **Browser**: Requests `/` (home page)
2. **Flask-Login**: Checks session cookie
3. **If logged in**:
   - Calls `load_user()` to get User object
   - Sets `current_user` to that user
   - Allows access to page
4. **If not logged in**:
   - Redirects to `/login`
   - Adds `?next=/` to URL (so after login, user goes back to home)

### Scenario 4: User Logs Out
1. **Frontend**: User clicks "Logout"
2. **Browser**: Sends GET request to `/logout`
3. **Backend**: 
   - `logout_user()` destroys session
   - Deletes session cookie
4. **Response**: Redirect to login page

---

## Database Schema

### User Table
| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | Integer | Primary Key, Auto-increment | Unique user ID |
| username | String(80) | Unique, Not Null | User's display name |
| email | String(120) | Unique, Not Null | User's email address |
| password_hash | String(200) | Not Null | Hashed password |

**Example Data:**
```
id | username | email              | password_hash
---+----------+--------------------+--------------------------------
1  | john_doe | john@example.com   | pbkdf2:sha256:260000$abc123...
2  | jane_s   | jane@example.com   | pbkdf2:sha256:260000$def456...
```

---

## Summary

You now have a fully functional authentication system! Here's what you learned:

1. **User Registration**: Creates accounts with hashed passwords
2. **User Login**: Validates credentials and creates sessions
3. **Session Management**: Keeps users logged in across pages
4. **Protected Routes**: Restricts access to authenticated users only
5. **Security**: Passwords are hashed, sessions are encrypted

The authentication system is production-ready for learning purposes. For a real production app, you'd want to add:
- Email verification
- Password reset functionality
- "Remember me" option
- Rate limiting (prevent brute force attacks)
- HTTPS (encrypted connections)
