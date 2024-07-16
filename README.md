from flask import Flask, request, jsonify
from flask_sqlalchemy import SQLAlchemy
from datetime import datetime

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'postgresql://username:password@localhost/calendar'
db = SQLAlchemy(app)

class User(db.Model):
    user_id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(80), nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)

class Room(db.Model):
    room_id = db.Column(db.Integer, primary_key=True)
    room_name = db.Column(db.String(80), nullable=False)

class Meeting(db.Model):
    meeting_id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(80), nullable=False)
    description = db.Column(db.String(200), nullable=True)
    start_time = db.Column(db.DateTime, nullable=False)
    end_time = db.Column(db.DateTime, nullable=False)
    room_id = db.Column(db.Integer, db.ForeignKey('room.room_id'), nullable=False)
    organizer_id = db.Column(db.Integer, db.ForeignKey('user.user_id'), nullable=False)

class Attendee(db.Model):
    attendee_id = db.Column(db.Integer, primary_key=True)
    meeting_id = db.Column(db.Integer, db.ForeignKey('meeting.meeting_id'), nullable=False)
    user_id = db.Column(db.Integer, db.ForeignKey('user.user_id'), nullable=False)

@app.route('/users', methods=['POST'])
def create_user():
    data = request.json
    user = User(name=data['name'], email=data['email'])
    db.session.add(user)
    db.session.commit()
    return jsonify({"user_id": user.user_id, "name": user.name, "email": user.email})

@app.route('/rooms', methods=['POST'])
def create_room():
    data = request.json
    room = Room(room_name=data['room_name'])
    db.session.add(room)
    db.session.commit()
    return jsonify({"room_id": room.room_id, "room_name": room.room_name})

@app.route('/meetings', methods=['POST'])
def schedule_meeting():
    data = request.json
    start_time = datetime.fromisoformat(data['start_time'])
    end_time = datetime.fromisoformat(data['end_time'])
    
    # Collision detection logic here
    
    meeting = Meeting(
        title=data['title'],
        description=data['description'],
        start_time=start_time,
        end_time=end_time,
        room_id=data['room_id'],
        organizer_id=data['organizer_id']
    )
    db.session.add(meeting)
    db.session.commit()
    
    for attendee_id in data['attendees']:
        attendee = Attendee(meeting_id=meeting.meeting_id, user_id=attendee_id)
        db.session.add(attendee)
    
    db.session.commit()
    return jsonify({"meeting_id": meeting.meeting_id, "title": meeting.title, "start_time": meeting.start_time.isoformat(), "end_time": meeting.end_time.isoformat(), "room_id": meeting.room_id, "organizer_id": meeting.organizer_id, "attendees": data['attendees']})

if __name__ == '__main__':
    db.create_all()
    app.run(debug=True)
