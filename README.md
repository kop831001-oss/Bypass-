from flask import Flask, request, jsonify, redirect, session
from flask_sqlalchemy import SQLAlchemy

app = Flask(__name__)
app.secret_key = "admin123"

# DB
app.config["SQLALCHEMY_DATABASE_URI"] = "sqlite:///db.sqlite3"
db = SQLAlchemy(app)

# Model
class License(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    key = db.Column(db.String(50), unique=True)
    active = db.Column(db.Boolean, default=True)

# Create DB
with app.app_context():
    db.create_all()

# ================= API =================
@app.route("/verify", methods=["POST"])
def verify():
    data = request.json
    key = data.get("license_key")

    lic = License.query.filter_by(key=key).first()

    if not lic:
        return jsonify({"status": "invalid"})
    if not lic.active:
        return jsonify({"status": "disabled"})

    return jsonify({"status": "valid"})


# ================= ADMIN =================
@app.route("/admin", methods=["GET", "POST"])
def admin():
    if request.method == "POST":
        if request.form["user"] == "admin" and request.form["pass"] == "1234":
            session["login"] = True
            return redirect("/panel")

    return """
    <h3>Login</h3>
    <form method="post">
    <input name="user" placeholder="user"><br>
    <input name="pass" type="password"><br>
    <button>Login</button>
    </form>
    """

@app.route("/panel")
def panel():
    if not session.get("login"):
        return redirect("/admin")

    data = License.query.all()

    html = "<h3>Licenses</h3><a href='/add'>Add</a><br><br>"
    for l in data:
        html += f"{l.key} | {l.active} <a href='/toggle/{l.id}'>[Toggle]</a><br>"

    return html

@app.route("/add", methods=["GET", "POST"])
def add():
    if not session.get("login"):
        return redirect("/admin")

    if request.method == "POST":
        k = request.form["key"]
        db.session.add(License(key=k))
        db.session.commit()
        return redirect("/panel")

    return """
    <form method="post">
    <input name="key" placeholder="License key">
    <button>Add</button>
    </form>
    """

@app.route("/toggle/<int:id>")
def toggle(id):
    if not session.get("login"):
        return redirect("/admin")

    lic = License.query.get(id)
    lic.active = not lic.active
    db.session.commit()
    return redirect("/panel")


# ================= RUN =================
app.run()
