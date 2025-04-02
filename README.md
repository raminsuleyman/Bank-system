# Bank-system
import sqlite3
import hashlib
import secrets

# Məlumat bazasına qoşuluruq
conn = sqlite3.connect("bank_system.db")
cursor = conn.cursor()

# Cədvəllərin yaradılması
cursor.execute('''
CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    fullname TEXT,
    card_number TEXT UNIQUE,
    cvv TEXT,
    exp_date TEXT,
    currency TEXT,
    password TEXT,
    balance REAL DEFAULT 0.0,
    role TEXT DEFAULT "customer",
    loan_balance REAL DEFAULT 0.0
)
''')

cursor.execute('''
CREATE TABLE IF NOT EXISTS transactions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    sender TEXT,
    receiver TEXT,
    amount REAL,
    currency TEXT,
    commission REAL DEFAULT 0.0,
    date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    type TEXT
)
''')

conn.commit()

# Şifrəni hash-ləmək
def hash_password(password):
    return hashlib.sha256(password.encode()).hexdigest()

# 3D Secure OTP doğrulama
def generate_otp():
    return str(secrets.randbelow(9000) + 1000)  # 1000-9999 arası OTP kodu

# Qeydiyyat funksiyası
def register():
    fullname = input("Ad və soyadınızı daxil edin: ")
    card_number = str(secrets.randbelow(10**16)).zfill(16)  # 16 rəqəmli kart nömrəsi
    cvv = str(secrets.randbelow(900)).zfill(3)  # 3 rəqəmli CVV kodu
    exp_date = f"{secrets.randbelow(12) + 1}/{secrets.randbelow(6) + 25}"
    currency = input("Valyutanı daxil edin (AZN, USD, EUR): ")
    password = input("Şifrənizi daxil edin: ")
    role = input("İstifadəçi rolu (customer/manager): ").lower()

    hashed_password = hash_password(password)

    try:
        cursor.execute("INSERT INTO users (fullname, card_number, cvv, exp_date, currency, password, role) VALUES (?, ?, ?, ?, ?, ?, ?)", 
                       (fullname, card_number, cvv, exp_date, currency, hashed_password, role))
        conn.commit()
        print(f"Qeydiyyat tamamlandı! Kart nömrəniz: {card_number}, CVV: {cvv}, Exp: {exp_date}")
    except sqlite3.IntegrityError:
        print("Bu kart nömrəsi artıq mövcuddur. Yenidən cəhd edin!")

# Giriş funksiyası
def login():
    card_number = input("Kart nömrənizi daxil edin: ")
    password = input("Şifrənizi daxil edin: ")

    hashed_password = hash_password(password)  # Daxil edilən şifrəni hash-ləyirik

    cursor.execute("SELECT * FROM users WHERE card_number = ?", (card_number,))
    user = cursor.fetchone()

    if user:
        stored_password = user[6]  # 6-cı sütun password üçündür
        if hashed_password == stored_password:  # Daxil edilən hash-lənmiş şifrəni müqayisə edirik
            otp = generate_otp()
            print(f"Təsdiqləmə kodu: {otp}")
            user_otp = input("OTP kodunu daxil edin: ")

            if user_otp == otp:
                print(f"Xoş gəldiniz, {user[1]}!")
                return user
            else:
                print("Yanlış OTP kodu!")
        else:
            print("Şifrə səhvdir!")  
    else:
        print("Kart nömrəsi səhvdir!")

    return None

# Balansı yoxlamaq
def check_balance(user):
    cursor.execute("SELECT balance FROM users WHERE card_number = ?", (user[2],))
    balance = cursor.fetchone()[0]
    print(f"Balansınız: {balance} {user[5]}")

# Əsas menyu
def main():
    while True:
        print("\n1. Qeydiyyat")
        print("2. Giriş")
        print("3. Çıxış")

        choice = input("Seçim edin: ")

        if choice == "1":
            register()
        elif choice == "2":
            user = login()
            if user:
                while True:
                    print("\n1. Balansı yoxlamaq")
                    print("2. Çıxış")
                    sub_choice = input("Seçim edin: ")

                    if sub_choice == "1":
                        check_balance(user)
                    elif sub_choice == "2":
                        break
        elif choice == "3":
            print("Çıxış edilir...")
            break

if __name__ == "__main__":
    main()
