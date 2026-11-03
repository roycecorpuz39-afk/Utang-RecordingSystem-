from datetime import datetime
from pathlib import Path

# GST Rice Trading - Debt Recording System
# Clean, self-contained version. Records are stored locally in text files.

ADMIN_USERNAME = "admin"
ADMIN_PASSWORD = "admin123"  # Change this before real use.
MAX_LOGIN_ATTEMPTS = 3
MAX_SACKS_ON_DEBT = 8
INTEREST_PERCENT = 5
WIDTH = 56
LINE = "=" * WIDTH
DATA_DIR = Path(".")
CUSTOMER_FILE = DATA_DIR / "customers.txt"
PURCHASE_FILE = DATA_DIR / "purchases.txt"
PAYMENT_FILE = DATA_DIR / "payments.txt"
PRICE_FILE = DATA_DIR / "prices.txt"

DEFAULT_PRODUCTS = {
    "Princess Bea": 1200,
    "Young Chow": 1350,
    "Sweet Rice": 1100,
    "Head Rice": 1380,
    "Valencia Rice": 1370,
    "Dinurado Master Chief": 1250,
    "Aroma Master Chief": 1270,
}
products = DEFAULT_PRODUCTS.copy()
customers = []  # dictionaries: name, contact, address, date_added
purchases = []  # dictionaries: id, name, item, sacks, price, total, down, interest, date
payments = []   # dictionaries: id, name, amount, type, reference, date


def now():
    return datetime.now().strftime("%Y-%m-%d %H:%M:%S")


def banner(*titles):
    print("\n" + LINE)
    for title in titles:
        print(title.center(WIDTH))
    print(LINE)


def pause():
    input("\nPress Enter to continue...")


def money(value):
    return "PHP " + format(int(value), ",")


def safe_text(value):
    return str(value).replace("|", "/").replace("\n", " ").strip()


def read_records(path):
    records = []
    try:
        with path.open("r", encoding="utf-8") as file:
            for line in file:
                parts = line.rstrip("\n").split("|")
                if parts and any(parts):
                    records.append(parts)
    except FileNotFoundError:
        pass
    except OSError as exc:
        print(f"Warning: Could not read {path}: {exc}")
    return records


def write_records(path, records):
    try:
        with path.open("w", encoding="utf-8") as file:
            for record in records:
                file.write("|".join(safe_text(v) for v in record) + "\n")
        return True
    except OSError as exc:
        print(f"Error: Could not save {path}: {exc}")
        return False


def save_all():
    ok1 = write_records(CUSTOMER_FILE, [
        [c["name"], c["contact"], c["address"], c["date_added"]] for c in customers
    ])
    ok2 = write_records(PURCHASE_FILE, [
        [p["id"], p["name"], p["item"], p["sacks"], p["price"], p["total"],
         p["down"], p["interest"], p["date"]] for p in purchases
    ])
    ok3 = write_records(PAYMENT_FILE, [
        [p["id"], p["name"], p["amount"], p["type"], p["reference"], p["date"]]
        for p in payments
    ])
    return ok1 and ok2 and ok3


def load_all():
    global customers, purchases, payments, products
    customers, purchases, payments = [], [], []
    for r in read_records(CUSTOMER_FILE):
        if len(r) == 4:
            customers.append(dict(name=r[0], contact=r[1], address=r[2], date_added=r[3]))
    for r in read_records(PURCHASE_FILE):
        if len(r) == 9:
            try:
                purchases.append(dict(id=r[0], name=r[1], item=r[2], sacks=int(r[3]),
                                      price=int(r[4]), total=int(r[5]), down=int(r[6]),
                                      interest=int(r[7]), date=r[8]))
            except ValueError:
                print("Warning: Skipped an invalid purchase record.")
    for r in read_records(PAYMENT_FILE):
        if len(r) == 6:
            try:
                payments.append(dict(id=r[0], name=r[1], amount=int(r[2]), type=r[3],
                                     reference=r[4], date=r[5]))
            except ValueError:
                print("Warning: Skipped an invalid payment record.")
    products = DEFAULT_PRODUCTS.copy()
    saved_prices = read_records(PRICE_FILE)
    for r in saved_prices:
        if len(r) == 2 and r[0] in products:
            try:
                price = int(r[1])
                if price > 0:
                    products[r[0]] = price
            except ValueError:
                print("Warning: Skipped an invalid rice price.")
    if not saved_prices:
        save_prices()


def save_prices():
    return write_records(PRICE_FILE, [[name, price] for name, price in products.items()])


def get_text(prompt):
    while True:
        value = safe_text(input(prompt))
        if value:
            return value
        print("This field cannot be empty.")


def get_number(prompt, minimum=0, maximum=10_000_000):
    while True:
        raw = input(prompt).strip()
        try:
            value = int(raw)
            if str(value) != raw and raw not in (f"+{value}",):
                raise ValueError
            if minimum <= value <= maximum:
                return value
            print(f"Enter a whole number from {minimum:,} to {maximum:,}.")
        except ValueError:
            print("Invalid input. Enter a whole number.")


def get_choice(prompt, low, high):
    return get_number(prompt, low, high)


def yes_no(prompt):
    while True:
        answer = input(prompt).strip().lower()
        if answer in ("y", "yes"):
            return True
        if answer in ("n", "no"):
            return False
        print("Please enter Y or N.")


def customer_by_name(name):
    return next((c for c in customers if c["name"].casefold() == name.casefold()), None)


def ask_for_customer():
    if not customers:
        print("No customers registered yet. Register a customer first.")
        pause()
        return None
    print("\nRegistered customers:")
    for i, customer in enumerate(customers, 1):
        print(f"{i}. {customer['name']} ({customer['contact']})")
    print("0. Cancel")
    choice = get_choice("Select customer: ", 0, len(customers))
    return None if choice == 0 else customers[choice - 1]


def new_transaction_id(prefix):
    stamp = datetime.now().strftime("%Y%m%d%H%M%S")
    existing = {p["id"] for p in purchases} | {p["id"] for p in payments}
    candidate, counter = prefix + stamp, 1
    while candidate in existing:
        counter += 1
        candidate = f"{prefix}{stamp}{counter}"
    return candidate


def customer_purchases(name):
    return [p for p in purchases if p["name"].casefold() == name.casefold()]


def customer_payments(name):
    return [p for p in payments if p["name"].casefold() == name.casefold()]


def total_credit(name):
    return sum(p["total"] - p["down"] + p["interest"] for p in customer_purchases(name))


def total_payments(name):
    return sum(p["amount"] for p in customer_payments(name))


def balance(name):
    return max(0, total_credit(name) - total_payments(name))


def outstanding_sacks(name):
    # Conservative approximation: count sacks from purchases with an unpaid balance.
    # Partial payments reduce money owed but are not allocated to individual sacks.
    owed = balance(name)
    sacks = 0
    for p in customer_purchases(name):
        purchase_due = p["total"] - p["down"] + p["interest"]
        if purchase_due > 0 and owed > 0:
            sacks += p["sacks"]
            owed -= purchase_due
    return sacks


def register_customer():
    banner("REGISTER CUSTOMER")
    name = get_text("Full name: ")
    if customer_by_name(name):
        print("A customer with that name is already registered.")
        pause()
        return
    contact = get_text("Contact number: ")
    house = get_text("House/lot number (or N/A): ")
    street = get_text("Street (or N/A): ")
    barangay = get_text("Barangay: ")
    city = get_text("City/Municipality: ")
    province = get_text("Province: ")
    postal = get_text("Postal code (or N/A): ")
    address = ", ".join([house, street, barangay, city, province, postal])
    customers.append({"name": name, "contact": contact, "address": address, "date_added": now()})
    save_all()
    print(f"\nCustomer {name} registered successfully.")
    pause()


def print_products():
    print("\nAvailable rice products:")
    for i, (name, price) in enumerate(products.items(), 1):
        print(f"{i}. {name:<28} {money(price)} / sack")


def place_order(customer):
    banner("NEW CREDIT PURCHASE", customer["name"])
    print_products()
    names = list(products)
    print("0. Cancel")
    choice = get_choice("Select product: ", 0, len(names))
    if choice == 0:
        return
    item = names[choice - 1]
    current_sacks = outstanding_sacks(customer["name"])
    remaining_allowed = MAX_SACKS_ON_DEBT - current_sacks
    if remaining_allowed <= 0:
        print(f"Credit limit reached ({MAX_SACKS_ON_DEBT} sacks on debt).")
        pause()
        return
    sacks = get_number(f"Number of sacks (1-{remaining_allowed}): ", 1, remaining_allowed)
    price = products[item]
    total = price * sacks
    print(f"\nItem: {item}\nPrice per sack: {money(price)}\nQuantity: {sacks}")
    print("Purchase total:", money(total))
    down = get_number("Down payment (PHP): ", 0, total)
    debt_before_interest = total - down
    interest = int(debt_before_interest * INTEREST_PERCENT / 100 + 0.5)
    print("Interest (5% of unpaid amount):", money(interest))
    print("Balance to pay:", money(debt_before_interest + interest))
    if not yes_no("Confirm this purchase? (Y/N): "):
        print("Purchase cancelled.")
        pause()
        return
    purchase = {"id": new_transaction_id("P"), "name": customer["name"], "item": item,
                "sacks": sacks, "price": price, "total": total, "down": down,
                "interest": interest, "date": now()}
    purchases.append(purchase)
    if down:
        payments.append({"id": new_transaction_id("D"), "name": customer["name"],
                         "amount": down, "type": "Down Payment", "reference": purchase["id"],
                         "date": now()})
    save_all()
    print("\nCredit purchase recorded successfully!")
    print("Transaction ID:", purchase["id"])
    print("Customer:", customer["name"])
    print("Total:", money(total), "| Down payment:", money(down))
    print("Interest:", money(interest), "| Remaining balance:", money(balance(customer["name"])))
    pause()


def record_credit_purchase():
    banner("RECORD CREDIT PURCHASE")
    customer = ask_for_customer()
    if customer is None:
        return
    while True:
        place_order(customer)
        if not yes_no("Would you like to make another purchase for this customer? (Y/N): "):
            break


def record_payment():
    banner("RECORD CUSTOMER PAYMENT")
    customer = ask_for_customer()
    if customer is None:
        return
    current = balance(customer["name"])
    print("Current balance:", money(current))
    if current <= 0:
        print("There is no outstanding balance.")
        pause()
        return
    amount = get_number("Payment amount (PHP): ", 1, current)
    if not yes_no(f"Confirm payment of {money(amount)}? (Y/N): "):
        print("Payment cancelled.")
        pause()
        return
    payment = {"id": new_transaction_id("R"), "name": customer["name"], "amount": amount,
               "type": "Regular Payment", "reference": "none", "date": now()}
    payments.append(payment)
    save_all()
    print("Payment recorded successfully.")
    print("Transaction ID:", payment["id"])
    print("Remaining balance:", money(balance(customer["name"])))
    pause()


def update_prices():
    banner("UPDATE RICE PRICES")
    print_products()
    print("0. Back")
    names = list(products)
    choice = get_choice("Select product: ", 0, len(names))
    if choice == 0:
        return
    name = names[choice - 1]
    old = products[name]
    new = get_number(f"New price for {name} (PHP): ", 1)
    products[name] = new
    if save_prices():
        print(f"{name}: {money(old)} -> {money(new)}")
    else:
        products[name] = old
        print("Price was not changed because it could not be saved.")
    pause()


def view_customer_history():
    banner("CUSTOMER HISTORY")
    customer = ask_for_customer()
    if customer is None:
        return
    name = customer["name"]
    print("\nName:", name, "\nContact:", customer["contact"], "\nAddress:", customer["address"])
    print("\n--- Credit Purchases ---")
    found = customer_purchases(name)
    if not found:
        print("No credit purchases.")
    for p in found:
        due = p["total"] - p["down"] + p["interest"]
        print(f"{p['id']} | {p['date']} | {p['item']} x{p['sacks']} | "
              f"Total {money(p['total'])} | Down {money(p['down'])} | "
              f"Interest {money(p['interest'])} | Due {money(due)}")
    print("\n--- Payments ---")
    found_payments = customer_payments(name)
    if not found_payments:
        print("No payments.")
    for p in found_payments:
        print(f"{p['id']} | {p['date']} | {p['type']} | {money(p['amount'])} | Ref: {p['reference']}")
    print("\n--- Summary ---")
    print("Total credit (including interest):", money(total_credit(name)))
    print("Total payments:", money(total_payments(name)))
    print("Remaining balance:", money(balance(name)))
    pause()


def view_all_customers():
    banner("ALL CUSTOMERS")
    if not customers:
        print("No customer records found.")
    else:
        for i, c in enumerate(customers, 1):
            print(f"{i}. {c['name']} | {c['contact']} | Added: {c['date_added']}")
    pause()


def staff_menu():
    while True:
        banner("STAFF MENU")
        print("1. Register customer")
        print("2. Record credit purchase")
        print("3. Record customer payment")
        print("4. Update rice prices")
        print("5. View customer history")
        print("6. View all customers")
        print("7. Log out")
        choice = get_choice("Choose (1-7): ", 1, 7)
        if choice == 1:
            register_customer()
        elif choice == 2:
            record_credit_purchase()
        elif choice == 3:
            record_payment()
        elif choice == 4:
            update_prices()
        elif choice == 5:
            view_customer_history()
        elif choice == 6:
            view_all_customers()
        else:
            break


def staff_login():
    banner("STAFF LOGIN")
    for attempt in range(MAX_LOGIN_ATTEMPTS):
        username = input("Username: ").strip()
        password = input("Password: ").strip()
        if username == ADMIN_USERNAME and password == ADMIN_PASSWORD:
            print("Login successful.")
            return True
        print(f"Incorrect username or password. Attempts left: {MAX_LOGIN_ATTEMPTS - attempt - 1}")
    print("Maximum login attempts reached.")
    pause()
    return False


def customer_login():
    banner("CUSTOMER LOGIN")
    if not customers:
        print("No customers registered yet.")
        pause()
        return None
    name = get_text("Enter your registered full name: ")
    contact = get_text("Enter your contact number: ")
    customer = customer_by_name(name)
    if customer and customer["contact"] == contact:
        return customer
    print("Customer name or contact number did not match.")
    pause()
    return None


def customer_menu(customer):
    while True:
        banner("CUSTOMER PORTAL", customer["name"])
        print("1. View my purchases and balance")
        print("2. View my payment history")
        print("3. Log out")
        choice = get_choice("Choose (1-3): ", 1, 3)
        if choice == 1:
            print("\n--- My Purchases ---")
            records = customer_purchases(customer["name"])
            if not records:
                print("No purchases found.")
            for p in records:
                print(f"{p['date']} | {p['item']} x{p['sacks']} | "
                      f"Total {money(p['total'])} | Interest {money(p['interest'])}")
            print("Remaining balance:", money(balance(customer["name"])))
            pause()
        elif choice == 2:
            print("\n--- My Payments ---")
            records = customer_payments(customer["name"])
            if not records:
                print("No payments found.")
            for p in records:
                print(f"{p['date']} | {p['type']} | {money(p['amount'])}")
            pause()
        else:
            break


def main():
    load_all()
    while True:
        banner("WELCOME TO GST RICE TRADING", "DEBT RECORDING SYSTEM")
        print("1. Staff Login")
        print("2. Customer Login")
        print("3. Exit")
        choice = get_choice("Enter your choice (1-3): ", 1, 3)
        if choice == 1:
            if staff_login():
                staff_menu()
        elif choice == 2:
            customer = customer_login()
            if customer:
                customer_menu(customer)
        else:
            save_all()
            save_prices()
            print("Program ended. Records saved locally.")
            break


if __name__ == "__main__":
    main()
