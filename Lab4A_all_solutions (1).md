# Lab 4A — Classes, Responsibilities and Encapsulation

Design of Software System · Solutions (Tasks 1–10 and Extra E1–E5)

---

## Task 1 — The cart that belongs to everybody

File: `task01.py` (run: `pytest task01.py`)

```python
# Lab 4A - Task 1: The cart that belongs to everybody
#
# 1.2 Two bugs, one root cause.
#   Bug 1: `items = []` is in the CLASS body, so it is a class attribute: one list shared by all carts.
#   Bug 2: `coupons=[]` is a default argument. It is evaluated ONCE, when `def` runs, so every cart
#          created without coupons receives that same list object.
#   Root cause: a mutable object ([]) is created once in a shared place (class body / function
#   definition) instead of once per object inside __init__.
#
# 1.3 In a dataclass, `coupons: list = []` raises ValueError ("mutable default <class 'list'> for
#   field coupons is not allowed: use default_factory"). Write `field(default_factory=list)` instead:
#   it calls list() again for every new object.
#
# 1.4 `self.items.append(x)`: the object has no own attribute "items", so Python finds the CLASS
#   attribute and changes that one list in place - every cart sees the change.
#   `self.count += 1` means `self.count = self.count + 1`: it READS the class value but ASSIGNS to the
#   instance, which creates a new instance attribute on this one object. Assignment never changes
#   the class attribute; mutation of the shared object does.

from dataclasses import dataclass, field

import pytest


# --- the original code, kept ONLY to prove that the test reproduces ticket #4127 -------------
class BuggyCart:
    items = []

    def __init__(self, user_id, coupons=[]):
        self.user_id = user_id
        self.coupons = coupons

    def add(self, sku, qty=1):
        self.items.append((sku, qty))

    def apply_coupon(self, code):
        self.coupons.append(code)


# --- 1.3 fixed version --------------------------------------------------------------------
class FixedCart:
    def __init__(self, user_id, coupons=None):
        self.user_id = user_id
        self.items = []                                  # new list for every cart
        self.coupons = list(coupons) if coupons else []  # copy: never keep the caller's list

    def add(self, sku, qty=1):
        self.items.append((sku, qty))

    def apply_coupon(self, code):
        self.coupons.append(code)


# --- 1.3 the same class as a dataclass ----------------------------------------------------
@dataclass
class Cart:
    user_id: int
    items: list = field(default_factory=list)
    coupons: list = field(default_factory=list)

    def __post_init__(self):
        self.items = list(self.items)      # copy what the caller gave us
        self.coupons = list(self.coupons)

    def add(self, sku, qty=1):
        self.items.append((sku, qty))

    def apply_coupon(self, code):
        self.coupons.append(code)


# --- 1.1 tests ------------------------------------------------------------------------------
@pytest.mark.parametrize(
    "cart_class",
    [
        pytest.param(BuggyCart, marks=pytest.mark.xfail(strict=True, reason="ticket #4127")),
        FixedCart,
        Cart,
    ],
)
def test_two_carts_do_not_share_data(cart_class):
    first = cart_class(1)
    second = cart_class(2)

    first.add("A1", 2)
    first.apply_coupon("SALE10")

    assert second.items == []
    assert second.coupons == []
    assert first.items == [("A1", 2)]
    assert first.coupons == ["SALE10"]


@pytest.mark.parametrize("cart_class", [FixedCart, Cart])
def test_passed_coupons_are_copied(cart_class):
    mine = ["WELCOME"]
    cart = cart_class(1, coupons=mine)
    cart.apply_coupon("SALE10")
    assert mine == ["WELCOME"]  # the caller's list is untouched


def test_dataclass_rejects_mutable_default():
    with pytest.raises(ValueError):

        @dataclass
        class Bad:
            coupons: list = []


def test_assignment_vs_mutation_on_class_attribute():  # 1.4
    class A:
        items = []
        count = 0

    a, b = A(), A()
    a.items.append("x")  # mutation of the shared object
    a.count += 1         # creates an instance attribute on `a` only
    assert b.items == ["x"]
    assert b.count == 0 and a.count == 1
    assert "count" in a.__dict__ and "count" not in b.__dict__
```

---

## Task 2 — Three different delivery fees

File: `task02.py` (run: `pytest task02.py`)

```python
# Lab 4A - Task 2: Three different delivery fees
#
# 2.1 By hand (rule: free from 150 000; else 12 000 + 3 000 per STARTED km after the first 3 km).
#
#   order                     | correct | Checkout | Receipt | Courier
#   --------------------------+---------+----------+---------+---------
#   150 000 sum,  4.2 km      |       0 |   0  OK  |  15000  |   0  OK
#    90 000 sum,  4.2 km      |   18000 |  15000   |  15000  | 15600.0
#    90 000 sum,  3.6 km      |   15000 |  12000   |  15000 OK | 13800.0
#
#   (I also ran the old code: it prints exactly these numbers.)
#   Checkout uses int()   -> cuts 1.2 to 1 (rounds DOWN),
#   Receipt uses > 150000 -> exactly 150 000 is not free, and round() -> 1.2 -> 1, 0.6 -> 1,
#   Courier uses no rounding at all -> a float fee (15600.0), and money must never be a float.
#   Only Receipt is right once (3.6 km), and only by luck.
#
# 2.2 Information Expert: Order has food_total and distance_km, so Order owns the fee rule.
#     The three services only call order.delivery_fee().
#
# 2.3 math.ceil() gives "every started kilometre": ceil(1.2) = 2, ceil(0.0) = 0.
#     It returns an int, so the fee is an int.
#
# Done-when check: the rule is in ONE method; the numbers are class constants, so changing 3 000 to
# 3 500 is the one line `FEE_PER_KM = 3_500` (see test_one_line_change).

import math

import pytest


class Order:
    FREE_DELIVERY_FROM = 150_000   # sum
    BASE_FEE = 12_000              # sum, covers the first BASE_KM km
    BASE_KM = 3
    FEE_PER_KM = 3_000             # sum, for every started km after BASE_KM

    def __init__(self, food_total, distance_km):
        self.food_total = food_total
        self.distance_km = distance_km

    def delivery_fee(self):
        if self.food_total >= self.FREE_DELIVERY_FROM:
            return 0
        extra_km = max(0, math.ceil(self.distance_km - self.BASE_KM))
        return self.BASE_FEE + extra_km * self.FEE_PER_KM


class CheckoutService:
    def delivery_fee(self, order):
        return order.delivery_fee()


class ReceiptService:
    def delivery_fee(self, order):
        return order.delivery_fee()


class CourierPayoutService:
    def delivery_fee(self, order):
        return order.delivery_fee()


SERVICES = [CheckoutService(), ReceiptService(), CourierPayoutService()]


# --- 2.4 tests ----------------------------------------------------------------------------
@pytest.mark.parametrize(
    "food_total, expected",
    [
        (149_999, 12_000),  # one sum below the limit: still pays
        (150_000, 0),       # exactly the limit: free
        (150_001, 0),
    ],
)
def test_free_delivery_boundary(food_total, expected):
    assert Order(food_total, 2.0).delivery_fee() == expected


@pytest.mark.parametrize(
    "distance_km, expected",
    [
        (0.5, 12_000),
        (3.0, 12_000),   # exactly 3 km: base fee only
        (3.01, 15_000),  # 0.01 km after the limit already counts as one started km
        (4.0, 15_000),
        (4.01, 18_000),
    ],
)
def test_distance_boundary(distance_km, expected):
    assert Order(90_000, distance_km).delivery_fee() == expected


@pytest.mark.parametrize(
    "food_total, distance_km, expected",
    [(150_000, 4.2, 0), (90_000, 4.2, 18_000), (90_000, 3.6, 15_000)],
)
def test_the_three_orders_from_2_1_in_every_service(food_total, distance_km, expected):
    order = Order(food_total, distance_km)
    for service in SERVICES:
        assert service.delivery_fee(order) == expected


def test_fee_is_always_int():
    for distance in (1, 3.2, 4.2, 10):
        assert isinstance(Order(1_000, distance).delivery_fee(), int)


def test_one_line_change(monkeypatch):
    monkeypatch.setattr(Order, "FEE_PER_KM", 3_500)  # the "one-line change"
    order = Order(90_000, 4.2)                        # 2 started km
    for service in SERVICES:
        assert service.delivery_fee(order) == 12_000 + 2 * 3_500
```

---

## Task 3 — From the product owner's message to CRC cards

> "A patient books an appointment with a dentist for one procedure. Each dentist works in one treatment room and has working hours. A procedure has a name, a duration and a price. An appointment can be moved for free until 24 hours before it starts; after that, a fee of 50 000 sum is charged. The receptionist confirms every appointment, and the system sends the patient an SMS reminder the evening before."

### 3.1 Nouns and verbs

**Verbs:** books, works (in a room), has (working hours), moved, charged, confirms, sends.

| Noun / phrase | Group | Why |
|---|---|---|
| patient | **class** | has its own data (name, phone) and lives on after one booking |
| dentist | **class** | has data (name, working hours, room) and rules (is he free?) |
| appointment | **class** (event with data) | one patient + one dentist + one procedure + a start time + a status; it is the "booking event" like Loan in the lecture |
| procedure | **class** | has its own data: name, duration, price |
| treatment room | **attribute** of Dentist | only a name/number, no behaviour of its own (becomes a class only if rooms must be booked) |
| working hours | **attribute** of Dentist | a value (days and from-to times) |
| name, duration, price | **attributes** of Procedure | plain data |
| 24 hours | **attribute / rule constant** | a rule about Appointment's start time |
| fee of 50 000 sum | **rule constant** | a number inside the move rule; money, so an int |
| receptionist | **actor** (becomes a class after 3.4) | a person who triggers "confirm"; today we store nothing about him |
| the system | **vague word** | which part? it is not a class - it is the whole software; the real doer is ReminderService |
| SMS reminder | **event with data** | phone, text, send time - created by ReminderService, not stored by Appointment |
| evening before | **vague word** | what time exactly? in which time zone? |
| moved | **vague word** | moved to when? by whom? how many times? |
| one procedure | **vague word** | really only one? what if a visit needs two? |
| confirms | **vague word** | what statuses exist (requested -> confirmed -> cancelled)? |
| working hours (details) | **vague word** | per weekday? breaks? holidays? |
| "each dentist works in one treatment room" | **vague word** | can two dentists share a room? then rooms need double-booking checks too |

### 3.2 CRC cards (6 cards, at most 4 responsibilities each)

| **Appointment** | |
|---|---|
| Responsibilities | Collaborators |
| Know patient, dentist, procedure, start time and status | Patient, Dentist, Procedure |
| Calculate its end time (start + procedure duration) | Procedure |
| Decide if it can be moved for free and calculate the move fee (50 000 sum after the 24-hour limit) | - |
| Move itself to a new time; be confirmed | Dentist, Receptionist |

| **Dentist** | |
|---|---|
| Responsibilities | Collaborators |
| Know name, working hours and treatment room | - |
| Decide if he is free for a time interval (inside working hours and not booked twice) | Appointment |
| Know his appointments | Appointment |

| **Procedure** | |
|---|---|
| Responsibilities | Collaborators |
| Know name, duration and price | - |

| **Patient** | |
|---|---|
| Responsibilities | Collaborators |
| Know name and phone number | - |
| Know his appointments (to show the history) | Appointment |

| **Receptionist** | |
|---|---|
| Responsibilities | Collaborators |
| Confirm an appointment (the action starts here, the status change belongs to Appointment) | Appointment |

| **ReminderService** | |
|---|---|
| Responsibilities | Collaborators |
| Find the appointments that start tomorrow | Appointment |
| Build the reminder text and ask the SMS gateway to send it | Patient, SmsGateway |

### 3.3 Who decides?

- **Dentist is not booked twice -> Dentist.** Who has the data? The dentist knows his working hours and all his appointments. An Appointment knows only its own interval, so it cannot see a clash with another one. The class that holds *all* the appointments of one dentist is the only one that can compare them.
- **The 50 000 sum fee -> Appointment.** Who has the data? The rule needs the start time of the appointment and "now". The start time lives in Appointment, so Appointment calculates `move_fee(now)`. The fee amount is a constant of that rule, not data of Patient or Dentist.

### 3.4 The auditor: "who confirmed every appointment?"

Yes, it changes my decision. Before, the receptionist was only an **actor**: a person who presses a button, and we stored nothing about him. Now the system must *remember* him, so the receptionist becomes a **class** (a staff user with an id and a name).

Changes on the cards:
- **Appointment** gets new data: `confirmed_by` (a Receptionist) and `confirmed_at`. Its "be confirmed" responsibility becomes "be confirmed by a receptionist and record who and when". New collaborator: Receptionist.
- **Receptionist** now has a real responsibility: "know id and name" (identity for the audit log).
- The confirm rule stays in Appointment (it has the status data); the receptionist is only passed in.

### 3.5 Is sending the SMS Appointment's responsibility?

**No.** Appointment has booking data and booking rules. Sending an SMS needs the provider, a key, retries and errors - data Appointment does not have. If it sent SMS itself, it would have two reasons to change (booking rules and the SMS provider) and could not be tested without sending real messages (see Task 10).

**ReminderService sends it.** It finds tomorrow's appointments, builds the text and calls an SMS gateway.

Relationship: ReminderService **depends on** Appointment (a *dependency*: it receives appointments as a method parameter and does not store them), and Appointment knows nothing about ReminderService (one direction only). ReminderService -> SmsGateway is an *association*: the gateway is given to the service from outside.

---

## Task 4 — The negative balance in production

File: `task04.py` (run: `pytest task04.py`)

```python
# Lab 4A - Task 4: The negative balance in production
#
# 4.1 amount must be a positive int.
#     deposit(10.5) -> TypeError  (wrong TYPE: it is not an int at all)
#     deposit(-10)  -> ValueError (right type, wrong VALUE: not positive)
#     deposit(True) -> TypeError. isinstance(True, int) is True, so a naive isinstance check would
#     accept it (True == 1 tiyin). I reject bool explicitly.
#
# 4.3 refund() is a THIRD kind of entry. Money goes up like a deposit, but it is not new money from
#     the owner: it is the return of an earlier payment. Accounting, the support team and fraud
#     checks must tell "topped up" from "got a refund", so the history keeps the kind.
#
# 4.4 >>> 0.1 + 0.2        ->  0.30000000000000004
#     A float stores a binary fraction; 0.1 has no exact binary form, so errors appear and grow.
#     An int in tiyin is exact: 10 + 20 == 30 always, there are no fractions of the smallest unit.
#
# Design notes: the balance starts at 0 (no balance argument in the constructor, so nobody can
# create a wallet that already breaks the rules). `balance` and `history` have no setters;
# `history` returns a tuple copy, so the outside world cannot change the real history.

import pytest


class InsufficientFundsError(ValueError):
    pass


class Wallet:
    def __init__(self, owner_id):
        self.owner_id = owner_id
        self._balance = 0                # tiyin
        self._history = []               # (kind, amount, balance_after)

    @property
    def balance(self):
        return self._balance

    @property
    def history(self):
        return tuple(self._history)      # a copy, and immutable

    def deposit(self, amount):
        self._check_amount(amount)
        self._apply("deposit", amount, +amount)

    def withdraw(self, amount):
        self._check_amount(amount)
        if amount > self._balance:
            raise InsufficientFundsError(f"balance {self._balance}, requested {amount}")
        self._apply("withdraw", amount, -amount)

    def refund(self, amount):
        self._check_amount(amount)
        self._apply("refund", amount, +amount)

    # --- private helpers: the ONLY places that touch _balance and _history ---
    @staticmethod
    def _check_amount(amount):
        if isinstance(amount, bool) or not isinstance(amount, int):
            raise TypeError(f"amount must be an int (tiyin), got {type(amount).__name__}")
        if amount <= 0:
            raise ValueError("amount must be positive")

    def _apply(self, kind, amount, delta):
        self._balance += delta
        self._history.append((kind, amount, self._balance))


# --- tests ----------------------------------------------------------------------------------
def test_admin_script_line_is_blocked():
    wallet = Wallet(1)
    wallet.deposit(1_000)
    with pytest.raises(AttributeError):
        wallet.balance -= 500
    assert wallet.balance == 1_000


def test_balance_never_negative():
    wallet = Wallet(1)
    wallet.deposit(100)
    with pytest.raises(InsufficientFundsError):
        wallet.withdraw(101)
    assert wallet.balance == 100
    wallet.withdraw(100)
    assert wallet.balance == 0


@pytest.mark.parametrize("bad, error", [(10.5, TypeError), ("10", TypeError), (True, TypeError),
                                        (None, TypeError), (-10, ValueError), (0, ValueError)])
def test_bad_amounts_are_rejected(bad, error):
    wallet = Wallet(1)
    wallet.deposit(50)
    for method in (wallet.deposit, wallet.withdraw, wallet.refund):
        with pytest.raises(error):
            method(bad)
    assert wallet.balance == 50 and len(wallet.history) == 1  # nothing changed


def test_history_records_every_change():
    wallet = Wallet(1)
    wallet.deposit(1_000)
    wallet.withdraw(300)
    wallet.refund(50)
    assert wallet.history == (
        ("deposit", 1_000, 1_000),
        ("withdraw", 300, 700),
        ("refund", 50, 750),
    )


def test_history_cannot_be_changed_from_outside():
    wallet = Wallet(1)
    wallet.deposit(100)
    with pytest.raises(AttributeError):
        wallet.history.append(("deposit", 999, 999))
    with pytest.raises(TypeError):
        wallet.history[0] = ("deposit", 999, 999)
    with pytest.raises(AttributeError):
        wallet.history = ()
    assert wallet.history == (("deposit", 100, 100),)


def test_float_is_wrong_for_money_int_is_exact():
    assert 0.1 + 0.2 != 0.3
    assert 10 + 20 == 30


def test_failed_withdraw_leaves_no_history_entry():
    wallet = Wallet(1)
    with pytest.raises(InsufficientFundsError):
        wallet.withdraw(1)
    assert wallet.history == ()
```

---

## Task 5 — Validation without touching 40 call sites

File: `task05.py` (run: `pytest task05.py`)

```python
# Lab 4A - Task 5: Validation without touching 40 call sites
#
# 5.1/5.2 price and discount_percent are properties. The line `self.price = price` in __init__ goes
#         through the setter too, so the constructor and later assignments use the SAME check.
#         The 40 old call sites (`product.price = 120_000`) do not change at all.
# 5.3     final_price() = price * (100 - discount) // 100: integer maths, rounded down.
#         99_990 * 85 // 100 = 84_991 (the float version gave 84991.5).
# 5.4     Three lines that show the problem with a stored final price:
#             p = CachedProduct("A1", 100_000, 10)   # final computed once in __init__ -> 90_000
#             p.price = 200_000                      # the price changes later...
#             p.final                                # ...still 90_000, but final_price() is 180_000
#         The copy is a second source of truth and it goes stale. Compute it when it is asked for.
# 5.5     sku is a property with only a getter: p.sku = 'X' raises AttributeError.
#
# Exceptions: wrong value (0, -5, 150) -> ValueError; wrong type (12.5, True, "5") -> TypeError.

import pytest


def _check_int(value, name, minimum, maximum=None):
    if isinstance(value, bool) or not isinstance(value, int):
        raise TypeError(f"{name} must be an int, got {type(value).__name__}")
    if value < minimum or (maximum is not None and value > maximum):
        limit = f">= {minimum}" if maximum is None else f"from {minimum} to {maximum}"
        raise ValueError(f"{name} must be {limit}, got {value}")
    return value


class Product:
    MAX_DISCOUNT = 90

    def __init__(self, sku, price, discount_percent=0):
        self._sku = sku
        self.price = price                      # goes through the setter
        self.discount_percent = discount_percent  # goes through the setter

    @property
    def sku(self):
        return self._sku

    @property
    def price(self):
        return self._price

    @price.setter
    def price(self, value):
        self._price = _check_int(value, "price", 1)

    @property
    def discount_percent(self):
        return self._discount_percent

    @discount_percent.setter
    def discount_percent(self, value):
        self._discount_percent = _check_int(value, "discount_percent", 0, self.MAX_DISCOUNT)

    def final_price(self):
        return self._price * (100 - self._discount_percent) // 100


# --- tests ----------------------------------------------------------------------------------
def test_done_when_conditions():
    p = Product("A1", 100_000)
    with pytest.raises(ValueError):
        p.price = 0
    with pytest.raises(ValueError):
        p.discount_percent = 150
    with pytest.raises(ValueError):
        Product("A1", -5)
    p.price = 120_000                      # the old line still works
    assert p.price == 120_000
    with pytest.raises(AttributeError):
        p.sku = "X"


@pytest.mark.parametrize("price, discount", [(0, 0), (-1, 0), (100, -1), (100, 91), (100, 150)])
def test_constructor_rejects_bad_values(price, discount):
    with pytest.raises(ValueError):
        Product("A1", price, discount)


@pytest.mark.parametrize("price, discount", [(12.5, 0), (True, 0), ("100", 0), (100, 10.0)])
def test_constructor_rejects_wrong_types(price, discount):
    with pytest.raises(TypeError):
        Product("A1", price, discount)


def test_discount_limits_are_inclusive():
    assert Product("A1", 1_000, 0).final_price() == 1_000
    assert Product("A1", 1_000, 90).final_price() == 100


def test_final_price_is_int_rounded_down():
    price = Product("A1", 99_990, 15).final_price()
    assert price == 84_991 and isinstance(price, int)


def test_failed_assignment_keeps_old_value():
    p = Product("A1", 5_000, 10)
    with pytest.raises(ValueError):
        p.price = -1
    assert p.price == 5_000


def test_cached_final_price_goes_stale():  # 5.4
    class CachedProduct(Product):
        def __init__(self, sku, price, discount_percent=0):
            super().__init__(sku, price, discount_percent)
            self.final = self.final_price()   # computed once

    p = CachedProduct("A1", 100_000, 10)
    p.price = 200_000
    assert p.final == 90_000               # stale
    assert p.final_price() == 180_000      # correct
```

---

## Task 6 — Duplicates after the nightly import

File: `task06.py` (run: `pytest task06.py`)

```python
# Lab 4A - Task 6: Duplicates after the nightly import
#
# 6.1 If a class defines neither __eq__ nor __hash__, Python uses the defaults from `object`:
#     == compares IDENTITY (a == b only if they are the very same object) and hash() is derived from
#     the object's identity (its id). Every `Student(*row)` creates a new object, so the set sees
#     two different students -> len(seen) == 2, and after a week 7 copies.
#
# 6.2 If a class defines only __eq__, Python sets __hash__ = None. The object becomes unhashable,
#     so `seen.add(...)` raises TypeError: unhashable type: 'Student'. (Python does this on purpose:
#     equal objects MUST have equal hashes, and it cannot guess a hash that agrees with your __eq__.)
#
# 6.3 "The same student" = the same student_id. The id is the identity of the student; name and
#     group are attributes that change over time. A student moved from SE-31 to SE-32 is still the
#     same student, so the new CSV row must be recognised as him. __eq__ and __hash__ both use only
#     student_id, so they always agree. (Note: set.add() keeps the OLD object when an equal one is
#     already inside; to update the group, use a dict {student_id: Student} instead of a set.)
#
# 6.4 If the hash depends on group (see BadHashStudent in the tests), then
#         s = BadHashStudent("U1", "Ali", "SE-31"); seen = {s}; s.group = "SE-32"; s in seen -> False
#     The object sits in the bucket of its OLD hash; the lookup now searches the bucket of the NEW
#     hash. The set "lost" a student that is still inside. Rule: a hash may use only fields that
#     never change while the object is in a set.
#
# 6.5 @dataclass(frozen=True) with default settings generates __eq__ and __hash__ from ALL fields
#     (student_id, full_name, group). Then SE-31 and SE-32 versions are different students (the
#     duplicate problem comes back), and frozen forbids changing the group at all, so a transfer
#     would need a new object. We need identity by student_id only and mutable attributes, which
#     a plain frozen dataclass does not give (field(compare=False) + frozen would fix equality but
#     still block the group change).

import pytest


class Student:
    def __init__(self, student_id, full_name, group):
        self.student_id = student_id
        self.full_name = full_name
        self.group = group

    def __eq__(self, other):
        if not isinstance(other, Student):
            return NotImplemented
        return self.student_id == other.student_id

    def __hash__(self):
        return hash(self.student_id)

    def __repr__(self):  # 6.6
        return f"Student(student_id={self.student_id!r}, group={self.group!r})"


def import_students(rows):
    seen = set()
    for row in rows:
        seen.add(Student(*row))
    return seen


# --- tests ----------------------------------------------------------------------------------
ROWS = [("U2310045", "Ali Valiyev", "SE-31"), ("U2310045", "Ali Valiyev", "SE-31")]


def test_loop_gives_one_student():
    assert len(import_students(ROWS)) == 1  # the loop "prints 1"


def test_student_is_found_after_group_change():
    s = Student("U2310045", "Ali Valiyev", "SE-31")
    seen = {s}
    s.group = "SE-32"
    assert s in seen
    assert Student("U2310045", "Ali Valiyev", "SE-32") in seen  # same student, new CSV row


def test_equal_objects_have_equal_hashes():
    a = Student("U1", "Ali", "SE-31")
    b = Student("U1", "Ali V.", "SE-32")
    assert a == b and hash(a) == hash(b)
    assert a != Student("U2", "Ali", "SE-31")


def test_not_equal_to_other_types():
    assert Student("U1", "Ali", "SE-31") != "U1"


def test_repr_for_logs():
    s = Student("U2310045", "Ali Valiyev", "SE-31")
    assert repr(s) == "Student(student_id='U2310045', group='SE-31')"


# --- the explanations from 6.1, 6.2, 6.4 and 6.5 as running code --------------------------------
def test_6_1_default_eq_and_hash_use_identity():
    class Plain:
        def __init__(self, x):
            self.x = x

    a, b = Plain(1), Plain(1)
    assert a != b and a == a
    assert len({a, b}) == 2


def test_6_2_only_eq_makes_the_class_unhashable():
    class OnlyEq:
        def __eq__(self, other):
            return True

    assert OnlyEq.__hash__ is None
    with pytest.raises(TypeError):
        {OnlyEq()}


def test_6_4_hash_that_depends_on_a_changing_field():
    class BadHashStudent:
        def __init__(self, student_id, group):
            self.student_id, self.group = student_id, group

        def __eq__(self, other):
            return self.student_id == other.student_id

        def __hash__(self):
            return hash((self.student_id, self.group))  # <-- the mistake

    s = BadHashStudent("U1", "SE-31")
    seen = {s}
    s.group = "SE-32"
    assert (s in seen) is False  # four lines, as the task asks


def test_6_5_frozen_dataclass_is_not_the_fix():
    from dataclasses import FrozenInstanceError, dataclass

    @dataclass(frozen=True)
    class FrozenStudent:
        student_id: str
        full_name: str
        group: str

    old = FrozenStudent("U1", "Ali", "SE-31")
    new = FrozenStudent("U1", "Ali", "SE-32")
    assert old != new                      # compares ALL fields -> the transfer makes a "new" student
    assert len({old, new}) == 2            # duplicates are back
    with pytest.raises(FrozenInstanceError):
        old.group = "SE-32"                # and the group can never be updated in place
```

---

## Task 7 — The class design decides what happens on delete

### 7.1 and 7.2 Relationships and multiplicity

| Pair | Relationship | Why (from the code) | Multiplicity |
|---|---|---|---|
| Order -> OrderLine | **composition** | `Order.add()` creates `OrderLine(...)` inside itself; lines have no life without the order | Order **1** — **0..\*** OrderLine (a confirmed order needs 1..\*) |
| OrderLine -> Product | **association** | stores `self.product`, but the product lives on its own | OrderLine **\*** — **1** Product |
| Order -> Customer | **association** | stores `self.customer`; the customer lives on its own | Order **\*** — **1** Customer |
| Order -> Courier | **association** | `assign()` stores the courier; the courier lives on its own | Order **\*** — **0..1** Courier (`None` until assigned) |
| Catalog -> Product | **aggregation** | `Catalog` holds `list(products)`: a whole-part link, but products exist without the catalog and can be in several catalogs | Catalog **0..\*** — **0..\*** Product |
| Order -> Product (in `add(product, qty)`) | **dependency** | the method only *uses* `product.price` during the call; the order itself does not store the product (the OrderLine does) | - |
| SmsGateway -> Order | **dependency** | `order_shipped(order)` receives the order as a parameter and does not store it | - |

**Multiplicity not enforced by the code yet:** `Order 1 — 1..* OrderLine`. Nothing stops an order from having *zero* lines (and `qty` can be 0 or negative). Also the "exactly one customer" end is not checked: `Order(None)` works.

### 7.3 ON DELETE for each foreign key

Lifetime test: **"does the part disappear with the whole?"**

| Foreign key | ON DELETE | Justification |
|---|---|---|
| `order_lines.order_id` | **CASCADE** | Composition. An order line is a part of the order; without the order it has no meaning. The part disappears with the whole. |
| `order_lines.product_id` | **RESTRICT** | Association. The product does not belong to the line; the line is a record of what was sold in the past. Deleting a product must not erase sales history (and SET NULL would leave a line that no longer says what was bought). Retire products with an "archived" flag instead. |
| `orders.customer_id` | **RESTRICT** | Association. The order is a financial record. CASCADE would wipe revenue history; SET NULL would leave orders nobody can be refunded or contacted for. A customer who has orders cannot simply be deleted (anonymise him instead). |
| `orders.courier_id` | **SET NULL** | Association with multiplicity **0..1**: the model already allows "no courier". If a courier leaves, the order stays valid and just becomes unassigned. CASCADE would delete orders because of a staff change, which is clearly wrong. |

Only one of the four is CASCADE, because only one relationship is a composition.

### 7.4 Why is `unit_price` a copy and not a DRY violation?

`Product.price` is the **current** price; `OrderLine.unit_price` is the price **at the moment of sale**. These are two different facts, with different reasons to change. DRY says "do not repeat the same knowledge"; here the knowledge is not the same.

If the line read `product.price`, then after a price change every old order would change its total, and the amount the customer actually paid would no longer match the order (the same kind of bug as in Task 8). The copy is a snapshot of history, so it is correct and necessary in production.

---

## Task 8 — Somebody changed my order

File: `task08.py` (run: `pytest task08.py`)

```python
# Lab 4A - Task 8: Somebody changed my order
#
# 8.1 Two causes.
#   (a) ALIASING: Order(1001, cart_lines) stores the very same list object. `cart_lines` (the cart's
#       name) and `order.lines` (the order's name) are two names for ONE list. The cart keeps
#       appending after checkout, and the order "changes" although nobody touched the order.
#       Order is the owner of its lines (composition): the whole must own its parts, so it must hold
#       its OWN copy, not a list that somebody else also holds.
#   (b) BROKEN ENCAPSULATION: `lines` is a public mutable list, so the promo module can do
#       order.lines.append(...) from outside, even after confirm(). Order cannot protect its own
#       rule ("a confirmed order does not change") because the outsider bypasses every method.
#       Composition also means that nobody else is allowed to change the parts directly.
#
# 8.4 The invariant "an order with no lines cannot be confirmed" is checked in confirm(): that is
#     the method that moves the order from draft to confirmed, and Order is the class that holds
#     the lines, so it is the expert for this rule (add_line cannot check it - it only adds).


class OrderLockedError(Exception):
    """The order is already confirmed and cannot be changed."""


class EmptyOrderError(Exception):
    """An order without lines cannot be confirmed."""


class Order:
    def __init__(self, order_id, lines=()):
        self.order_id = order_id
        self._lines = []          # our own list, never the caller's
        self._status = "draft"
        for sku, qty, unit_price in lines:   # copy the data in (and validate it)
            self.add_line(sku, qty, unit_price)

    @property
    def status(self):
        return self._status

    @property
    def lines(self):
        return tuple(self._lines)   # read-only view: .append() does not exist on a tuple

    def total(self):
        return sum(qty * price for _, qty, price in self._lines)

    def add_line(self, sku, qty, unit_price):
        if self._status != "draft":
            raise OrderLockedError(f"order {self.order_id} is {self._status}")
        if qty < 1:
            raise ValueError("qty must be at least 1")
        if unit_price < 0:
            raise ValueError("unit_price cannot be negative")
        self._lines.append((sku, qty, unit_price))

    def confirm(self):
        if self._status != "draft":
            raise OrderLockedError(f"order {self.order_id} is already {self._status}")
        if not self._lines:
            raise EmptyOrderError("cannot confirm an order with no lines")
        self._status = "confirmed"


# --- tests ----------------------------------------------------------------------------------
import pytest


def test_script_from_the_task_prints_100000():
    cart_lines = [("A1", 2, 50_000)]
    order = Order(1001, cart_lines)
    order.confirm()
    cart_lines.append(("B7", 1, 30_000))   # the cart keeps working
    assert order.total() == 100_000


def test_lines_cannot_be_appended_from_outside():
    order = Order(1, [("A1", 1, 10_000)])
    with pytest.raises(AttributeError):
        order.lines.append(("B7", 1, 30_000))
    with pytest.raises(AttributeError):
        order.lines = []
    assert order.lines == (("A1", 1, 10_000),)


def test_add_line_after_confirm_raises_order_locked_error():
    order = Order(1, [("A1", 1, 10_000)])
    order.confirm()
    with pytest.raises(OrderLockedError):
        order.add_line("B7", 1, 30_000)
    assert order.total() == 10_000


def test_empty_order_cannot_be_confirmed():
    order = Order(1)
    with pytest.raises(EmptyOrderError):
        order.confirm()
    assert order.status == "draft"


def test_add_line_rejects_bad_qty():
    order = Order(1)
    for qty in (0, -3):
        with pytest.raises(ValueError):
            order.add_line("A1", qty, 10_000)
    assert order.lines == ()


def test_status_is_read_only_and_confirm_is_final():
    order = Order(1, [("A1", 1, 10_000)])
    with pytest.raises(AttributeError):
        order.status = "paid"
    order.confirm()
    with pytest.raises(OrderLockedError):
        order.confirm()


def test_constructor_validates_lines():
    with pytest.raises(ValueError):
        Order(1, [("A1", 0, 10_000)])
```

---

## Task 9 — Splitting the god class

File: `task09.py` (run: `pytest task09.py`)

```python
# Lab 4A - Task 9: Splitting the god class
#
# 9.1 Which attributes each method uses (and the groups):
#
#   method            | students | scores | weights | deadlines | submitted_at | smtp_host
#   ------------------+----------+--------+---------+-----------+--------------+----------
#   add_student       |    x     |        |         |           |              |
#   record_score      |   (x)    |   x    |         |           |      x       |
#   late_penalty      |          |   x    |         |     x     |      x       |
#   final_grade       |   (x)    |   x    |    x    |    (x)    |     (x)      |   (through late_penalty)
#   letter            |          |        |         |           |              |  uses only the grade number
#   export_csv        |    x     |  (x)   |         |           |              |  (through final_grade/letter)
#   email_results     |    x     |  (x)   |         |           |              |    x
#
#   Group A - "one submission event": record_score + late_penalty (scores, deadlines, submitted_at)
#   Group B - "all marks of a student, weighted": final_grade (scores, weights)
#   Group C - "number -> letter": letter (no stored data at all, only a rule)
#   Group D - "output": export_csv (file system) and email_results (smtp_host) - they only READ results
#
# 9.2 The event with data is the SUBMISSION: one student handed in one assessment at one moment and
#     got a score. (Like Loan in the lecture, it owns the fine rule: here the late penalty.)
#
#   CRC cards (max 4 responsibilities each)
#   ----------------------------------------------------------------------------------------------
#   Student        | know student_id and name                             | collaborators: -
#   Assessment     | know name, weight and deadline                        | -
#   Submission     | know student, assessment, raw score, submitted_at     | Student, Assessment
#                  | decide how many days late it is                       |
#                  | calculate the late penalty (%)                        |
#                  | calculate the score after the penalty                 |
#   Gradebook      | store students, assessments and submissions           | Student, Assessment, Submission
#                  | record a score (creates a Submission)                 |
#                  | calculate a student's weighted final grade            |
#   GradeScale     | turn a grade number into a letter A/B/C/D/F           | -
#   CsvExporter    | write the results of a gradebook to a CSV file        | Gradebook, GradeScale
#   ResultsMailer  | know the smtp host; e-mail the results                | Gradebook, GradeScale
#
# 9.3 Late penalty: Submission - it has the score, submitted_at and (through Assessment) the deadline.
#     Final grade: Gradebook - it knows ALL submissions of a student and the weights.
#     Dependencies (used, not stored): CsvExporter -> Gradebook, CsvExporter -> GradeScale,
#       ResultsMailer -> Gradebook, ResultsMailer -> GradeScale (all are method parameters only).
#     Stored relationships: Submission -> Student, Submission -> Assessment (association);
#       Gradebook owns its Submissions (composition) and holds Students/Assessments (aggregation).
#     Three developers can now work in parallel: (1) Submission + Assessment (penalty rules),
#       (2) Gradebook + GradeScale (grades), (3) CsvExporter + ResultsMailer (output).

import math
from dataclasses import dataclass
from datetime import datetime, timedelta

import pytest

ONE_DAY = timedelta(days=1)


@dataclass
class Student:
    student_id: str
    name: str


@dataclass(frozen=True)
class Assessment:
    name: str
    weight: float          # 0..1
    deadline: datetime


class Submission:
    PENALTY_PERCENT_PER_DAY = 10
    MAX_PENALTY_PERCENT = 50

    def __init__(self, student, assessment, score, submitted_at):
        if not 0 <= score <= 100:
            raise ValueError("score must be from 0 to 100")
        self.student = student
        self.assessment = assessment
        self.score = score
        self.submitted_at = submitted_at

    def days_late(self):
        """Every STARTED day after the deadline counts: 1 minute late = 1 day late."""
        late = self.submitted_at - self.assessment.deadline
        if late <= timedelta(0):
            return 0
        return -(-late // ONE_DAY)           # ceiling division, integers only

    def late_penalty_percent(self):
        return min(self.days_late() * self.PENALTY_PERCENT_PER_DAY, self.MAX_PENALTY_PERCENT)

    def final_score(self):
        return self.score * (100 - self.late_penalty_percent()) / 100


class GradeScale:
    def letter(self, grade):
        for limit, letter in ((90, "A"), (80, "B"), (70, "C"), (60, "D")):
            if grade >= limit:
                return letter
        return "F"


class Gradebook:
    def __init__(self, assessments):
        if abs(sum(a.weight for a in assessments) - 1) > 1e-9:
            raise ValueError("weights must add up to 1")
        self._assessments = {a.name: a for a in assessments}
        self._students = {}
        self._submissions = {}   # (student_id, assessment_name) -> Submission

    def add_student(self, student):
        self._students[student.student_id] = student

    def record_score(self, student_id, assessment_name, score, submitted_at):
        student = self._students[student_id]
        assessment = self._assessments[assessment_name]
        self._submissions[(student_id, assessment_name)] = Submission(
            student, assessment, score, submitted_at
        )

    def final_grade(self, student_id):
        total = 0.0
        for name, assessment in self._assessments.items():
            submission = self._submissions.get((student_id, name))
            score = submission.final_score() if submission else 0   # no submission = 0
            total += assessment.weight * score
        return total


class CsvExporter:
    """Skeleton: only writes results. Takes a Gradebook as a parameter (dependency)."""

    def export(self, gradebook, scale, path):
        raise NotImplementedError


class ResultsMailer:
    """Skeleton: only e-mails results. smtp_host belongs HERE, not in the grade rules."""

    def __init__(self, smtp_host):
        self.smtp_host = smtp_host

    def send(self, gradebook, scale):
        raise NotImplementedError


# --- 9.4 tests: four submissions ---------------------------------------------------------------
DEADLINE = datetime(2026, 11, 10, 23, 59)


def make_submission(delay, score=80):
    exam = Assessment("midterm", 0.3, DEADLINE)
    return Submission(Student("U1", "Ali"), exam, score, DEADLINE + delay)


@pytest.mark.parametrize(
    "delay, days_late, penalty, final",
    [
        (timedelta(0), 0, 0, 80),                              # on time
        (timedelta(minutes=1), 1, 10, 72),                     # 1 minute late = 1 started day
        (timedelta(days=1, minutes=1), 2, 20, 64),             # 1 day 1 minute late = 2 started days
        (timedelta(days=6), 6, 50, 40),                        # 6 days late: 60% -> capped at 50%
    ],
)
def test_late_penalty_for_four_submissions(delay, days_late, penalty, final):
    s = make_submission(delay)
    assert s.days_late() == days_late
    assert s.late_penalty_percent() == penalty
    assert s.final_score() == pytest.approx(final)


def test_early_submission_has_no_penalty():
    assert make_submission(timedelta(hours=-5)).late_penalty_percent() == 0


def test_exactly_one_day_late_is_one_day():
    assert make_submission(timedelta(days=1)).days_late() == 1


def test_score_must_be_0_to_100():
    with pytest.raises(ValueError):
        make_submission(timedelta(0), score=101)


def test_final_grade_and_letter():
    book = Gradebook([
        Assessment("midterm", 0.3, DEADLINE),
        Assessment("final", 0.5, DEADLINE),
        Assessment("labs", 0.2, DEADLINE),
    ])
    book.add_student(Student("U1", "Ali"))
    book.record_score("U1", "midterm", 80, DEADLINE)
    book.record_score("U1", "final", 90, DEADLINE)
    book.record_score("U1", "labs", 100, DEADLINE + timedelta(hours=3))  # 1 started day late -> 90
    grade = book.final_grade("U1")
    assert grade == pytest.approx(0.3 * 80 + 0.5 * 90 + 0.2 * 90)  # 87
    assert GradeScale().letter(grade) == "B"


def test_missing_submission_counts_as_zero():
    book = Gradebook([Assessment("midterm", 0.5, DEADLINE), Assessment("final", 0.5, DEADLINE)])
    book.add_student(Student("U1", "Ali"))
    book.record_score("U1", "final", 100, DEADLINE)
    assert book.final_grade("U1") == pytest.approx(50)


def test_weights_must_add_up_to_one():
    with pytest.raises(ValueError):
        Gradebook([Assessment("midterm", 0.5, DEADLINE)])


@pytest.mark.parametrize("grade, letter", [(95, "A"), (90, "A"), (89.9, "B"), (70, "C"), (60, "D"), (59, "F")])
def test_grade_scale(grade, letter):
    assert GradeScale().letter(grade) == letter
```

---

## Task 10 — Three thousand test SMS

File: `task10.py` (run: `pytest task10.py`)

```python
# Lab 4A - Task 10: Three thousand test SMS
#
# 10.1 Today ReminderService CREATES its SmsClient in __init__ and keeps it in a field. That is
#      COMPOSITION (the service owns the client and builds it itself). It is the wrong choice:
#      - Lifetime: an SMS client is an external resource (HTTP connection, credentials) that lives
#        independently and may be shared by many services; it is not a "part" of ReminderService.
#      - Coupling: the service is welded to ONE concrete class and to ONE production key. A test
#        cannot put a fake in its place, so the test run sent real SMS.
#      The right relationship is ASSOCIATION by dependency injection: the service HOLDS a reference
#      to a client that somebody else created and gave to it.
#
# 10.2 ReminderService(sms) receives the client in the constructor. The real SmsClient is created in
#      ONE place, the "composition root" - the start-up code of the application
#      (create_production_service() below, called from main). Tests never call it.
#      The API key is read from the environment variable SMS_API_KEY, never from the source code.
#
# 10.4 Another production problem: there is no error handling. If ONE send() fails (network error,
#      provider timeout), the exception leaves the for-loop and every patient after that one gets no
#      reminder - and nobody notices. Now remind() catches the error per appointment, logs it and
#      returns the list of failed phones, so the job can retry them.
#      (Also worth fixing next: running the job twice sends every reminder twice - no idempotency;
#      and phone numbers are personal data and should not be written to logs in full.)

import logging
import os

import pytest

log = logging.getLogger(__name__)


class SmsClient:
    def __init__(self, api_key):
        self.api_key = api_key

    def send(self, phone, text):
        # in production this calls the SMS provider's HTTP API
        print(f"REAL SMS to {phone}: {text}")


class ReminderService:
    def __init__(self, sms):
        self.sms = sms                      # given from outside, never created here

    def remind(self, appointments):
        failed = []
        for a in appointments:
            text = f"Reminder: your visit tomorrow at {a['time']}"
            try:
                self.sms.send(a["phone"], text)
            except Exception:               # one broken send must not stop the others
                log.exception("SMS to a patient failed")
                failed.append(a["phone"])
        return failed


def create_production_service():
    """The composition root: the only place that builds the real SmsClient."""
    return ReminderService(SmsClient(api_key=os.environ["SMS_API_KEY"]))


# --- 10.3 fake + tests ---------------------------------------------------------------------------
class FakeSmsClient:
    def __init__(self, fail_for=()):
        self.sent = []                      # (phone, text)
        self.fail_for = set(fail_for)

    def send(self, phone, text):
        if phone in self.fail_for:
            raise ConnectionError("provider is down")
        self.sent.append((phone, text))


APPOINTMENTS = [
    {"phone": "+998901112233", "time": "09:00"},
    {"phone": "+998935556677", "time": "14:30"},
]


def test_two_appointments_two_messages(capsys):
    fake = FakeSmsClient()
    ReminderService(fake).remind(APPOINTMENTS)

    assert fake.sent == [
        ("+998901112233", "Reminder: your visit tomorrow at 09:00"),
        ("+998935556677", "Reminder: your visit tomorrow at 14:30"),
    ]
    assert "REAL SMS" not in capsys.readouterr().out


def test_one_failed_send_does_not_stop_the_others():
    fake = FakeSmsClient(fail_for=["+998901112233"])
    failed = ReminderService(fake).remind(APPOINTMENTS)

    assert failed == ["+998901112233"]
    assert [phone for phone, _ in fake.sent] == ["+998935556677"]


def test_service_does_not_create_a_client_itself():
    with pytest.raises(TypeError):
        ReminderService()                   # a client is required


def test_no_api_key_in_the_source_code():
    old_key = "prod" + "-key-7f3a91"        # split, so this test does not contain the key itself
    source = open(__file__, encoding="utf-8").read()
    assert old_key not in source
```

---

## E1 — Someone changed the settings at runtime

File: `extra1.py` (run: `pytest extra1.py`)

```python
# Lab 4A - Extra E1: Someone changed the settings at runtime
#
# E1.3 settings.max_upload_mb = 500 raises dataclasses.FrozenInstanceError
#      (it is a subclass of AttributeError). The shared object cannot be changed after start-up.
#
# E1.4 dataclasses.replace(settings, max_upload_mb=...) builds a NEW object by calling __init__,
#      and __init__ of a dataclass calls __post_init__. So YES, replace() runs the checks again:
#      replace(settings, max_upload_mb=500) raises ValueError (see the test). The handler gets its own
#      checked copy; the shared `settings` stays the same for everybody else.

from dataclasses import FrozenInstanceError, dataclass, replace

import pytest

ENVIRONMENTS = ("dev", "staging", "prod")


@dataclass(frozen=True)
class Settings:
    env: str
    max_upload_mb: int
    db_url: str

    def __post_init__(self):          # E1.2: a wrong configuration stops the program at start-up
        if self.env not in ENVIRONMENTS:
            raise ValueError(f"env must be one of {ENVIRONMENTS}, got {self.env!r}")
        if isinstance(self.max_upload_mb, bool) or not isinstance(self.max_upload_mb, int):
            raise ValueError("max_upload_mb must be an int")
        if not 1 <= self.max_upload_mb <= 100:
            raise ValueError(f"max_upload_mb must be from 1 to 100, got {self.max_upload_mb}")
        if not isinstance(self.db_url, str) or not self.db_url:
            raise ValueError("db_url must be a non-empty string")


def make_settings():
    return Settings(env="dev", max_upload_mb=20, db_url="postgresql://localhost/app")


# --- tests ---------------------------------------------------------------------------------------
def test_changing_a_field_raises():
    settings = make_settings()
    with pytest.raises(FrozenInstanceError):
        settings.max_upload_mb = 500
    with pytest.raises(AttributeError):          # FrozenInstanceError is an AttributeError
        settings.env = "prod"
    assert settings.max_upload_mb == 20


@pytest.mark.parametrize("bad", [0, 101, 500, -1, True, 10.5])
def test_wrong_limit_fails_at_startup(bad):
    with pytest.raises(ValueError):
        Settings("dev", bad, "postgresql://localhost/app")


def test_wrong_env_fails_at_startup():
    with pytest.raises(ValueError):
        Settings("production", 20, "postgresql://localhost/app")


def test_limits_are_inclusive():
    assert Settings("prod", 1, "x").max_upload_mb == 1
    assert Settings("prod", 100, "x").max_upload_mb == 100


def test_replace_returns_a_checked_copy():
    settings = make_settings()
    big = replace(settings, max_upload_mb=100)
    assert big.max_upload_mb == 100
    assert settings.max_upload_mb == 20          # the shared object is untouched
    assert big is not settings

    with pytest.raises(ValueError):              # replace() runs __post_init__ again
        replace(settings, max_upload_mb=500)
    with pytest.raises(ValueError):
        replace(settings, env="nope")
```

---

## E2 — API tokens in the error tracker

File: `extra2.py` (run: `pytest extra2.py`)

```python
# Lab 4A - Extra E2: API tokens in the error tracker
#
# E2.1 The generated repr of the original User prints EVERY field:
#        User(user_id=7, email='ali@example.com', password_hash='$2b$12$...', api_token='tok_abc123...9f2a')
#      Leaks: password_hash (can be attacked offline), api_token (full access to the account),
#      and the e-mail (personal data, should not be spread into logs without a reason).
#
# E2.3 __str__ is for people (UI), __repr__ is for developers (logs). Yes, they should differ here:
#      __str__ shows only what a person sees on screen: the e-mail.
#      __repr__ shows what a developer needs to find the record: user_id, plus the masked token.
#      Neither of them contains a secret.
#
# E2.4 field(repr=False) does NOT remove a field from ==. repr and compare are independent switches:
#      the default is compare=True. For User it matters: with compare=True two objects of the same
#      user would be "different" after a token rotation, and == would compare secrets (not in
#      constant time). So the secrets also get compare=False.

from dataclasses import dataclass, field

import pytest


def mask_token(token):
    """'tok_abc123XYZ9f2a' -> 'tok_****9f2a'. Too short tokens are hidden completely."""
    if len(token) < 8:
        return "****"
    prefix, sep, _ = token.partition("_")
    prefix = prefix + sep if sep and len(prefix) <= 5 else ""
    return f"{prefix}****{token[-4:]}"


@dataclass(repr=False)
class User:
    user_id: int
    email: str
    password_hash: str = field(repr=False, compare=False)
    api_token: str = field(repr=False, compare=False)

    def __str__(self):                  # for people
        return self.email

    def __repr__(self):                 # for developers / logs
        return f"User(user_id={self.user_id!r}, api_token={mask_token(self.api_token)})"


# --- tests ---------------------------------------------------------------------------------------
def make_user():
    return User(7, "ali@example.com", "$2b$12$verysecrethash", "tok_abc123XYZ9f2a")


def test_no_secret_in_repr_or_str():
    user = make_user()
    for text in (repr(user), str(user), "%r" % user, f"{user!r}", repr([user])):
        assert "verysecrethash" not in text
        assert "abc123XYZ" not in text
        assert "$2b$" not in text


def test_token_shows_only_last_four_characters():
    assert "tok_****9f2a" in repr(make_user())
    assert mask_token("tok_abc123XYZ9f2a") == "tok_****9f2a"
    assert mask_token("abcdefghij1234") == "****1234"
    assert mask_token("short") == "****"


def test_str_and_repr_are_different():
    user = make_user()
    assert str(user) == "ali@example.com"
    assert repr(user) == "User(user_id=7, api_token=tok_****9f2a)"


def test_repr_false_alone_does_not_remove_field_from_eq():     # E2.4
    @dataclass
    class Plain:
        name: str
        secret: str = field(repr=False)

    assert Plain("a", "1") != Plain("a", "2")                  # still compared!
    assert "secret" not in repr(Plain("a", "1"))


def test_user_equality_ignores_secrets():
    a = make_user()
    b = User(7, "ali@example.com", "other-hash", "tok_newtoken0000")
    assert a == b                                              # same user after a token rotation
```

---

## E3 — One seat, two tickets

File: `extra3.py` (run: `pytest extra3.py`)

```python
# Lab 4A - Extra E3: One seat, two tickets
#
# E3.1 Session must enforce "a Seat has 0..1 Ticket per Session". Who has the data? Session is the
#      only class that knows ALL tickets of one session. A Seat or a Ticket knows only itself, and
#      a Customer knows only his own tickets - none of them can see a conflict.
#
# E3.4 No, the object check is not enough. Four API servers load four separate Session objects, and
#      each one only sees ITS OWN copy: two servers can both think seat 5A is free and both sell it
#      (a race between "check" and "write" across processes). The DATABASE is the only place that
#      sees all tickets, so it must add a UNIQUE constraint:
#          UNIQUE (session_id, seat)
#      The second INSERT then fails with an integrity error, which the application turns into
#      SeatTakenError. (A row lock - SELECT ... FOR UPDATE - inside a transaction also works.)
#      The check in Session.book() stays: it gives fast, friendly errors inside one process.

import sqlite3
import threading
from dataclasses import dataclass

import pytest


class SeatTakenError(Exception):
    pass


@dataclass(frozen=True)
class Ticket:
    session_id: int
    seat: str
    customer: str


class Session:
    def __init__(self, session_id, seats):
        self.session_id = session_id
        self._seats = frozenset(seats)
        self._tickets = {}                 # seat -> Ticket   (at most one per seat)
        self._lock = threading.Lock()      # "check, then write" must be one step inside this process

    def book(self, seat, customer):
        if seat not in self._seats:
            raise ValueError(f"no such seat: {seat}")
        with self._lock:
            if seat in self._tickets:
                raise SeatTakenError(f"seat {seat} is already sold")
            ticket = Ticket(self.session_id, seat, customer)
            self._tickets[seat] = ticket
            return ticket


# --- tests ---------------------------------------------------------------------------------------
def test_double_booking_raises():
    session = Session(1, ["5A", "5B"])
    ticket = session.book("5A", "Ali")
    assert ticket == Ticket(1, "5A", "Ali")
    with pytest.raises(SeatTakenError):
        session.book("5A", "Vali")
    assert session.book("5B", "Vali").seat == "5B"   # another seat is fine


def test_unknown_seat_is_rejected():
    with pytest.raises(ValueError):
        Session(1, ["5A"]).book("99Z", "Ali")


def test_same_seat_in_another_session_is_fine():
    assert Session(1, ["5A"]).book("5A", "Ali")
    assert Session(2, ["5A"]).book("5A", "Vali")


def test_threads_cannot_double_book():
    session = Session(1, ["5A"])
    results = []

    def try_book(name):
        try:
            session.book("5A", name)
            results.append("ok")
        except SeatTakenError:
            results.append("taken")

    threads = [threading.Thread(target=try_book, args=(f"c{i}",)) for i in range(20)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    assert results.count("ok") == 1 and results.count("taken") == 19


def test_database_unique_constraint_is_the_real_protection():
    """Two 'servers' (two connections' worth of objects) cannot both sell one seat."""
    db = sqlite3.connect(":memory:")
    db.execute("CREATE TABLE tickets (session_id INT, seat TEXT, customer TEXT, "
               "UNIQUE (session_id, seat))")
    db.execute("INSERT INTO tickets VALUES (1, '5A', 'Ali')")       # server 1
    with pytest.raises(sqlite3.IntegrityError):                      # server 2, its own Session object
        db.execute("INSERT INTO tickets VALUES (1, '5A', 'Vali')")
    assert db.execute("SELECT COUNT(*) FROM tickets").fetchone()[0] == 1
```

---

## E4 — The lost phone number

File: `extra4.py` (run: `pytest extra4.py`)

```python
# Lab 4A - Extra E4: The lost phone number
#
# E4.1 Reproduced in test_lost_update_without_identity_map: two screens hold two DIFFERENT Customer
#      objects for id 42. Screen 1 saves the new phone; screen 2 then saves its object, which still
#      has the old phone (and the old name), and overwrites the change silently.
#
# E4.2 The repository keeps a dict {customer_id: Customer} (an identity map). get(42) returns the
#      same object every time, so both screens work on ONE object and there is nothing stale to save.
#
# E4.3 AGGREGATION. Lifetime test: "does the part disappear with the whole?" No. If the repository is
#      thrown away, a screen that still holds a Customer keeps working with it, and the customer's
#      data lives in the database anyway. The repository collects and hands out customers, it does
#      not own their life. (Composition would mean the customers die with the repository.)
#
# E4.4 Optimistic locking: keep a `version` number on every row. A save does
#          UPDATE customers SET phone=?, version=version+1 WHERE id=? AND version=?
#      If it changes 0 rows, somebody saved first -> raise a conflict error and let the user reload
#      or merge. It works across servers, because the check happens inside the database.


class Customer:
    def __init__(self, customer_id, name, phone):
        self.customer_id, self.name, self.phone = customer_id, name, phone


class PlainCustomerRepository:
    """The original: get() builds a new object every time."""

    def __init__(self, rows):
        self._rows = rows          # stands in for the database: id -> dict

    def get(self, customer_id):
        row = self._rows[customer_id]
        return Customer(customer_id, row["name"], row["phone"])

    def save(self, customer):
        self._rows[customer.customer_id] = {"name": customer.name, "phone": customer.phone}


class CustomerRepository:
    """E4.2: with an identity map - one object per customer_id inside this repository."""

    def __init__(self, rows):
        self._rows = rows
        self._identity_map = {}    # customer_id -> Customer

    def get(self, customer_id):
        customer = self._identity_map.get(customer_id)
        if customer is None:
            row = self._rows[customer_id]          # KeyError if the customer does not exist
            customer = Customer(customer_id, row["name"], row["phone"])
            self._identity_map[customer_id] = customer
        return customer

    def save(self, customer):
        self._rows[customer.customer_id] = {"name": customer.name, "phone": customer.phone}


# --- tests ---------------------------------------------------------------------------------------
def make_rows():
    return {42: {"name": "Ali", "phone": "+998901112233"}}


def test_lost_update_without_identity_map():          # E4.1
    rows = make_rows()
    repo = PlainCustomerRepository(rows)

    screen1 = repo.get(42)
    screen2 = repo.get(42)                           # opened earlier, holds the old phone
    screen1.phone = "+998935556677"
    repo.save(screen1)
    assert rows[42]["phone"] == "+998935556677"

    repo.save(screen2)                               # the second screen saves its old copy
    assert rows[42]["phone"] == "+998901112233"      # the change is LOST


def test_same_object_every_time():                   # E4.2
    repo = CustomerRepository(make_rows())
    assert repo.get(42) is repo.get(42)


def test_lost_update_cannot_happen_inside_one_repository():
    rows = make_rows()
    repo = CustomerRepository(rows)

    screen1 = repo.get(42)
    screen2 = repo.get(42)
    screen1.phone = "+998935556677"
    repo.save(screen1)
    repo.save(screen2)                               # it is the same object, nothing old to write

    assert rows[42]["phone"] == "+998935556677"
    assert screen2.phone == "+998935556677"


def test_different_customers_are_different_objects():
    rows = make_rows()
    rows[43] = {"name": "Vali", "phone": "+998900000000"}
    repo = CustomerRepository(rows)
    assert repo.get(42) is not repo.get(43)


def test_unknown_customer_raises_and_is_not_cached():
    repo = CustomerRepository(make_rows())
    try:
        repo.get(99)
        assert False, "expected KeyError"
    except KeyError:
        pass
    assert 99 not in repo._identity_map
```

---

## E5 — Appointments five hours late

File: `extra5.py` (run: `pytest extra5.py`)

```python
# Lab 4A - Extra E5: Appointments five hours late
#
# E5.1 Invariant: starts_at inside the object is ALWAYS a timezone-aware datetime in UTC.
#      Time zones are only for showing a time to a person (local_start), never for storing it.
#      A naive datetime has no meaning ("9:00 where?"), so it can never get inside.
#      On Windows run once:  pip install tzdata   (zoneinfo needs the time-zone database).
#
# E5.4 `now` is a parameter, not datetime.now() inside the method, for two reasons:
#      - the method becomes a pure function of its inputs, so a test can use fixed values of `now`
#        (with datetime.now() the test would give a different answer every minute);
#      - the clock is a dependency, like the SMS client in Task 10: the caller supplies it.

from datetime import datetime, timedelta, timezone
from zoneinfo import ZoneInfo

import pytest


def _to_utc(moment):
    if moment.tzinfo is None or moment.utcoffset() is None:
        raise ValueError("datetime must have a time zone (naive datetimes are not allowed)")
    return moment.astimezone(timezone.utc)


class Appointment:
    FREE_MOVE_BEFORE = timedelta(hours=24)   # the 24-hour rule from Task 3

    def __init__(self, patient, starts_at):
        self.patient = patient
        self._starts_at = _to_utc(starts_at)  # E5.2: naive -> ValueError; otherwise stored in UTC

    @property
    def starts_at(self):
        return self._starts_at

    def local_start(self, tz_name):           # E5.3
        return self._starts_at.astimezone(ZoneInfo(tz_name))

    def is_movable(self, now):
        """True while moving is FREE: at least 24 hours (inclusive) before the start."""
        return self._starts_at - _to_utc(now) >= self.FREE_MOVE_BEFORE


# --- tests ---------------------------------------------------------------------------------------
TASHKENT = ZoneInfo("Asia/Tashkent")


def test_naive_datetime_is_rejected():
    with pytest.raises(ValueError):
        Appointment("Ali", datetime(2026, 11, 3, 9, 0))


def test_tashkent_nine_is_stored_as_four_utc():
    a = Appointment("Ali", datetime(2026, 11, 3, 9, 0, tzinfo=TASHKENT))
    assert a.starts_at == datetime(2026, 11, 3, 4, 0, tzinfo=timezone.utc)
    assert a.starts_at.utcoffset() == timedelta(0)
    assert (a.starts_at.hour, a.starts_at.tzinfo) == (4, timezone.utc)


def test_local_start_shows_nine_in_tashkent():
    a = Appointment("Ali", datetime(2026, 11, 3, 4, 0, tzinfo=timezone.utc))
    local = a.local_start("Asia/Tashkent")
    assert (local.year, local.month, local.day, local.hour, local.minute) == (2026, 11, 3, 9, 0)


def test_same_moment_from_different_zones_is_equal():
    a = Appointment("Ali", datetime(2026, 11, 3, 9, 0, tzinfo=TASHKENT))
    b = Appointment("Ali", datetime(2026, 11, 3, 4, 0, tzinfo=timezone.utc))
    assert a.starts_at == b.starts_at


# the appointment starts 2026-11-03 09:00 Tashkent = 04:00 UTC
@pytest.mark.parametrize(
    "now_utc, movable",
    [
        (datetime(2026, 11, 2, 3, 59, tzinfo=timezone.utc), True),    # 24 h 1 min before
        (datetime(2026, 11, 2, 4, 0, tzinfo=timezone.utc), True),     # exactly 24 h before: still free
        (datetime(2026, 11, 2, 4, 1, tzinfo=timezone.utc), False),    # 23 h 59 min before: fee
        (datetime(2026, 11, 3, 5, 0, tzinfo=timezone.utc), False),    # already started
    ],
)
def test_24_hour_rule_with_fixed_now(now_utc, movable):
    a = Appointment("Ali", datetime(2026, 11, 3, 9, 0, tzinfo=TASHKENT))
    assert a.is_movable(now_utc) is movable


def test_now_can_be_given_in_any_zone_but_not_naive():
    a = Appointment("Ali", datetime(2026, 11, 3, 9, 0, tzinfo=TASHKENT))
    assert a.is_movable(datetime(2026, 11, 1, 9, 0, tzinfo=TASHKENT)) is True
    with pytest.raises(ValueError):
        a.is_movable(datetime(2026, 11, 1, 9, 0))
```
