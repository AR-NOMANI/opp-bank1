```python name=bank_system.py
import datetime
import time

class Bank:
    def __init__(self, name, acc_id, pin, balance=0, fraud_limit=50000, max_login_attempts=3):
        self.name = name
        self.acc_id = acc_id
        self._pin = str(pin)
        self.__balance = balance
        self.transactions = []
        self.fraud_limit = fraud_limit
        self.max_login_attempts = max_login_attempts
        self.login_attempts = 0
        self.login_locked = False
        self.fraud_locked = False
        self.login_lock_time = None
        self.fraud_lock_time = None
        self.logged_in = False

    def _check_login_lock(self):
        if self.login_locked and self.login_lock_time:
            if time.time() - self.login_lock_time >= 60:
                self.login_locked = False
                self.login_lock_time = None
                self.login_attempts = 0
        return self.login_locked

    def _check_fraud_lock(self):
        if self.fraud_locked and self.fraud_lock_time:
            if time.time() - self.fraud_lock_time >= 60:
                self.fraud_locked = False
                self.fraud_lock_time = None
        return self.fraud_locked

    def _validate_amount(self, amount):
        try:
            amount = float(amount)
        except (ValueError, TypeError):
            return False, "Invalid amount"
        
        if amount != amount:
            return False, "Invalid amount"
        
        if amount == float('inf') or amount == float('-inf'):
            return False, "Invalid amount"
        
        if amount <= 0:
            return False, "Amount must be positive"
        
        return True, amount

    def login(self, entered_pin):
        if self._check_login_lock():
            return "Account locked. Try again later"
        
        if self._check_fraud_lock():
            return "Account locked. Try again later"

        if str(entered_pin) == self._pin:
            self.logged_in = True
            self.login_attempts = 0
            return True

        self.login_attempts += 1

        if self.login_attempts >= self.max_login_attempts:
            self.login_locked = True
            self.login_lock_time = time.time()
            return "Account locked"

        remaining = self.max_login_attempts - self.login_attempts
        return f"Wrong PIN. Try again. Attempts left: {remaining}"

    def logout(self):
        self.logged_in = False
        return "Logged out"

    def _auth(self):
        self._check_login_lock()
        self._check_fraud_lock()
        
        if self.login_locked or self.fraud_locked:
            return False
        
        return self.logged_in

    def _fraud_check(self, amount):
        if amount > self.fraud_limit:
            return True

        current_time = time.time()
        recent_withdrawals = [t for t in self.transactions 
                            if t["type"] == "Withdraw" and 
                            (current_time - t["time"].timestamp()) < 60]
        
        total_recent = sum(t["amount"] for t in recent_withdrawals)
        
        if total_recent + amount > self.fraud_limit * 2:
            return True
        
        recent_all = [t for t in self.transactions 
                     if (current_time - t["time"].timestamp()) < 60]
        
        if len(recent_all) >= 4:
            return True
        
        return False

    def deposit(self, amount):
        if not self._auth():
            return "Login required"

        valid, msg = self._validate_amount(amount)
        if not valid:
            return msg

        amount = float(amount)
        self.__balance += amount
        self.transactions.append({
            "type": "Deposit",
            "amount": amount,
            "time": datetime.datetime.now()
        })
        return f"Deposit successful. Balance: {self.__balance}"

    def withdraw(self, amount):
        if not self._auth():
            return "Login required"

        valid, msg = self._validate_amount(amount)
        if not valid:
            return msg

        amount = float(amount)

        if amount > self.__balance:
            return "Insufficient balance"

        if self._fraud_check(amount):
            self.fraud_locked = True
            self.fraud_lock_time = time.time()
            return "Transaction blocked. Account locked"

        self.__balance -= amount
        self.transactions.append({
            "type": "Withdraw",
            "amount": amount,
            "time": datetime.datetime.now()
        })
        return f"Withdrawal successful. Balance: {self.__balance}"

    def show_transactions(self, t_type=None):
        if not self._auth():
            return "Login required"

        filtered = self.transactions
        
        if t_type:
            filtered = [t for t in self.transactions if t["type"] == t_type]

        if not filtered:
            print("No transactions")
            return []

        result = []
        for i, t in enumerate(filtered, 1):
            line = f"{i}. {t['type']} {t['amount']} {t['time'].strftime('%Y-%m-%d %H:%M:%S')}"
            print(line)
            result.append(line)
        
        return result

    def check_balance(self):
        if not self._auth():
            return "Login required"
        
        return self.__balance

    def generate_report(self):
        if not self._auth():
            return "Login required"

        total_deposit = sum(t["amount"] for t in self.transactions if t["type"] == "Deposit")
        total_withdraw = sum(t["amount"] for t in self.transactions if t["type"] == "Withdraw")

        status = "Active"
        if self.fraud_locked:
            status = "Locked"
        elif self.login_locked:
            status = "Locked"

        return {
            "name": self.name,
            "account": self.acc_id,
            "total_deposit": total_deposit,
            "total_withdraw": total_withdraw,
            "balance": self.__balance,
            "status": status
        }

    def unlock_login(self, admin_pin):
        if str(admin_pin) != self._pin:
            return "Invalid PIN"
        
        self.login_locked = False
        self.login_lock_time = None
        self.login_attempts = 0
        return "Account unlocked"

    def unlock_fraud(self, admin_pin):
        if str(admin_pin) != self._pin:
            return "Invalid PIN"
        
        self.fraud_locked = False
        self.fraud_lock_time = None
        return "Account unlocked"


if __name__ == "__main__":
    print("Test 1: Normal Operations")
    print("-" * 40)
    user = Bank("Ali", "65490", "1234", 100000)

    print(user.login("1234"))
    print(user.deposit(5000))
    print(user.withdraw(2000))
    print(user.withdraw(70000))
    print(user.check_balance())
    user.show_transactions()
    print(user.generate_report())
    print()

    print("Test 2: Failed Login")
    print("-" * 40)
    user2 = Bank("Bob", "12345", "5678", 50000)
    print(user2.login("wrong1"))
    print(user2.login("wrong2"))
    print(user2.login("wrong3"))
    print(user2.login("5678"))
    print(user2.unlock_login("5678"))
    print(user2.login("5678"))
    print()

    print("Test 3: Invalid Amounts")
    print("-" * 40)
    user3 = Bank("Carol", "99999", "9999", 10000)
    user3.login("9999")
    print(user3.deposit(float('nan')))
    print(user3.deposit(float('inf')))
    print(user3.deposit(-100))
    print(user3.withdraw("string"))
    print()

    print("Test 4: Multiple Withdrawals")
    print("-" * 40)
    user4 = Bank("David", "44444", "4444", 150000, fraud_limit=50000)
    user4.login("4444")
    print(user4.withdraw(45000))
    print(user4.withdraw(45000))
    print()

    print("Test 5: Rapid Transactions")
    print("-" * 40)
    user5 = Bank("Eve", "55555", "5555", 50000, fraud_limit=50000)
    user5.login("5555")
    print(user5.deposit(1000))
    print(user5.deposit(2000))
    print(user5.deposit(3000))
    print(user5.withdraw(1000))
    print(user5.generate_report())
```

