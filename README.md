from flask import Flask, render_template, request, redirect, session, flash
from flask_sqlalchemy import SQLAlchemy
from datetime import datetime

app = Flask(__name__)
app.secret_key = "secret123"
app.config["SQLALCHEMY_DATABASE_URI"] = "sqlite:///database.db"
db = SQLAlchemy(app)

# -------------------------
# MODELS
# -------------------------
class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100))
    email = db.Column(db.String(100), unique=True)
    password = db.Column(db.String(100))

class Doctor(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100))
    specialty = db.Column(db.String(100))

class Appointment(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'))
    doctor_id = db.Column(db.Integer, db.ForeignKey('doctor.id'))
    date = db.Column(db.String(100))
    time = db.Column(db.String(100))
    created_at = db.Column(db.DateTime, default=datetime.utcnow)

# -------------------------
# ROUTES
# -------------------------
@app.route('/')
def index():
    return render_template('index.html')

@app.route('/register', methods=['GET', 'POST'])
def register():
    if request.method == 'POST':
        name = request.form['name']
        email = request.form['email']
        password = request.form['password']
        if User.query.filter_by(email=email).first():
            flash("Email already exists!", "danger")
            return redirect('/register')
        user = User(name=name, email=email, password=password)
        db.session.add(user)
        db.session.commit()
        flash("Registration successful! Please login.", "success")
        return redirect('/login')
    return render_template('register.html')

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        email = request.form['email']
        password = request.form['password']
        user = User.query.filter_by(email=email, password=password).first()
        if user:
            session['user_id'] = user.id
            return redirect('/dashboard')
        else:
            flash("Invalid credentials!", "danger")
    return render_template('login.html')

@app.route('/dashboard')
def dashboard():
    if 'user_id' not in session:
        return redirect('/login')
    doctors = Doctor.query.all()
    user = User.query.get(session['user_id'])
    appointments = Appointment.query.filter_by(user_id=user.id).all()
    return render_template('dashboard.html', user=user, doctors=doctors, appointments=appointments)

@app.route('/book/<int:doctor_id>', methods=['GET', 'POST'])
def book(doctor_id):
    if 'user_id' not in session:
        return redirect('/login')
    doctor = Doctor.query.get(doctor_id)
    if request.method == 'POST':
        date = request.form['date']
        time = request.form['time']
        appt = Appointment(user_id=session['user_id'], doctor_id=doctor.id, date=date, time=time)
        db.session.add(appt)
        db.session.commit()
        flash("Appointment booked successfully!", "success")
        return redirect('/dashboard')
    return render_template('book.html', doctor=doctor)

@app.route('/logout')
def logout():
    session.pop('user_id', None)
    return redirect('/')

# -------------------------
# INITIALIZE DATABASE
# -------------------------
if __name__ == '__main__':
    with app.app_context():
        db.create_all()

        # Add sample doctors if not exist
        if Doctor.query.count() == 0:
            db.session.add_all([
                Doctor(name="Dr. Alice Smith", specialty="Cardiology"),
                Doctor(name="Dr. John Doe", specialty="Dermatology"),
                Doctor(name="Dr. Sarah Lee", specialty="Pediatrics")
            ])
            db.session.commit()

    app.run(debug=True)
