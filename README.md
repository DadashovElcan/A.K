# A.K
A.K – dəvətlə girişli əlaqə platformas
ak-app/
├── main.py
├── models.py
├── database.py
├── auth.py
├── invite.py
from sqlalchemy import Column, Integer, String, Boolean, DateTime, ForeignKey
from sqlalchemy.orm import relationship
from datetime import datetime
from database import Base

class User(Base):
    __tablename__ = 'users'

    id = Column(Integer, primary_key=True, index=True)
    email = Column(String, unique=True, index=True)
    hashed_password = Column(String)
    role = Column(String, default='üzv')
    is_active = Column(Boolean, default=True)

class Invite(Base):
    __tablename__ = 'invites'

    id = Column(Integer, primary_key=True, index=True)
    email = Column(String, index=True)
    role = Column(String)
    token = Column(String, unique=True, index=True)
    is_used = Column(Boolean, default=False)
    created_at = Column(DateTime, default=datetime.utcnow)
import uuid
from fastapi import APIRouter, HTTPException
from database import SessionLocal
from models import Invite
from email_sender import send_invite_email

router = APIRouter()

@router.post("/invite/")
def create_invite(email: str, role: str):
    if role not in ["üzv", "moderator", "kapitan"]:
        raise HTTPException(status_code=400, detail="Yanlış rol")

    db = SessionLocal()
    token = str(uuid.uuid4())
    invite = Invite(email=email, role=role, token=token)
    db.add(invite)
    db.commit()
    db.refresh(invite)

    invite_url = f"http://localhost:8000/register/{token}"
    send_invite_email(email, invite_url)

    return {"message": f"{email} üçün dəvət göndərildi"}
from fastapi import APIRouter, HTTPException
from models import User, Invite
from database import SessionLocal
from passlib.hash import bcrypt
from pydantic import BaseModel

router = APIRouter()

class RegisterModel(BaseModel):
    email: str
    password: str

@router.post("/register/{token}")
def register_user(token: str, data: RegisterModel):
    db = SessionLocal()
    invite = db.query(Invite).filter_by(token=token, is_used=False).first()

    if not invite or invite.email != data.email:
        raise HTTPException(status_code=400, detail="Dəvət linki etibarsızdır")

    hashed_pw = bcrypt.hash(data.password)
    user = User(email=data.email, hashed_password=hashed_pw, role=invite.role)
    db.add(user)
    invite.is_used = True
    db.commit()

    return {"message": "Qeydiyyat uğurlu oldu"}
import smtplib
from email.message import EmailMessage

def send_invite_email(to_email, invite_url):
    msg = EmailMessage()
    msg.set_content(f"A.K proqramına dəvət alırsınız. Qeydiyyat üçün keçid: {invite_url}")
    msg["Subject"] = "A.K Dəvəti"
    msg["From"] = "noreply@akprogram.com"
    msg["To"] = to_email

    # Sadə SMTP test serveri üçün
    with smtplib.SMTP("localhost", 1025) as server:
        server.send_message(msg)
<!-- register.html -->
<form action="/register/<token>" method="POST">
  <label for="email">E-poçt:</label>
  <input type="email" name="email" required />
  <label for="password">Şifrə:</label>
  <input type="password" name="password" required />
  <button type="submit">Qeydiyyatdan keç</button>
</form>
