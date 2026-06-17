"""
SHEHaven Bedding Stock Management System
Streamlit + Supabase/PostgreSQL version

What this version does:
- Uses Supabase/PostgreSQL instead of local SQLite.
- Saves products, added stock, sales, credit sales, payments, expenses, and corrections online.
- Has no hard maximum limits on amount paid, selling price, stock quantity, expenses, or costs.
- Includes an Admin correction section to reverse/delete incorrectly entered stock, sales, expenses, and products.
- Product names are no longer entered manually; the app generates internal descriptions and product codes automatically.
- Stock value is treated as the cost of acquiring stock for profit/dashboard purposes.
- Includes a client ordering portal and staff pending-order management in one deployed app.
- Email and WhatsApp notification hooks are intentionally left out for a later build.

Required Streamlit secret:
SUPABASE_DB_URL = "postgresql://..."  # the app converts this to the psycopg driver automatically
"""

from __future__ import annotations

import hashlib
import os
import re
from datetime import date, datetime, timedelta
from decimal import Decimal, InvalidOperation
from io import BytesIO
from typing import Any, Optional
from urllib.parse import urlparse, urlunparse

import pandas as pd
import streamlit as st
from sqlalchemy import create_engine, text
from sqlalchemy.engine import Engine


# ------------------------------------------------------------
# App constants
# ------------------------------------------------------------
APP_NAME = "SHEHaven Bedding Stock Management System"
APP_SHORT = "SHEHaven Bedding"

STOCK_TYPES = [
    "Blankets",
    "Fleece Blankets",
    "Winter Sheets",
    "Fleece Comforters",
    "Velvet Quilts",
    "Throws",
    "Summer Sheets",
    "Duvet Covers",
    "Mattress Protectors",
    "Other",
]

BRANDS = [
    "No Brand",
    "Mooi mooi",
    "Paris",
    "Pine leaf",
    "Zoya",
    "Orchid",
    "London",
    "Pandora",
    "Electric blankets",
    "Fashion",
    "Pine leaf/Golden",
    "Multi-colored",
    "Plains",
    "Bamboo",
    "Sabana",
    "Other",
]

SIZES = ["Single", "Double", "Queen", "King", "Super King", "One-sized", "1ply", "2ply", "3ply", "Other"]
PAYMENT_TYPES = ["Cash Sale", "Credit Sale"]
CREDIT_TERMS = ["1 Month", "2 Months"]
ROLES = ["Admin", "Manager", "Storekeeper", "Sales User", "Viewer"]
EXPENSE_CATEGORIES = [
    "Rent",
    "Transport",
    "Packaging",
    "Wages",
    "Utilities",
    "Marketing",
    "Repairs & Maintenance",
    "Bank Charges",
    "Cleaning",
    "Other",
]
PAYMENT_METHODS = ["Cash", "EcoCash", "Bank Transfer", "Swipe", "Other"]

ORDER_STATUSES = [
    "Pending",
    "Contacted",
    "Confirmed",
    "Ready for Collection",
    "Out for Delivery",
    "Delivered",
    "Cancelled",
]
ORDER_METHODS = ["Collect In Store", "Delivery"]
CLIENT_PAYMENT_OPTIONS = ["Cash on Collection", "Cash on Delivery", "EcoCash", "Bank Transfer", "To Be Confirmed"]


# ------------------------------------------------------------
# General helpers
# ------------------------------------------------------------
def hash_password(password: str) -> str:
    return hashlib.sha256(password.encode("utf-8")).hexdigest()


def clean_db_url(raw_url: str) -> str:
    """Prepare a Supabase/Postgres URL for SQLAlchemy.

    This deploy build uses psycopg v3 instead of psycopg2 because it installs
    more reliably on Streamlit Cloud's newer Python versions.
    """
    if not raw_url:
        return raw_url
    url = raw_url.strip().strip('"').strip("'")

    # Supabase may provide postgres:// or postgresql://. SQLAlchemy needs a driver.
    if url.startswith("postgres://"):
        url = "postgresql://" + url[len("postgres://") :]
    if url.startswith("postgresql+psycopg2://"):
        url = "postgresql+psycopg://" + url[len("postgresql+psycopg2://") :]
    elif url.startswith("postgresql://"):
        url = "postgresql+psycopg://" + url[len("postgresql://") :]

    # Supabase requires SSL. Do not double-add if already present.
    parsed = urlparse(url)
    if parsed.scheme.startswith("postgresql") and "sslmode=" not in parsed.query:
        query = (parsed.query + "&" if parsed.query else "") + "sslmode=require"
        url = urlunparse(parsed._replace(query=query))
    return url


def get_secret(name: str, default: str = "") -> str:
    try:
        if name in st.secrets:
            return str(st.secrets[name])
    except Exception:
        pass
    return os.environ.get(name, default)


@st.cache_resource(show_spinner=False)
def get_engine() -> Engine:
    db_url = clean_db_url(get_secret("SUPABASE_DB_URL"))
    if not db_url:
        raise RuntimeError(
            "Missing SUPABASE_DB_URL. Add it in Streamlit Cloud under App → Settings → Secrets."
        )
    return create_engine(db_url, pool_pre_ping=True, pool_size=5, max_overflow=10)


def exec_sql(sql: str, params: Optional[dict[str, Any]] = None) -> None:
    with get_engine().begin() as conn:
        conn.execute(text(sql), params or {})


def fetch_df(sql: str, params: Optional[dict[str, Any]] = None) -> pd.DataFrame:
    with get_engine().connect() as conn:
        return pd.read_sql_query(text(sql), conn, params=params or {})


def fetch_one(sql: str, params: Optional[dict[str, Any]] = None) -> Optional[dict[str, Any]]:
    with get_engine().connect() as conn:
        row = conn.execute(text(sql), params or {}).mappings().first()
        return dict(row) if row else None


def parse_decimal(value: Any, field_name: str = "value", allow_blank: bool = False) -> Decimal:
    """
    Parse money/quantity fields from text.
    No hard maximum limit is applied.
    """
    if value is None:
        value = ""
    raw = str(value).replace(",", "").strip()
    if raw == "":
        if allow_blank:
            return Decimal("0")
        raise ValueError(f"Please enter {field_name}.")
    try:
        return Decimal(raw)
    except (InvalidOperation, ValueError):
        raise ValueError(f"{field_name} must be a valid number.")


def money(value: Any) -> str:
    try:
        d = Decimal(str(value or 0))
    except Exception:
        d = Decimal("0")
    return f"${d:,.2f}"


def qty_fmt(value: Any) -> str:
    try:
        d = Decimal(str(value or 0))
    except Exception:
        d = Decimal("0")
    if d == d.to_integral_value():
        return f"{int(d):,}"
    return f"{d:,.2f}"


def now_ts() -> datetime:
    return datetime.now()


def refresh_app() -> None:
    """Clear cached dashboard/product data after a write, then refresh the app."""
    try:
        st.cache_data.clear()
    except Exception:
        pass
    st.rerun()


def due_status(due_date_value: Any, balance: Any) -> str:
    try:
        bal = Decimal(str(balance or 0))
    except Exception:
        bal = Decimal("0")
    if bal <= 0:
        return "Paid"

    if due_date_value is None or pd.isna(due_date_value):
        return "No Due Date"
    if isinstance(due_date_value, str):
        d = pd.to_datetime(due_date_value).date()
    elif isinstance(due_date_value, datetime):
        d = due_date_value.date()
    else:
        d = due_date_value

    today = date.today()
    if d < today:
        return "Overdue"
    if d == today:
        return "Due Today"
    if d <= today + timedelta(days=7):
        return "Due Soon"
    return "Not Yet Due"


def audit(action: str, table_name: str = "", record_id: Any = None, details: str = "") -> None:
    user = st.session_state.get("user", {})
    exec_sql(
        """
        INSERT INTO audit_log(action, table_name, record_id, details, created_by, created_at)
        VALUES(:action, :table_name, :record_id, :details, :created_by, NOW())
        """,
        {
            "action": action,
            "table_name": table_name,
            "record_id": str(record_id or ""),
            "details": details,
            "created_by": user.get("username", "system"),
        },
    )


def can_access(page: str) -> bool:
    role = st.session_state.get("user", {}).get("role", "Viewer")
    if role == "Admin":
        return True
    rules = {
        "Dashboard": ["Manager", "Storekeeper", "Sales User", "Viewer"],
        "Products": ["Manager", "Storekeeper"],
        "Add Stock": ["Manager", "Storekeeper"],
        "Sales": ["Manager", "Sales User"],
        "Credit Reminders": ["Manager", "Sales User"],
        "Client Orders": ["Manager", "Sales User", "Storekeeper"],
        "Expenses": ["Manager"],
        "Corrections / Delete": [],
        "Reports": ["Manager", "Viewer"],
        "User Management": [],
    }
    return role in rules.get(page, [])


# ------------------------------------------------------------
# Database schema
# ------------------------------------------------------------
@st.cache_resource(show_spinner=False)
def init_db() -> None:
    schema = """
    CREATE TABLE IF NOT EXISTS users (
        user_id SERIAL PRIMARY KEY,
        full_name TEXT NOT NULL,
        username TEXT UNIQUE NOT NULL,
        password_hash TEXT NOT NULL,
        role TEXT NOT NULL DEFAULT 'Viewer',
        active BOOLEAN NOT NULL DEFAULT TRUE,
        created_at TIMESTAMP NOT NULL DEFAULT NOW()
    );

    CREATE TABLE IF NOT EXISTS items (
        item_id SERIAL PRIMARY KEY,
        item_code TEXT UNIQUE NOT NULL,
        stock_type TEXT NOT NULL DEFAULT 'Other',
        item_name TEXT NOT NULL,
        brand TEXT,
        size TEXT,
        colour TEXT,
        cost_price NUMERIC NOT NULL DEFAULT 0,
        selling_price NUMERIC NOT NULL DEFAULT 0,
        current_stock NUMERIC NOT NULL DEFAULT 0,
        reorder_level NUMERIC NOT NULL DEFAULT 0,
        active BOOLEAN NOT NULL DEFAULT TRUE,
        created_at TIMESTAMP NOT NULL DEFAULT NOW(),
        updated_at TIMESTAMP NOT NULL DEFAULT NOW()
    );

    CREATE TABLE IF NOT EXISTS stock_movements (
        movement_id SERIAL PRIMARY KEY,
        item_id INTEGER REFERENCES items(item_id),
        movement_type TEXT NOT NULL,
        qty NUMERIC NOT NULL DEFAULT 0,
        unit_price NUMERIC NOT NULL DEFAULT 0,
        total_amount NUMERIC NOT NULL DEFAULT 0,
        reference_no TEXT,
        notes TEXT,
        created_by TEXT,
        created_at TIMESTAMP NOT NULL DEFAULT NOW(),
        is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
        deleted_at TIMESTAMP,
        deleted_by TEXT,
        delete_reason TEXT
    );

    CREATE TABLE IF NOT EXISTS sales (
        sale_id SERIAL PRIMARY KEY,
        movement_id INTEGER REFERENCES stock_movements(movement_id),
        item_id INTEGER REFERENCES items(item_id),
        sale_date DATE NOT NULL DEFAULT CURRENT_DATE,
        payment_type TEXT NOT NULL,
        customer_name TEXT,
        member_name TEXT,
        mobile_number TEXT,
        receipt_no TEXT,
        qty NUMERIC NOT NULL DEFAULT 0,
        selling_price NUMERIC NOT NULL DEFAULT 0,
        total_amount NUMERIC NOT NULL DEFAULT 0,
        amount_paid NUMERIC NOT NULL DEFAULT 0,
        balance_due NUMERIC NOT NULL DEFAULT 0,
        credit_term TEXT,
        due_date DATE,
        payment_status TEXT NOT NULL DEFAULT 'Outstanding',
        notes TEXT,
        created_by TEXT,
        created_at TIMESTAMP NOT NULL DEFAULT NOW(),
        is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
        deleted_at TIMESTAMP,
        deleted_by TEXT,
        delete_reason TEXT
    );

    CREATE TABLE IF NOT EXISTS credit_payments (
        payment_id SERIAL PRIMARY KEY,
        sale_id INTEGER REFERENCES sales(sale_id),
        payment_date DATE NOT NULL DEFAULT CURRENT_DATE,
        amount_paid NUMERIC NOT NULL DEFAULT 0,
        payment_method TEXT,
        reference_no TEXT,
        notes TEXT,
        captured_by TEXT,
        created_at TIMESTAMP NOT NULL DEFAULT NOW(),
        is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
        deleted_at TIMESTAMP,
        deleted_by TEXT,
        delete_reason TEXT
    );

    CREATE TABLE IF NOT EXISTS expenses (
        expense_id SERIAL PRIMARY KEY,
        expense_date DATE NOT NULL DEFAULT CURRENT_DATE,
        category TEXT NOT NULL,
        amount NUMERIC NOT NULL DEFAULT 0,
        payment_method TEXT,
        paid_to TEXT,
        reference_no TEXT,
        description TEXT,
        captured_by TEXT,
        created_at TIMESTAMP NOT NULL DEFAULT NOW(),
        is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
        deleted_at TIMESTAMP,
        deleted_by TEXT,
        delete_reason TEXT
    );

    CREATE TABLE IF NOT EXISTS stocktake (
        stocktake_id SERIAL PRIMARY KEY,
        item_id INTEGER REFERENCES items(item_id),
        system_qty NUMERIC NOT NULL DEFAULT 0,
        physical_qty NUMERIC NOT NULL DEFAULT 0,
        difference_qty NUMERIC NOT NULL DEFAULT 0,
        count_date DATE NOT NULL DEFAULT CURRENT_DATE,
        counted_by TEXT,
        notes TEXT,
        created_at TIMESTAMP NOT NULL DEFAULT NOW()
    );

    CREATE TABLE IF NOT EXISTS client_orders (
        order_id SERIAL PRIMARY KEY,
        order_no TEXT UNIQUE,
        order_date DATE NOT NULL DEFAULT CURRENT_DATE,
        customer_name TEXT NOT NULL,
        mobile_number TEXT NOT NULL,
        order_method TEXT NOT NULL DEFAULT 'Collect In Store',
        delivery_address TEXT,
        preferred_payment TEXT,
        notes TEXT,
        order_total NUMERIC NOT NULL DEFAULT 0,
        status TEXT NOT NULL DEFAULT 'Pending',
        created_at TIMESTAMP NOT NULL DEFAULT NOW(),
        updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
        is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
        deleted_at TIMESTAMP,
        deleted_by TEXT,
        delete_reason TEXT
    );

    CREATE TABLE IF NOT EXISTS client_order_items (
        order_item_id SERIAL PRIMARY KEY,
        order_id INTEGER REFERENCES client_orders(order_id) ON DELETE CASCADE,
        item_id INTEGER REFERENCES items(item_id),
        item_code TEXT,
        item_description TEXT,
        qty NUMERIC NOT NULL DEFAULT 0,
        unit_price NUMERIC NOT NULL DEFAULT 0,
        line_total NUMERIC NOT NULL DEFAULT 0,
        created_at TIMESTAMP NOT NULL DEFAULT NOW()
    );

    CREATE TABLE IF NOT EXISTS order_status_history (
        history_id SERIAL PRIMARY KEY,
        order_id INTEGER REFERENCES client_orders(order_id) ON DELETE CASCADE,
        old_status TEXT,
        new_status TEXT NOT NULL,
        notes TEXT,
        changed_by TEXT,
        created_at TIMESTAMP NOT NULL DEFAULT NOW()
    );

    CREATE TABLE IF NOT EXISTS audit_log (
        audit_id SERIAL PRIMARY KEY,
        action TEXT NOT NULL,
        table_name TEXT,
        record_id TEXT,
        details TEXT,
        created_by TEXT,
        created_at TIMESTAMP NOT NULL DEFAULT NOW()
    );

    CREATE INDEX IF NOT EXISTS idx_items_active ON items(active);
    CREATE INDEX IF NOT EXISTS idx_sales_deleted ON sales(is_deleted);
    CREATE INDEX IF NOT EXISTS idx_sales_due_date ON sales(due_date);
    CREATE INDEX IF NOT EXISTS idx_movements_deleted ON stock_movements(is_deleted);
    CREATE INDEX IF NOT EXISTS idx_expenses_deleted ON expenses(is_deleted);
    CREATE INDEX IF NOT EXISTS idx_client_orders_status ON client_orders(status);
    CREATE INDEX IF NOT EXISTS idx_client_orders_mobile ON client_orders(mobile_number);
    CREATE INDEX IF NOT EXISTS idx_client_orders_deleted ON client_orders(is_deleted);
    CREATE INDEX IF NOT EXISTS idx_client_order_items_order ON client_order_items(order_id);
    """
    with get_engine().begin() as conn:
        # Execute each DDL statement separately. This is more reliable on hosted
        # Postgres/Supabase than sending one large multi-statement block.
        for statement in schema.split(';'):
            statement = statement.strip()
            if statement:
                conn.execute(text(statement))
        admin_exists = conn.execute(text("SELECT COUNT(*) FROM users WHERE username = 'admin'")).scalar()
        if not admin_exists:
            conn.execute(
                text(
                    """
                    INSERT INTO users(full_name, username, password_hash, role, active, created_at)
                    VALUES(:full_name, :username, :password_hash, 'Admin', TRUE, NOW())
                    """
                ),
                {
                    "full_name": "System Administrator",
                    "username": "admin",
                    "password_hash": hash_password("admin123"),
                },
            )

        # Cash-basis credit payment alignment:
        # Older versions stored credit deposits/paid amounts only inside sales.amount_paid.
        # For correct profit reporting, every paid credit amount must also exist in credit_payments
        # with a payment date, so unpaid credit balances are NOT counted as profit.
        conn.execute(
            text(
                """
                INSERT INTO credit_payments(
                    sale_id, payment_date, amount_paid, payment_method, reference_no,
                    notes, captured_by, created_at
                )
                SELECT
                    s.sale_id,
                    s.sale_date,
                    s.amount_paid - COALESCE(p.total_paid, 0) AS amount_paid,
                    'Opening/Existing Paid',
                    s.receipt_no,
                    'Auto-created so credit profit is based on paid amounts only',
                    'system',
                    NOW()
                FROM sales s
                LEFT JOIN (
                    SELECT sale_id, SUM(amount_paid) AS total_paid
                    FROM credit_payments
                    WHERE is_deleted = FALSE
                    GROUP BY sale_id
                ) p ON p.sale_id = s.sale_id
                WHERE s.is_deleted = FALSE
                  AND s.payment_type = 'Credit Sale'
                  AND s.amount_paid > COALESCE(p.total_paid, 0)
                """
            )
        )


# ------------------------------------------------------------
# Data access helpers
# ------------------------------------------------------------
@st.cache_data(ttl=120, show_spinner=False)
def get_active_items() -> pd.DataFrame:
    return fetch_df(
        """
        SELECT item_id, item_code, stock_type, item_name, brand, size, colour,
               cost_price, selling_price, current_stock, reorder_level, active
        FROM items
        WHERE active = TRUE
        ORDER BY stock_type, brand, size, colour, item_code
        """
    )


def product_label(row: pd.Series | dict[str, Any]) -> str:
    return f"{row['item_code']} | {row['stock_type']} | {row['brand'] or ''} | {row['size'] or ''} | {row.get('colour') or ''} | Stock: {qty_fmt(row['current_stock'])}"


def make_product_name(stock_type: str, brand: str, size: str, colour: str) -> str:
    """Internal product description stored for compatibility; not shown as a form field."""
    parts = [brand.strip(), stock_type.strip(), size.strip(), colour.strip()]
    cleaned = [x for x in parts if x]
    return " ".join(cleaned) or stock_type.strip() or "Product"


def display_products_df(df: pd.DataFrame) -> pd.DataFrame:
    """Hide internal product_name from user-facing product tables."""
    if df is None or df.empty:
        return df
    return df.drop(columns=["item_name"], errors="ignore")


def _code_part(value: str, fallback: str = "GEN", length: int = 4) -> str:
    """Return a clean uppercase code part for automatic product codes."""
    value = (value or "").strip()
    if not value:
        return fallback[:length].upper()
    cleaned = re.sub(r"[^A-Za-z0-9]+", " ", value).strip()
    if not cleaned:
        return fallback[:length].upper()
    words = cleaned.split()
    if len(words) >= 2:
        code = "".join(w[0] for w in words if w)
    else:
        code = words[0][:length]
    return (code or fallback).upper()[:length]


def stock_type_code(stock_type: str) -> str:
    mapping = {
        "Blankets": "BLK",
        "Fleece Blankets": "FBL",
        "Winter Sheets": "WSH",
        "Fleece Comforters": "FCM",
        "Velvet Quilts": "VQL",
        "Throws": "THR",
        "Summer Sheets": "SSH",
        "Duvet Covers": "DVT",
        "Mattress Protectors": "MTP",
        "Other": "OTH",
    }
    return mapping.get(stock_type, _code_part(stock_type, "OTH", 3))


def size_code(size: str) -> str:
    mapping = {
        "Single": "SG",
        "Double": "DB",
        "Queen": "QN",
        "King": "KG",
        "Super King": "SK",
        "One-sized": "OS",
        "1ply": "1P",
        "2ply": "2P",
        "3ply": "3P",
        "Other": "OT",
    }
    return mapping.get(size, _code_part(size, "OT", 2))


def generate_product_code(stock_type: str, brand: str, size: str) -> str:
    """Generate a unique product code such as BLK-MOOI-QN-001."""
    type_part = stock_type_code(stock_type)
    brand_part = _code_part(brand, "NBR", 4)
    size_part = size_code(size)
    prefix = f"{type_part}-{brand_part}-{size_part}"

    existing = fetch_df(
        "SELECT item_code FROM items WHERE item_code LIKE :pattern ORDER BY item_code",
        {"pattern": f"{prefix}-%"},
    )
    max_no = 0
    for code in existing.get("item_code", []):
        match = re.search(r"-(\d+)$", str(code))
        if match:
            max_no = max(max_no, int(match.group(1)))
    return f"{prefix}-{max_no + 1:03d}"


@st.cache_data(ttl=120, show_spinner=False)
def get_item(item_id: int) -> Optional[dict[str, Any]]:
    return fetch_one("SELECT * FROM items WHERE item_id = :item_id", {"item_id": item_id})


def update_payment_status_for_sale(sale_id: int) -> None:
    sale = fetch_one(
        "SELECT total_amount, amount_paid, balance_due FROM sales WHERE sale_id = :sale_id",
        {"sale_id": sale_id},
    )
    if not sale:
        return
    balance = Decimal(str(sale.get("balance_due") or 0))
    paid = Decimal(str(sale.get("amount_paid") or 0))
    if balance <= 0:
        status = "Paid"
    elif paid > 0:
        status = "Part Paid"
    else:
        status = "Outstanding"
    exec_sql("UPDATE sales SET payment_status = :status WHERE sale_id = :sale_id", {"status": status, "sale_id": sale_id})


# ------------------------------------------------------------
# Authentication
# ------------------------------------------------------------
def login_page() -> None:
    st.title("🛏️ SHEHaven Bedding")
    st.caption("Stock, sales, credit reminders, and expenses")

    with st.form("login_form"):
        username = st.text_input("Username")
        password = st.text_input("Password", type="password")
        submitted = st.form_submit_button("Login")

    if submitted:
        user = fetch_one(
            """
            SELECT user_id, full_name, username, role, active
            FROM users
            WHERE username = :username AND password_hash = :password_hash
            """,
            {"username": username.strip(), "password_hash": hash_password(password)},
        )
        if user and user["active"]:
            st.session_state["user"] = user
            st.success("Login successful")
            refresh_app()
        else:
            st.error("Invalid username or password")

    st.info("Default first login: username `admin`, password `admin123`. Change it after going live.")



@st.cache_data(ttl=30, show_spinner=False)
def get_dashboard_cards_cached() -> dict[str, Any]:
    return fetch_one(
        """
        SELECT
            COALESCE((SELECT SUM(current_stock) FROM items WHERE active = TRUE), 0) AS total_stock_qty,
            COALESCE((SELECT SUM(current_stock * cost_price) FROM items WHERE active = TRUE), 0) AS stock_cost_value,
            COALESCE((SELECT SUM(current_stock * selling_price) FROM items WHERE active = TRUE), 0) AS stock_selling_value,
            COALESCE((SELECT SUM(qty) FROM sales WHERE is_deleted = FALSE), 0) AS total_sales_qty,
            COALESCE((SELECT SUM(total_amount) FROM sales WHERE is_deleted = FALSE), 0) AS total_sales_value,
            COALESCE((SELECT SUM(total_amount) FROM sales WHERE is_deleted = FALSE AND payment_type = 'Cash Sale'), 0) AS cash_sales_value,
            COALESCE((SELECT SUM(total_amount) FROM sales WHERE is_deleted = FALSE AND payment_type = 'Credit Sale'), 0) AS credit_sales_value,
            COALESCE((SELECT SUM(amount_paid) FROM sales WHERE is_deleted = FALSE AND payment_type = 'Credit Sale'), 0) AS credit_paid_value,
            COALESCE((SELECT SUM(balance_due) FROM sales WHERE is_deleted = FALSE AND payment_type = 'Credit Sale'), 0) AS credit_balance,
            COALESCE((SELECT SUM(amount) FROM expenses WHERE is_deleted = FALSE), 0) AS total_expenses,
            COALESCE((
                SELECT SUM(total_amount)
                FROM stock_movements
                WHERE is_deleted = FALSE AND movement_type = 'Stock In'
            ), 0) AS stock_acquisition_cost_all,
            COALESCE((
                SELECT SUM(total_amount)
                FROM stock_movements
                WHERE is_deleted = FALSE
                  AND movement_type = 'Stock In'
                  AND date_trunc('month', created_at::timestamp) = date_trunc('month', CURRENT_DATE::timestamp)
            ), 0) AS stock_acquisition_cost_month,
            (
                COALESCE((
                    SELECT SUM(total_amount)
                    FROM sales
                    WHERE is_deleted = FALSE
                      AND payment_type = 'Cash Sale'
                      AND date_trunc('month', sale_date::timestamp) = date_trunc('month', CURRENT_DATE::timestamp)
                ), 0)
                +
                COALESCE((
                    SELECT SUM(p.amount_paid)
                    FROM credit_payments p
                    JOIN sales s ON s.sale_id = p.sale_id
                    WHERE p.is_deleted = FALSE
                      AND s.is_deleted = FALSE
                      AND date_trunc('month', p.payment_date::timestamp) = date_trunc('month', CURRENT_DATE::timestamp)
                ), 0)
            ) AS paid_sales_month,
            COALESCE((SELECT SUM(amount) FROM expenses WHERE is_deleted = FALSE AND date_trunc('month', expense_date::timestamp) = date_trunc('month', CURRENT_DATE::timestamp)), 0) AS expenses_month
        """
    ) or {}


@st.cache_data(ttl=30, show_spinner=False)
def get_low_stock_cached() -> pd.DataFrame:
    return fetch_df(
        """
        SELECT item_code, stock_type, brand, size, colour, current_stock, reorder_level
        FROM items
        WHERE active = TRUE AND current_stock <= reorder_level
        ORDER BY current_stock ASC
        LIMIT 50
        """
    )


@st.cache_data(ttl=30, show_spinner=False)
def get_recent_sales_cached() -> pd.DataFrame:
    return fetch_df(
        """
        SELECT s.sale_date, i.item_code, i.brand, i.stock_type, i.size, s.payment_type, s.member_name, s.mobile_number,
               s.qty, s.selling_price, s.total_amount, s.amount_paid, s.balance_due, s.payment_status
        FROM sales s
        JOIN items i ON i.item_id = s.item_id
        WHERE s.is_deleted = FALSE
        ORDER BY s.created_at DESC
        LIMIT 10
        """
    )

# ------------------------------------------------------------
# Pages
# ------------------------------------------------------------
def dashboard_page() -> None:
    st.header("Dashboard")

    cards = get_dashboard_cards_cached()

    cols = st.columns(4)
    cols[0].metric("Total Stock Qty", qty_fmt(cards.get("total_stock_qty")))
    cols[1].metric("Cost of Acquiring Current Stock", money(cards.get("stock_cost_value")))
    cols[2].metric("Potential Sales Value", money(cards.get("stock_selling_value")))
    cols[3].metric("Total Sales Qty", qty_fmt(cards.get("total_sales_qty")))

    cols = st.columns(4)
    cols[0].metric("Total Sales Value", money(cards.get("total_sales_value")))
    cols[1].metric("Cash Sales", money(cards.get("cash_sales_value")))
    cols[2].metric("Credit Sales", money(cards.get("credit_sales_value")))
    cols[3].metric("Credit Balance Due", money(cards.get("credit_balance")))

    paid_sales_month = Decimal(str(cards.get("paid_sales_month") or 0))
    expenses_month = Decimal(str(cards.get("expenses_month") or 0))
    stock_acquisition_cost_month = Decimal(str(cards.get("stock_acquisition_cost_month") or 0))
    net_month = paid_sales_month - expenses_month - stock_acquisition_cost_month

    cols = st.columns(4)
    cols[0].metric("Paid Sales This Month", money(paid_sales_month))
    cols[1].metric("Expenses This Month", money(expenses_month))
    cols[2].metric("Stock Acquisition Cost This Month", money(stock_acquisition_cost_month))
    cols[3].metric("Estimated Profit This Month", money(net_month))
    st.caption("Profit uses cash-basis logic: unpaid credit balances are excluded until payment is received. Stock value is treated as cost of acquiring stock when stock is added.")

    st.subheader("Low Stock / Reorder Alerts")
    low_stock = get_low_stock_cached()
    if low_stock.empty:
        st.success("No low stock alerts right now.")
    else:
        st.warning("These items have reached or passed their reorder level.")
        st.dataframe(low_stock, use_container_width=True, hide_index=True)

    st.subheader("Recent Sales")
    recent_sales = get_recent_sales_cached()
    st.dataframe(recent_sales, use_container_width=True, hide_index=True)

def products_page() -> None:
    st.header("Products")

    tab1, tab2, tab3 = st.tabs(["Add Product", "Product List", "Corrections / Delete"])

    with tab1:
        st.subheader("Add New Product")
        st.info("Product Code is generated automatically after you click Add Product.")
        with st.form("add_product_form"):
            c1, c2 = st.columns(2)
            stock_type = c1.selectbox("Stock Type", STOCK_TYPES)
            if stock_type == "Other":
                stock_type = c1.text_input("Type New Stock Type")

            brand_choice = c2.selectbox("Brand", BRANDS)
            brand = brand_choice
            if brand_choice == "Other":
                brand = c2.text_input("Type New Brand")
            elif brand_choice == "No Brand":
                brand = ""

            c1, c2 = st.columns(2)
            size_choice = c1.selectbox("Size", SIZES)
            size = size_choice
            if size_choice == "Other":
                size = c1.text_input("Type New Size")
            colour = c2.text_input("Colour")

            c1, c2, c3 = st.columns(3)
            cost_price_text = c1.text_input("Cost Price", value="0")
            selling_price_text = c2.text_input("Selling Price", value="0")
            reorder_level_text = c3.text_input("Reorder Level", value="0")

            submitted = st.form_submit_button("Add Product")

        if submitted:
            try:
                cost_price = parse_decimal(cost_price_text, "Cost Price", allow_blank=True)
                selling_price = parse_decimal(selling_price_text, "Selling Price", allow_blank=True)
                reorder_level = parse_decimal(reorder_level_text, "Reorder Level", allow_blank=True)
                if not stock_type.strip():
                    raise ValueError("Stock Type is required.")

                item_name = make_product_name(stock_type, brand, size, colour)
                item_code = generate_product_code(stock_type.strip(), brand.strip(), size.strip())
                exec_sql(
                    """
                    INSERT INTO items(item_code, stock_type, item_name, brand, size, colour, cost_price, selling_price, current_stock, reorder_level, active, created_at, updated_at)
                    VALUES(:item_code, :stock_type, :item_name, :brand, :size, :colour, :cost_price, :selling_price, 0, :reorder_level, TRUE, NOW(), NOW())
                    """,
                    {
                        "item_code": item_code,
                        "stock_type": stock_type.strip(),
                        "item_name": item_name,
                        "brand": brand.strip(),
                        "size": size.strip(),
                        "colour": colour.strip(),
                        "cost_price": cost_price,
                        "selling_price": selling_price,
                        "reorder_level": reorder_level,
                    },
                )
                audit("ADD_PRODUCT", "items", item_code, f"Added product; generated code {item_code}")
                st.success(f"Product added successfully. Generated Product Code: {item_code}")
                refresh_app()
            except Exception as e:
                st.error(str(e))

    with tab2:
        st.subheader("Active Products")
        df = get_active_items()
        st.dataframe(display_products_df(df), use_container_width=True, hide_index=True)

        if can_access("Products") and not df.empty:
            st.subheader("Quick Edit Prices / Reorder Level")
            labels = {product_label(row): int(row["item_id"]) for _, row in df.iterrows()}
            selected_label = st.selectbox("Select product to edit", list(labels.keys()))
            item_id = labels[selected_label]
            item = get_item(item_id)
            if item:
                with st.form("edit_product_form"):
                    c1, c2, c3 = st.columns(3)
                    cost = c1.text_input("Cost Price", value=str(item.get("cost_price") or 0))
                    sell = c2.text_input("Selling Price", value=str(item.get("selling_price") or 0))
                    reorder = c3.text_input("Reorder Level", value=str(item.get("reorder_level") or 0))
                    submitted_edit = st.form_submit_button("Update Product")
                if submitted_edit:
                    try:
                        exec_sql(
                            """
                            UPDATE items
                            SET cost_price = :cost, selling_price = :sell, reorder_level = :reorder, updated_at = NOW()
                            WHERE item_id = :item_id
                            """,
                            {
                                "cost": parse_decimal(cost, "Cost Price", allow_blank=True),
                                "sell": parse_decimal(sell, "Selling Price", allow_blank=True),
                                "reorder": parse_decimal(reorder, "Reorder Level", allow_blank=True),
                                "item_id": item_id,
                            },
                        )
                        audit("EDIT_PRODUCT", "items", item_id, "Updated prices/reorder level")
                        st.success("Product updated.")
                        refresh_app()
                    except Exception as e:
                        st.error(str(e))

    with tab3:
        st.subheader("Product Corrections / Delete")
        if not can_access("Corrections / Delete"):
            st.info("Only Admin users can delete or hide incorrectly entered products.")
        else:
            st.warning("Use this tab only for incorrectly entered products. Historical stock and sales records are kept safe.")
            products = get_active_items()
            st.dataframe(display_products_df(products), use_container_width=True, hide_index=True)
            if products.empty:
                st.success("No active products to correct.")
            else:
                labels = {product_label(row): int(row["item_id"]) for _, row in products.iterrows()}
                with st.form("product_tab_delete_form"):
                    selected = st.selectbox("Select product to hide/delete", list(labels.keys()))
                    reason = st.text_input("Reason")
                    confirm = st.checkbox("I confirm this product was entered incorrectly")
                    submitted_delete = st.form_submit_button("Hide Product")
                if submitted_delete:
                    if not confirm or not reason.strip():
                        st.error("Please confirm and enter a reason.")
                    else:
                        item_id = labels[selected]
                        exec_sql(
                            "UPDATE items SET active = FALSE, updated_at = NOW() WHERE item_id = :item_id",
                            {"item_id": item_id},
                        )
                        audit("HIDE_PRODUCT", "items", item_id, reason.strip())
                        st.success("Product hidden from active product lists. Historical records remain safe.")
                        refresh_app()

def add_stock_page() -> None:
    st.header("Add Stock")
    items = get_active_items()
    if items.empty:
        st.warning("Add products first before adding stock.")
        return

    labels = {product_label(row): int(row["item_id"]) for _, row in items.iterrows()}
    with st.form("add_stock_form"):
        selected_label = st.selectbox("Select Product", list(labels.keys()))
        item_id = labels[selected_label]
        item = get_item(item_id)
        st.info(f"Current stock: {qty_fmt(item['current_stock'])}")

        c1, c2, c3 = st.columns(3)
        qty_text = c1.text_input("Quantity to Add", placeholder="Enter any quantity")
        cost_price_text = c2.text_input("New/Updated Cost Price", value=str(item.get("cost_price") or 0))
        selling_price_text = c3.text_input("New/Updated Selling Price", value=str(item.get("selling_price") or 0))

        c1, c2 = st.columns(2)
        reference_no = c1.text_input("Reference / Invoice Number")
        notes = c2.text_input("Notes")

        submitted = st.form_submit_button("Add Stock")

    if submitted:
        try:
            qty = parse_decimal(qty_text, "Quantity to Add")
            cost_price = parse_decimal(cost_price_text, "Cost Price", allow_blank=True)
            selling_price = parse_decimal(selling_price_text, "Selling Price", allow_blank=True)
            total_cost = qty * cost_price
            user = st.session_state["user"]["username"]

            with get_engine().begin() as conn:
                conn.execute(
                    text(
                        """
                        UPDATE items
                        SET current_stock = current_stock + :qty,
                            cost_price = :cost_price,
                            selling_price = :selling_price,
                            updated_at = NOW()
                        WHERE item_id = :item_id
                        """
                    ),
                    {"qty": qty, "cost_price": cost_price, "selling_price": selling_price, "item_id": item_id},
                )
                conn.execute(
                    text(
                        """
                        INSERT INTO stock_movements(item_id, movement_type, qty, unit_price, total_amount, reference_no, notes, created_by, created_at)
                        VALUES(:item_id, 'Stock In', :qty, :unit_price, :total_amount, :reference_no, :notes, :created_by, NOW())
                        """
                    ),
                    {
                        "item_id": item_id,
                        "qty": qty,
                        "unit_price": cost_price,
                        "total_amount": total_cost,
                        "reference_no": reference_no.strip(),
                        "notes": notes.strip(),
                        "created_by": user,
                    },
                )
            audit("ADD_STOCK", "items", item_id, f"Added {qty} stock")
            st.success("Stock added successfully.")
            refresh_app()
        except Exception as e:
            st.error(str(e))


def sales_page() -> None:
    st.header("Sales")
    items = get_active_items()
    if items.empty:
        st.warning("Add products first before recording sales.")
        return

    labels = {product_label(row): int(row["item_id"]) for _, row in items.iterrows()}
    with st.form("sales_form"):
        selected_label = st.selectbox("Select Product Sold", list(labels.keys()))
        item_id = labels[selected_label]
        item = get_item(item_id)
        st.info(f"Current stock: {qty_fmt(item['current_stock'])}. The system has no hard maximum entry limit, but it will show negative stock if you sell more than available.")

        c1, c2, c3 = st.columns(3)
        sale_date = c1.date_input("Sale Date", value=date.today())
        qty_text = c2.text_input("Quantity Sold", placeholder="Enter quantity")
        selling_price_text = c3.text_input("Selling Price per Unit", value=str(item.get("selling_price") or 0))

        c1, c2, c3 = st.columns(3)
        payment_type = c1.selectbox("Payment Type", PAYMENT_TYPES)
        receipt_no = c2.text_input("Receipt / Reference Number")
        customer_name = c3.text_input("Customer Name / Optional")

        member_name = ""
        mobile_number = ""
        credit_term = None
        due_date = None
        amount_paid_text = "0"
        notes = ""

        if payment_type == "Credit Sale":
            st.markdown("#### Credit Customer Details")
            c1, c2 = st.columns(2)
            member_name = c1.text_input("Member Name")
            mobile_number = c2.text_input("Mobile Number")
            c1, c2, c3 = st.columns(3)
            credit_term = c1.selectbox("Credit Type", CREDIT_TERMS)
            default_due = sale_date + timedelta(days=30 if credit_term == "1 Month" else 60)
            due_date = c2.date_input("Due Date", value=default_due)
            amount_paid_text = c3.text_input("Deposit / Amount Paid Now", value="0", help="No hard maximum limit. You may enter any amount.")
            notes = st.text_area("Credit Notes")
        else:
            c1, c2 = st.columns(2)
            amount_paid_text = c1.text_input("Amount Paid", value="", placeholder="Leave blank to use full total")
            notes = c2.text_input("Notes")

        submitted = st.form_submit_button("Save Sale")

    if submitted:
        try:
            qty = parse_decimal(qty_text, "Quantity Sold")
            selling_price = parse_decimal(selling_price_text, "Selling Price")
            total = qty * selling_price

            if str(amount_paid_text).strip() == "" and payment_type == "Cash Sale":
                amount_paid = total
            else:
                amount_paid = parse_decimal(amount_paid_text, "Amount Paid", allow_blank=True)

            balance_due = total - amount_paid
            if payment_type == "Cash Sale":
                due_date_value = None
                credit_term_value = None
                status = "Paid" if balance_due <= 0 else "Part Paid"
            else:
                due_date_value = due_date
                credit_term_value = credit_term
                if not member_name.strip():
                    raise ValueError("Member Name is required for credit sales.")
                if not mobile_number.strip():
                    raise ValueError("Mobile Number is required for credit sales.")
                status = "Paid" if balance_due <= 0 else ("Part Paid" if amount_paid > 0 else "Outstanding")

            current_stock = Decimal(str(item.get("current_stock") or 0))
            if qty > current_stock:
                st.warning("You sold more than current available stock. The system will allow it and show negative stock because no hard limit is applied.")

            user = st.session_state["user"]["username"]

            with get_engine().begin() as conn:
                conn.execute(
                    text("UPDATE items SET current_stock = current_stock - :qty, updated_at = NOW() WHERE item_id = :item_id"),
                    {"qty": qty, "item_id": item_id},
                )
                movement_id = conn.execute(
                    text(
                        """
                        INSERT INTO stock_movements(item_id, movement_type, qty, unit_price, total_amount, reference_no, notes, created_by, created_at)
                        VALUES(:item_id, 'Sold', :qty, :unit_price, :total_amount, :reference_no, :notes, :created_by, NOW())
                        RETURNING movement_id
                        """
                    ),
                    {
                        "item_id": item_id,
                        "qty": qty,
                        "unit_price": selling_price,
                        "total_amount": total,
                        "reference_no": receipt_no.strip(),
                        "notes": notes.strip(),
                        "created_by": user,
                    },
                ).scalar_one()
                sale_id = conn.execute(
                    text(
                        """
                        INSERT INTO sales(
                            movement_id, item_id, sale_date, payment_type, customer_name, member_name, mobile_number,
                            receipt_no, qty, selling_price, total_amount, amount_paid, balance_due,
                            credit_term, due_date, payment_status, notes, created_by, created_at
                        )
                        VALUES(
                            :movement_id, :item_id, :sale_date, :payment_type, :customer_name, :member_name, :mobile_number,
                            :receipt_no, :qty, :selling_price, :total_amount, :amount_paid, :balance_due,
                            :credit_term, :due_date, :payment_status, :notes, :created_by, NOW()
                        )
                        RETURNING sale_id
                        """
                    ),
                    {
                        "movement_id": movement_id,
                        "item_id": item_id,
                        "sale_date": sale_date,
                        "payment_type": payment_type,
                        "customer_name": customer_name.strip(),
                        "member_name": member_name.strip(),
                        "mobile_number": mobile_number.strip(),
                        "receipt_no": receipt_no.strip(),
                        "qty": qty,
                        "selling_price": selling_price,
                        "total_amount": total,
                        "amount_paid": amount_paid,
                        "balance_due": balance_due,
                        "credit_term": credit_term_value,
                        "due_date": due_date_value,
                        "payment_status": status,
                        "notes": notes.strip(),
                        "created_by": user,
                    },
                ).scalar_one()

                # For credit sales, only paid money counts toward profit.
                # Store the deposit as a dated credit payment so unpaid balances stay out of profit.
                if payment_type == "Credit Sale" and amount_paid > 0:
                    conn.execute(
                        text(
                            """
                            INSERT INTO credit_payments(
                                sale_id, payment_date, amount_paid, payment_method, reference_no, notes, captured_by, created_at
                            )
                            VALUES(
                                :sale_id, :payment_date, :amount_paid, :payment_method, :reference_no, :notes, :captured_by, NOW()
                            )
                            """
                        ),
                        {
                            "sale_id": sale_id,
                            "payment_date": sale_date,
                            "amount_paid": amount_paid,
                            "payment_method": "Deposit",
                            "reference_no": receipt_no.strip(),
                            "notes": "Initial credit deposit / amount paid now",
                            "captured_by": user,
                        },
                    )

            audit("ADD_SALE", "sales", sale_id, f"{payment_type}: qty {qty}, total {total}")
            st.success(f"Sale saved. Total: {money(total)} | Balance Due: {money(balance_due)}")
            refresh_app()
        except Exception as e:
            st.error(str(e))

    st.subheader("Recent Sales")
    df = fetch_df(
        """
        SELECT s.sale_id, s.sale_date, i.item_code, i.brand, i.stock_type, i.size, s.payment_type,
               s.customer_name, s.member_name, s.mobile_number, s.receipt_no,
               s.qty, s.selling_price, s.total_amount, s.amount_paid, s.balance_due,
               s.credit_term, s.due_date, s.payment_status
        FROM sales s
        JOIN items i ON i.item_id = s.item_id
        WHERE s.is_deleted = FALSE
        ORDER BY s.created_at DESC
        LIMIT 20
        """
    )
    st.dataframe(df, use_container_width=True, hide_index=True)


def credit_reminders_page() -> None:
    st.header("Credit Reminders")
    credits = fetch_df(
        """
        SELECT s.sale_id, s.sale_date, i.item_code, i.brand, i.stock_type, i.size,
               s.member_name, s.mobile_number, s.qty, s.total_amount, s.amount_paid,
               s.balance_due, s.credit_term, s.due_date, s.payment_status
        FROM sales s
        JOIN items i ON i.item_id = s.item_id
        WHERE s.is_deleted = FALSE AND s.payment_type = 'Credit Sale'
        ORDER BY s.due_date ASC, s.created_at DESC
        """
    )

    if credits.empty:
        st.success("No credit sales recorded yet.")
        return

    credits["reminder_status"] = credits.apply(lambda r: due_status(r["due_date"], r["balance_due"]), axis=1)
    outstanding = credits[credits["balance_due"].apply(lambda x: Decimal(str(x or 0)) > 0)]

    c1, c2, c3, c4 = st.columns(4)
    c1.metric("Credit Customers", len(credits))
    c2.metric("Outstanding Accounts", len(outstanding))
    c3.metric("Outstanding Balance", money(outstanding["balance_due"].sum() if not outstanding.empty else 0))
    c4.metric("Overdue", len(outstanding[outstanding["reminder_status"] == "Overdue"]))

    st.subheader("Outstanding / Reminder List")
    st.dataframe(outstanding, use_container_width=True, hide_index=True)

    st.subheader("Record Credit Payment")
    options_df = outstanding.copy()
    if options_df.empty:
        st.success("All credit sales are fully paid.")
        return

    labels = {
        f"Sale {row['sale_id']} | {row['member_name']} | {row['mobile_number']} | Due {row['due_date']} | Balance {money(row['balance_due'])}": int(row["sale_id"])
        for _, row in options_df.iterrows()
    }
    with st.form("credit_payment_form"):
        selected = st.selectbox("Select Credit Account", list(labels.keys()))
        sale_id = labels[selected]
        c1, c2, c3 = st.columns(3)
        payment_date = c1.date_input("Payment Date", value=date.today())
        amount_paid_text = c2.text_input("Amount Paid", placeholder="Enter any amount")
        method = c3.selectbox("Payment Method", PAYMENT_METHODS)
        c1, c2 = st.columns(2)
        reference_no = c1.text_input("Reference Number")
        notes = c2.text_input("Notes")
        submitted = st.form_submit_button("Save Payment")

    if submitted:
        try:
            amount_paid = parse_decimal(amount_paid_text, "Amount Paid")
            user = st.session_state["user"]["username"]
            with get_engine().begin() as conn:
                conn.execute(
                    text(
                        """
                        INSERT INTO credit_payments(sale_id, payment_date, amount_paid, payment_method, reference_no, notes, captured_by, created_at)
                        VALUES(:sale_id, :payment_date, :amount_paid, :payment_method, :reference_no, :notes, :captured_by, NOW())
                        """
                    ),
                    {
                        "sale_id": sale_id,
                        "payment_date": payment_date,
                        "amount_paid": amount_paid,
                        "payment_method": method,
                        "reference_no": reference_no.strip(),
                        "notes": notes.strip(),
                        "captured_by": user,
                    },
                )
                conn.execute(
                    text(
                        """
                        UPDATE sales
                        SET amount_paid = amount_paid + :amount_paid,
                            balance_due = balance_due - :amount_paid
                        WHERE sale_id = :sale_id
                        """
                    ),
                    {"amount_paid": amount_paid, "sale_id": sale_id},
                )
            update_payment_status_for_sale(sale_id)
            audit("ADD_CREDIT_PAYMENT", "credit_payments", sale_id, f"Payment {amount_paid}")
            st.success("Credit payment saved.")
            refresh_app()
        except Exception as e:
            st.error(str(e))

    st.subheader("Payment History")
    payments = fetch_df(
        """
        SELECT p.payment_date, s.sale_id, s.member_name, s.mobile_number, p.amount_paid,
               p.payment_method, p.reference_no, p.notes, p.captured_by
        FROM credit_payments p
        JOIN sales s ON s.sale_id = p.sale_id
        WHERE p.is_deleted = FALSE AND s.is_deleted = FALSE
        ORDER BY p.created_at DESC
        LIMIT 100
        """
    )
    st.dataframe(payments, use_container_width=True, hide_index=True)


def expenses_page() -> None:
    st.header("Expenses")

    with st.form("expense_form"):
        c1, c2, c3 = st.columns(3)
        expense_date = c1.date_input("Expense Date", value=date.today())
        category = c2.selectbox("Category", EXPENSE_CATEGORIES)
        amount_text = c3.text_input("Amount", placeholder="Enter any amount")

        c1, c2, c3 = st.columns(3)
        method = c1.selectbox("Payment Method", PAYMENT_METHODS)
        paid_to = c2.text_input("Paid To")
        reference_no = c3.text_input("Receipt / Reference Number")
        description = st.text_area("Description / Notes")
        submitted = st.form_submit_button("Save Expense")

    if submitted:
        try:
            amount = parse_decimal(amount_text, "Expense Amount")
            exec_sql(
                """
                INSERT INTO expenses(expense_date, category, amount, payment_method, paid_to, reference_no, description, captured_by, created_at)
                VALUES(:expense_date, :category, :amount, :payment_method, :paid_to, :reference_no, :description, :captured_by, NOW())
                """,
                {
                    "expense_date": expense_date,
                    "category": category,
                    "amount": amount,
                    "payment_method": method,
                    "paid_to": paid_to.strip(),
                    "reference_no": reference_no.strip(),
                    "description": description.strip(),
                    "captured_by": st.session_state["user"]["username"],
                },
            )
            audit("ADD_EXPENSE", "expenses", None, f"Expense {amount}")
            st.success("Expense saved.")
            refresh_app()
        except Exception as e:
            st.error(str(e))

    st.subheader("Recent Expenses")
    df = fetch_df(
        """
        SELECT expense_id, expense_date, category, amount, payment_method, paid_to, reference_no, description, captured_by
        FROM expenses
        WHERE is_deleted = FALSE
        ORDER BY created_at DESC
        LIMIT 30
        """
    )
    st.dataframe(df, use_container_width=True, hide_index=True)


def corrections_page() -> None:
    st.header("Corrections / Delete")
    st.warning("Admin-only area. Use this for incorrectly entered stock, sales, expenses, or products. Records are soft-deleted and kept in audit history.")

    tab1, tab2, tab3, tab4 = st.tabs(["Delete Stock Entry", "Delete Sale", "Delete Expense", "Delete Product"])

    with tab1:
        st.subheader("Delete / Reverse Incorrect Stock Addition")
        movements = fetch_df(
            """
            SELECT m.movement_id, m.created_at, i.item_code, i.brand, i.stock_type, i.size, m.movement_type,
                   m.qty, m.unit_price, m.total_amount, m.reference_no, m.notes, m.created_by
            FROM stock_movements m
            JOIN items i ON i.item_id = m.item_id
            WHERE m.is_deleted = FALSE AND m.movement_type = 'Stock In'
            ORDER BY m.created_at DESC
            LIMIT 50
            """
        )
        st.dataframe(movements, use_container_width=True, hide_index=True)
        if not movements.empty:
            labels = {
                f"{row['movement_id']} | {row['created_at']} | {row['item_code']} | Qty {qty_fmt(row['qty'])}": int(row["movement_id"])
                for _, row in movements.iterrows()
            }
            with st.form("delete_stock_form"):
                selected = st.selectbox("Select stock entry to reverse/delete", list(labels.keys()))
                reason = st.text_input("Reason for deletion/correction")
                confirm = st.checkbox("I confirm this stock entry is incorrect and must be reversed")
                submitted = st.form_submit_button("Reverse and Delete Stock Entry")
            if submitted:
                if not confirm or not reason.strip():
                    st.error("Please confirm and enter a reason.")
                else:
                    movement_id = labels[selected]
                    mv = fetch_one("SELECT * FROM stock_movements WHERE movement_id = :id", {"id": movement_id})
                    if mv:
                        with get_engine().begin() as conn:
                            conn.execute(
                                text("UPDATE items SET current_stock = current_stock - :qty, updated_at = NOW() WHERE item_id = :item_id"),
                                {"qty": mv["qty"], "item_id": mv["item_id"]},
                            )
                            conn.execute(
                                text(
                                    """
                                    UPDATE stock_movements
                                    SET is_deleted = TRUE, deleted_at = NOW(), deleted_by = :user, delete_reason = :reason
                                    WHERE movement_id = :movement_id
                                    """
                                ),
                                {"user": st.session_state["user"]["username"], "reason": reason.strip(), "movement_id": movement_id},
                            )
                        audit("DELETE_STOCK_ENTRY", "stock_movements", movement_id, reason.strip())
                        st.success("Stock entry reversed and deleted from active records.")
                        refresh_app()

    with tab2:
        st.subheader("Delete / Reverse Incorrect Sale")
        sales = fetch_df(
            """
            SELECT s.sale_id, s.sale_date, i.item_code, i.stock_type, i.size, s.payment_type,
                   s.member_name, s.mobile_number, s.qty, s.total_amount, s.amount_paid, s.balance_due, s.payment_status
            FROM sales s
            JOIN items i ON i.item_id = s.item_id
            WHERE s.is_deleted = FALSE
            ORDER BY s.created_at DESC
            LIMIT 50
            """
        )
        st.dataframe(sales, use_container_width=True, hide_index=True)
        if not sales.empty:
            labels = {
                f"Sale {row['sale_id']} | {row['sale_date']} | {row['item_code']} | Qty {qty_fmt(row['qty'])} | {money(row['total_amount'])}": int(row["sale_id"])
                for _, row in sales.iterrows()
            }
            with st.form("delete_sale_form"):
                selected = st.selectbox("Select sale to reverse/delete", list(labels.keys()))
                reason = st.text_input("Reason for deletion/correction")
                confirm = st.checkbox("I confirm this sale is incorrect and stock must be returned")
                submitted = st.form_submit_button("Reverse and Delete Sale")
            if submitted:
                if not confirm or not reason.strip():
                    st.error("Please confirm and enter a reason.")
                else:
                    sale_id = labels[selected]
                    sale = fetch_one("SELECT * FROM sales WHERE sale_id = :id", {"id": sale_id})
                    if sale:
                        with get_engine().begin() as conn:
                            conn.execute(
                                text("UPDATE items SET current_stock = current_stock + :qty, updated_at = NOW() WHERE item_id = :item_id"),
                                {"qty": sale["qty"], "item_id": sale["item_id"]},
                            )
                            conn.execute(
                                text(
                                    """
                                    UPDATE sales SET is_deleted = TRUE, deleted_at = NOW(), deleted_by = :user, delete_reason = :reason
                                    WHERE sale_id = :sale_id
                                    """
                                ),
                                {"user": st.session_state["user"]["username"], "reason": reason.strip(), "sale_id": sale_id},
                            )
                            if sale.get("movement_id"):
                                conn.execute(
                                    text(
                                        """
                                        UPDATE stock_movements SET is_deleted = TRUE, deleted_at = NOW(), deleted_by = :user, delete_reason = :reason
                                        WHERE movement_id = :movement_id
                                        """
                                    ),
                                    {"user": st.session_state["user"]["username"], "reason": reason.strip(), "movement_id": sale["movement_id"]},
                                )
                            conn.execute(
                                text(
                                    """
                                    UPDATE credit_payments SET is_deleted = TRUE, deleted_at = NOW(), deleted_by = :user, delete_reason = :reason
                                    WHERE sale_id = :sale_id
                                    """
                                ),
                                {"user": st.session_state["user"]["username"], "reason": reason.strip(), "sale_id": sale_id},
                            )
                        audit("DELETE_SALE", "sales", sale_id, reason.strip())
                        st.success("Sale reversed and deleted from active records.")
                        refresh_app()

    with tab3:
        st.subheader("Delete Incorrect Expense")
        expenses = fetch_df(
            """
            SELECT expense_id, expense_date, category, amount, payment_method, paid_to, reference_no, description
            FROM expenses
            WHERE is_deleted = FALSE
            ORDER BY created_at DESC
            LIMIT 100
            """
        )
        st.dataframe(expenses, use_container_width=True, hide_index=True)
        if not expenses.empty:
            labels = {
                f"Expense {row['expense_id']} | {row['expense_date']} | {row['category']} | {money(row['amount'])}": int(row["expense_id"])
                for _, row in expenses.iterrows()
            }
            with st.form("delete_expense_form"):
                selected = st.selectbox("Select expense to delete", list(labels.keys()))
                reason = st.text_input("Reason for deletion/correction")
                confirm = st.checkbox("I confirm this expense is incorrect")
                submitted = st.form_submit_button("Delete Expense")
            if submitted:
                if not confirm or not reason.strip():
                    st.error("Please confirm and enter a reason.")
                else:
                    expense_id = labels[selected]
                    exec_sql(
                        """
                        UPDATE expenses SET is_deleted = TRUE, deleted_at = NOW(), deleted_by = :user, delete_reason = :reason
                        WHERE expense_id = :expense_id
                        """,
                        {"user": st.session_state["user"]["username"], "reason": reason.strip(), "expense_id": expense_id},
                    )
                    audit("DELETE_EXPENSE", "expenses", expense_id, reason.strip())
                    st.success("Expense deleted from active records.")
                    refresh_app()

    with tab4:
        st.subheader("Delete / Hide Incorrect Product")
        products = get_active_items()
        st.dataframe(display_products_df(products), use_container_width=True, hide_index=True)
        if not products.empty:
            labels = {product_label(row): int(row["item_id"]) for _, row in products.iterrows()}
            with st.form("delete_product_form"):
                selected = st.selectbox("Select product to hide/delete", list(labels.keys()))
                reason = st.text_input("Reason")
                confirm = st.checkbox("I confirm this product was entered incorrectly")
                submitted = st.form_submit_button("Hide Product")
            if submitted:
                if not confirm or not reason.strip():
                    st.error("Please confirm and enter a reason.")
                else:
                    item_id = labels[selected]
                    exec_sql(
                        "UPDATE items SET active = FALSE, updated_at = NOW() WHERE item_id = :item_id",
                        {"item_id": item_id},
                    )
                    audit("HIDE_PRODUCT", "items", item_id, reason.strip())
                    st.success("Product hidden from active product lists. Historical records remain safe.")
                    refresh_app()


def reports_page() -> None:
    st.header("Reports")

    report = st.selectbox(
        "Choose Report",
        [
            "Current Stock",
            "Sales Report",
            "Credit Sales Report",
            "Credit Payments Report",
            "Expenses Report",
            "Stock Acquisition Cost Report",
            "Stock Movements Report",
            "Audit Log",
        ],
    )

    if report == "Current Stock":
        df = fetch_df("SELECT item_id, item_code, stock_type, brand, size, colour, cost_price, selling_price, current_stock, reorder_level, active, created_at, updated_at FROM items WHERE active = TRUE ORDER BY stock_type, brand, size, colour")
    elif report == "Sales Report":
        df = fetch_df(
            """
            SELECT s.*, i.item_code, i.brand, i.stock_type, i.size, i.colour, i.cost_price,
                   (s.qty * i.cost_price) AS estimated_cost,
                   (s.total_amount - s.balance_due) AS paid_amount_for_profit,
                   s.balance_due AS unpaid_amount_excluded_from_profit,
                   ((s.total_amount - s.balance_due) - (s.qty * i.cost_price)) AS estimated_paid_basis_gross_profit
            FROM sales s JOIN items i ON i.item_id = s.item_id
            WHERE s.is_deleted = FALSE
            ORDER BY s.sale_date DESC, s.created_at DESC
            """
        )
    elif report == "Credit Sales Report":
        df = fetch_df(
            """
            SELECT s.sale_id, s.sale_date, i.item_code, i.stock_type, i.size, s.member_name, s.mobile_number,
                   s.qty, s.total_amount, s.amount_paid, s.balance_due, s.credit_term, s.due_date, s.payment_status
            FROM sales s JOIN items i ON i.item_id = s.item_id
            WHERE s.is_deleted = FALSE AND s.payment_type = 'Credit Sale'
            ORDER BY s.due_date ASC
            """
        )
        if not df.empty:
            df["reminder_status"] = df.apply(lambda r: due_status(r["due_date"], r["balance_due"]), axis=1)
    elif report == "Credit Payments Report":
        df = fetch_df(
            """
            SELECT p.*, s.member_name, s.mobile_number
            FROM credit_payments p JOIN sales s ON s.sale_id = p.sale_id
            WHERE p.is_deleted = FALSE AND s.is_deleted = FALSE
            ORDER BY p.payment_date DESC
            """
        )
    elif report == "Stock Acquisition Cost Report":
        df = fetch_df(
            """
            SELECT sm.movement_id, sm.created_at::date AS purchase_date, i.item_code, i.brand, i.stock_type, i.size, i.colour,
                   sm.qty, sm.unit_price AS acquisition_unit_cost, sm.total_amount AS acquisition_total_cost,
                   sm.reference_no, sm.notes, sm.created_by
            FROM stock_movements sm
            JOIN items i ON i.item_id = sm.item_id
            WHERE sm.is_deleted = FALSE AND sm.movement_type = 'Stock In'
            ORDER BY sm.created_at DESC
            """
        )
    elif report == "Expenses Report":
        df = fetch_df("SELECT * FROM expenses WHERE is_deleted = FALSE ORDER BY expense_date DESC")
    elif report == "Stock Movements Report":
        df = fetch_df(
            """
            SELECT m.*, i.item_code, i.brand, i.stock_type, i.size, i.colour
            FROM stock_movements m JOIN items i ON i.item_id = m.item_id
            WHERE m.is_deleted = FALSE
            ORDER BY m.created_at DESC
            """
        )
    else:
        df = fetch_df("SELECT * FROM audit_log ORDER BY created_at DESC LIMIT 500")

    st.dataframe(df, use_container_width=True, hide_index=True)

    if not df.empty:
        csv = df.to_csv(index=False).encode("utf-8")
        st.download_button("Download CSV", csv, file_name=f"{report.lower().replace(' ', '_')}.csv", mime="text/csv")

        output = BytesIO()
        with pd.ExcelWriter(output, engine="openpyxl") as writer:
            df.to_excel(writer, index=False, sheet_name=report[:31])
        st.download_button(
            "Download Excel",
            output.getvalue(),
            file_name=f"{report.lower().replace(' ', '_')}.xlsx",
            mime="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
        )


def user_management_page() -> None:
    st.header("User Management")

    tab1, tab2 = st.tabs(["Add User", "Users"])
    with tab1:
        with st.form("add_user_form"):
            c1, c2 = st.columns(2)
            full_name = c1.text_input("Full Name")
            username = c2.text_input("Username")
            c1, c2 = st.columns(2)
            role = c1.selectbox("Role", ROLES)
            password = c2.text_input("Password", type="password")
            submitted = st.form_submit_button("Add User")
        if submitted:
            try:
                if not full_name.strip() or not username.strip() or not password:
                    raise ValueError("Full name, username, and password are required.")
                exec_sql(
                    """
                    INSERT INTO users(full_name, username, password_hash, role, active, created_at)
                    VALUES(:full_name, :username, :password_hash, :role, TRUE, NOW())
                    """,
                    {
                        "full_name": full_name.strip(),
                        "username": username.strip(),
                        "password_hash": hash_password(password),
                        "role": role,
                    },
                )
                audit("ADD_USER", "users", username, f"Added user {username}")
                st.success("User added.")
                refresh_app()
            except Exception as e:
                st.error(str(e))

    with tab2:
        users = fetch_df("SELECT user_id, full_name, username, role, active, created_at FROM users ORDER BY created_at DESC")
        st.dataframe(users, use_container_width=True, hide_index=True)

        if not users.empty:
            labels = {f"{row['username']} | {row['full_name']} | {row['role']}": int(row["user_id"]) for _, row in users.iterrows()}
            with st.form("edit_user_form"):
                selected = st.selectbox("Select User", list(labels.keys()))
                c1, c2 = st.columns(2)
                new_role = c1.selectbox("New Role", ROLES)
                active = c2.checkbox("Active", value=True)
                new_password = st.text_input("New Password Optional", type="password")
                submitted = st.form_submit_button("Update User")
            if submitted:
                user_id = labels[selected]
                if new_password:
                    exec_sql(
                        "UPDATE users SET role = :role, active = :active, password_hash = :password_hash WHERE user_id = :user_id",
                        {"role": new_role, "active": active, "password_hash": hash_password(new_password), "user_id": user_id},
                    )
                else:
                    exec_sql(
                        "UPDATE users SET role = :role, active = :active WHERE user_id = :user_id",
                        {"role": new_role, "active": active, "user_id": user_id},
                    )
                audit("EDIT_USER", "users", user_id, "Updated user")
                st.success("User updated.")
                refresh_app()



# ------------------------------------------------------------
# Client ordering portal and staff order management
# ------------------------------------------------------------
def clean_mobile(mobile: str) -> str:
    return re.sub(r"\s+", "", str(mobile or "").strip())


def item_description(row: pd.Series | dict[str, Any]) -> str:
    parts = [row.get("brand"), row.get("stock_type"), row.get("size"), row.get("colour")]
    return " ".join([str(x).strip() for x in parts if x and str(x).strip()]) or str(row.get("item_name") or "Product")


@st.cache_data(ttl=90, show_spinner=False)
def get_public_products(search: str = "") -> pd.DataFrame:
    search = (search or "").strip()
    params: dict[str, Any] = {}
    where = "WHERE active = TRUE AND current_stock > 0"
    if search:
        where += """
        AND (
            item_code ILIKE :search OR stock_type ILIKE :search OR brand ILIKE :search OR
            size ILIKE :search OR colour ILIKE :search OR item_name ILIKE :search
        )
        """
        params["search"] = f"%{search}%"
    return fetch_df(
        f"""
        SELECT item_id, item_code, stock_type, item_name, brand, size, colour,
               selling_price, current_stock
        FROM items
        {where}
        ORDER BY stock_type, brand, size, colour, item_code
        LIMIT 150
        """,
        params,
    )


def get_client_cart() -> dict[int, str]:
    if "client_cart" not in st.session_state:
        st.session_state["client_cart"] = {}
    return st.session_state["client_cart"]


def add_client_cart_item(item_id: int, qty_text: str) -> None:
    qty = parse_decimal(qty_text, "quantity")
    if qty <= 0:
        raise ValueError("Quantity must be greater than zero.")
    cart = get_client_cart()
    cart[int(item_id)] = str(qty)


def remove_client_cart_item(item_id: int) -> None:
    get_client_cart().pop(int(item_id), None)


def clear_client_cart() -> None:
    st.session_state["client_cart"] = {}


def cart_dataframe() -> pd.DataFrame:
    cart = get_client_cart()
    if not cart:
        return pd.DataFrame()
    ids = tuple(int(x) for x in cart.keys())
    df = fetch_df(
        """
        SELECT item_id, item_code, stock_type, item_name, brand, size, colour,
               selling_price, current_stock
        FROM items
        WHERE item_id = ANY(:ids) AND active = TRUE
        ORDER BY stock_type, brand, size, colour
        """,
        {"ids": list(ids)},
    )
    if df.empty:
        return df
    rows = []
    for _, row in df.iterrows():
        qty = parse_decimal(cart.get(int(row["item_id"]), "0"), "quantity", allow_blank=True)
        unit_price = Decimal(str(row["selling_price"] or 0))
        rows.append(
            {
                "item_id": int(row["item_id"]),
                "Product": item_description(row),
                "Code": row["item_code"],
                "Available": row["current_stock"],
                "Qty": qty,
                "Unit Price": unit_price,
                "Line Total": qty * unit_price,
            }
        )
    return pd.DataFrame(rows)


def place_client_order(customer_name: str, mobile_number: str, order_method: str, delivery_address: str,
                       preferred_payment: str, notes: str) -> tuple[int, str, Decimal]:
    cart_df = cart_dataframe()
    if cart_df.empty:
        raise ValueError("Your order is empty.")
    customer_name = customer_name.strip()
    mobile_number = clean_mobile(mobile_number)
    if not customer_name:
        raise ValueError("Please enter your name.")
    if not mobile_number:
        raise ValueError("Please enter your mobile number.")

    total = Decimal("0")
    lines: list[str] = []
    with get_engine().begin() as conn:
        # Recheck stock/prices from the database before saving the order.
        order_id = conn.execute(
            text(
                """
                INSERT INTO client_orders(
                    order_date, customer_name, mobile_number, order_method, delivery_address,
                    preferred_payment, notes, order_total, status, created_at, updated_at
                )
                VALUES(CURRENT_DATE, :customer_name, :mobile_number, :order_method, :delivery_address,
                       :preferred_payment, :notes, 0, 'Pending', NOW(), NOW())
                RETURNING order_id
                """
            ),
            {
                "customer_name": customer_name,
                "mobile_number": mobile_number,
                "order_method": order_method,
                "delivery_address": delivery_address.strip(),
                "preferred_payment": preferred_payment,
                "notes": notes.strip(),
            },
        ).scalar_one()
        order_no = f"SH{date.today().strftime('%y%m%d')}{int(order_id):05d}"
        conn.execute(text("UPDATE client_orders SET order_no = :order_no WHERE order_id = :order_id"), {"order_no": order_no, "order_id": order_id})

        for _, row in cart_df.iterrows():
            qty = Decimal(str(row["Qty"]))
            if qty <= 0:
                continue
            item = conn.execute(
                text("SELECT * FROM items WHERE item_id = :item_id AND active = TRUE"),
                {"item_id": int(row["item_id"])},
            ).mappings().first()
            if not item:
                continue
            available = Decimal(str(item.get("current_stock") or 0))
            if qty > available:
                raise ValueError(f"Only {qty_fmt(available)} available for {item_description(item)}.")
            unit_price = Decimal(str(item.get("selling_price") or 0))
            line_total = qty * unit_price
            total += line_total
            desc = item_description(item)
            lines.append(f"- {desc} x {qty_fmt(qty)} = {money(line_total)}")
            conn.execute(
                text(
                    """
                    INSERT INTO client_order_items(
                        order_id, item_id, item_code, item_description, qty, unit_price, line_total, created_at
                    )
                    VALUES(:order_id, :item_id, :item_code, :item_description, :qty, :unit_price, :line_total, NOW())
                    """
                ),
                {
                    "order_id": order_id,
                    "item_id": int(item["item_id"]),
                    "item_code": item["item_code"],
                    "item_description": desc,
                    "qty": qty,
                    "unit_price": unit_price,
                    "line_total": line_total,
                },
            )
        conn.execute(text("UPDATE client_orders SET order_total = :total WHERE order_id = :order_id"), {"total": total, "order_id": order_id})
        conn.execute(
            text(
                """
                INSERT INTO order_status_history(order_id, old_status, new_status, notes, changed_by, created_at)
                VALUES(:order_id, NULL, 'Pending', 'Order placed by client', 'client', NOW())
                """
            ),
            {"order_id": order_id},
        )
    clear_client_cart()
    try:
        st.cache_data.clear()
    except Exception:
        pass
    return int(order_id), order_no, total


@st.cache_data(ttl=30, show_spinner=False)
def pending_orders_count() -> int:
    row = fetch_one("SELECT COUNT(*) AS cnt FROM client_orders WHERE is_deleted = FALSE AND status = 'Pending'")
    return int(row.get("cnt") or 0) if row else 0


def portal_prompt_page() -> None:
    st.markdown("# 🛏️ Welcome to SHEHaven Bedding")
    st.caption("Choose where you want to go.")
    c1, c2 = st.columns(2)
    with c1:
        st.markdown("### I am a client")
        st.write("Browse bedding and place an order online.")
        if st.button("Continue as Client", use_container_width=True, type="primary"):
            st.session_state["portal_mode"] = "client"
            st.rerun()
    with c2:
        st.markdown("### I am a user/staff")
        st.write("Log in to manage stock, sales, credit, expenses, and orders.")
        if st.button("User Login", use_container_width=True):
            st.session_state["portal_mode"] = "user"
            st.rerun()


def client_shop_page() -> None:
    st.markdown("# 🛏️ SHEHaven Bedding Orders")
    st.caption("Place an order online. Our team will confirm availability and payment details.")
    top1, top2 = st.columns([3, 1])
    with top2:
        if st.button("User/Staff Login"):
            st.session_state["portal_mode"] = "user"
            st.rerun()

    tab_shop, tab_cart, tab_track = st.tabs(["Shop", f"Cart ({len(get_client_cart())})", "Track Order"])
    with tab_shop:
        search = st.text_input("Search products", placeholder="Search by brand, size, colour, type...")
        products = get_public_products(search)
        if products.empty:
            st.info("No available products found.")
        else:
            st.caption("Client orders do not automatically reduce stock. Staff will confirm the order first.")
            for _, row in products.iterrows():
                with st.container(border=True):
                    c1, c2, c3 = st.columns([3, 1, 1])
                    c1.markdown(f"**{item_description(row)}**")
                    c1.caption(f"Code: {row['item_code']} | Available: {qty_fmt(row['current_stock'])}")
                    c2.metric("Price", money(row["selling_price"]))
                    qty_key = f"client_qty_{int(row['item_id'])}"
                    qty_text = c3.text_input("Qty", value="1", key=qty_key)
                    if c3.button("Add", key=f"client_add_{int(row['item_id'])}", use_container_width=True):
                        try:
                            add_client_cart_item(int(row["item_id"]), qty_text)
                            st.success("Added to cart.")
                        except Exception as e:
                            st.error(str(e))

    with tab_cart:
        cart_df = cart_dataframe()
        if cart_df.empty:
            st.info("Your cart is empty.")
        else:
            display = cart_df.drop(columns=["item_id"], errors="ignore").copy()
            for col in ["Unit Price", "Line Total"]:
                display[col] = display[col].apply(money)
            display["Qty"] = display["Qty"].apply(qty_fmt)
            st.dataframe(display, use_container_width=True, hide_index=True)
            total = sum(Decimal(str(x)) for x in cart_df["Line Total"].tolist())
            st.metric("Order Total", money(total))
            remove_labels = {f"{row['Product']} | {row['Code']}": int(row["item_id"]) for _, row in cart_df.iterrows()}
            if remove_labels:
                selected_remove = st.selectbox("Remove item", list(remove_labels.keys()))
                if st.button("Remove Selected Item"):
                    remove_client_cart_item(remove_labels[selected_remove])
                    st.rerun()
            with st.form("place_client_order_form"):
                customer_name = st.text_input("Your Name")
                mobile_number = st.text_input("Mobile Number")
                order_method = st.selectbox("Order Method", ORDER_METHODS)
                delivery_address = ""
                if order_method == "Delivery":
                    delivery_address = st.text_area("Delivery Address")
                preferred_payment = st.selectbox("Preferred Payment", CLIENT_PAYMENT_OPTIONS)
                notes = st.text_area("Notes Optional")
                submitted = st.form_submit_button("Place Order", type="primary")
            if submitted:
                try:
                    _, order_no, order_total = place_client_order(
                        customer_name, mobile_number, order_method, delivery_address, preferred_payment, notes
                    )
                    st.success(f"Order placed successfully. Your order number is {order_no}. Total: {money(order_total)}")
                    st.info("Your order is now pending. SHEHaven staff will see it under Client Orders.")
                except Exception as e:
                    st.error(str(e))

    with tab_track:
        st.subheader("Track your order")
        order_no = st.text_input("Order Number", key="track_order_no")
        mobile = st.text_input("Mobile Number", key="track_mobile")
        if st.button("Track Order"):
            row = fetch_one(
                """
                SELECT order_no, order_date, customer_name, mobile_number, order_total, status, notes
                FROM client_orders
                WHERE LOWER(order_no) = LOWER(:order_no)
                  AND mobile_number = :mobile
                  AND is_deleted = FALSE
                """,
                {"order_no": order_no.strip(), "mobile": clean_mobile(mobile)},
            )
            if not row:
                st.error("Order not found. Check your order number and mobile number.")
            else:
                st.success(f"Status: {row['status']}")
                st.write(f"Order No: {row['order_no']}")
                st.write(f"Order Total: {money(row['order_total'])}")
                items = fetch_df(
                    """
                    SELECT item_description, qty, unit_price, line_total
                    FROM client_order_items
                    WHERE order_id = (SELECT order_id FROM client_orders WHERE order_no = :order_no LIMIT 1)
                    ORDER BY order_item_id
                    """,
                    {"order_no": row["order_no"]},
                )
                if not items.empty:
                    items["unit_price"] = items["unit_price"].apply(money)
                    items["line_total"] = items["line_total"].apply(money)
                    items["qty"] = items["qty"].apply(qty_fmt)
                    st.dataframe(items, use_container_width=True, hide_index=True)


def client_orders_page() -> None:
    st.subheader("Client Orders")
    pending = pending_orders_count()
    st.metric("Pending Orders", pending)
    status_filter = st.selectbox("Filter by Status", ["All"] + ORDER_STATUSES)
    where = "WHERE o.is_deleted = FALSE"
    params: dict[str, Any] = {}
    if status_filter != "All":
        where += " AND o.status = :status"
        params["status"] = status_filter
    orders = fetch_df(
        f"""
        SELECT o.order_id, o.order_no, o.order_date, o.customer_name, o.mobile_number,
               o.order_method, o.preferred_payment, o.order_total, o.status, o.created_at
        FROM client_orders o
        {where}
        ORDER BY CASE WHEN o.status = 'Pending' THEN 0 ELSE 1 END, o.created_at DESC
        LIMIT 200
        """,
        params,
    )
    if orders.empty:
        st.info("No client orders found.")
        return
    view = orders.copy()
    view["order_total"] = view["order_total"].apply(money)
    st.dataframe(view.drop(columns=["order_id"], errors="ignore"), use_container_width=True, hide_index=True)

    labels = {f"{row['order_no']} | {row['customer_name']} | {row['mobile_number']} | {row['status']}": int(row["order_id"]) for _, row in orders.iterrows()}
    selected = st.selectbox("Select Order", list(labels.keys()))
    order_id = labels[selected]
    order = fetch_one("SELECT * FROM client_orders WHERE order_id = :order_id", {"order_id": order_id})
    items = fetch_df(
        """
        SELECT item_description, item_code, qty, unit_price, line_total
        FROM client_order_items
        WHERE order_id = :order_id
        ORDER BY order_item_id
        """,
        {"order_id": order_id},
    )
    if order:
        st.markdown(f"### Order {order['order_no']} | {order['status']}")
        c1, c2, c3 = st.columns(3)
        c1.write(f"**Client:** {order['customer_name']}")
        c2.write(f"**Mobile:** {order['mobile_number']}")
        c3.write(f"**Total:** {money(order['order_total'])}")
        if order.get("delivery_address"):
            st.write(f"**Delivery Address:** {order['delivery_address']}")
        if order.get("notes"):
            st.write(f"**Notes:** {order['notes']}")
        if not items.empty:
            show_items = items.copy()
            show_items["qty"] = show_items["qty"].apply(qty_fmt)
            show_items["unit_price"] = show_items["unit_price"].apply(money)
            show_items["line_total"] = show_items["line_total"].apply(money)
            st.dataframe(show_items, use_container_width=True, hide_index=True)

        st.markdown("#### Update Order")
        with st.form("update_client_order_status_form"):
            new_status = st.selectbox("New Status", ORDER_STATUSES, index=ORDER_STATUSES.index(order["status"]) if order["status"] in ORDER_STATUSES else 0)
            notes = st.text_area("Status Notes Optional")
            submitted = st.form_submit_button("Update Status")
        if submitted:
            user = st.session_state.get("user", {})
            exec_sql(
                "UPDATE client_orders SET status = :status, updated_at = NOW() WHERE order_id = :order_id",
                {"status": new_status, "order_id": order_id},
            )
            exec_sql(
                """
                INSERT INTO order_status_history(order_id, old_status, new_status, notes, changed_by, created_at)
                VALUES(:order_id, :old_status, :new_status, :notes, :changed_by, NOW())
                """,
                {
                    "order_id": order_id,
                    "old_status": order["status"],
                    "new_status": new_status,
                    "notes": notes,
                    "changed_by": user.get("username", "user"),
                },
            )
            audit("UPDATE_ORDER_STATUS", "client_orders", order_id, f"{order['status']} to {new_status}")
            st.success("Order status updated.")
            refresh_app()

        with st.expander("Delete / Cancel Wrong Order"):
            reason = st.text_area("Delete reason")
            if st.button("Delete Selected Order", type="secondary"):
                if not reason.strip():
                    st.error("Please enter a reason.")
                else:
                    user = st.session_state.get("user", {})
                    exec_sql(
                        """
                        UPDATE client_orders
                        SET is_deleted = TRUE, deleted_at = NOW(), deleted_by = :deleted_by, delete_reason = :reason
                        WHERE order_id = :order_id
                        """,
                        {"deleted_by": user.get("username", "user"), "reason": reason, "order_id": order_id},
                    )
                    audit("DELETE_CLIENT_ORDER", "client_orders", order_id, reason)
                    st.success("Order deleted/corrected.")
                    refresh_app()


# ------------------------------------------------------------
# Main app
# ------------------------------------------------------------
def main() -> None:
    st.set_page_config(page_title=APP_SHORT, page_icon="🛏️", layout="wide")

    try:
        init_db()
    except Exception as e:
        st.title("SHEHaven Bedding")
        st.error("Database connection failed.")
        st.code(str(e))
        st.markdown(
            """
            Check that your Streamlit Cloud secret is set as:

            ```toml
            SUPABASE_DB_URL = "your_supabase_connection_string"
            ```
            """
        )
        return

    if "user" not in st.session_state:
        mode = st.session_state.get("portal_mode")
        if not mode:
            portal_prompt_page()
        elif mode == "client":
            client_shop_page()
        else:
            login_page()
        return

    user = st.session_state["user"]
    st.sidebar.title("🛏️ SHEHaven")
    st.sidebar.caption(f"Logged in: {user['full_name']} ({user['role']})")
    try:
        pcount = pending_orders_count()
        if pcount:
            st.sidebar.warning(f"{pcount} pending client order(s)")
    except Exception:
        pass

    pages = [
        "Dashboard",
        "Products",
        "Add Stock",
        "Sales",
        "Credit Reminders",
        "Client Orders",
        "Expenses",
        "Corrections / Delete",
        "Reports",
        "User Management",
    ]
    visible_pages = [p for p in pages if can_access(p)]
    page = st.sidebar.radio("Menu", visible_pages)

    if st.sidebar.button("Client Order Page"):
        st.session_state.clear()
        st.session_state["portal_mode"] = "client"
        st.rerun()

    if st.sidebar.button("Logout"):
        st.session_state.clear()
        refresh_app()

    st.title(APP_NAME)

    if page == "Dashboard":
        dashboard_page()
    elif page == "Products":
        products_page()
    elif page == "Add Stock":
        add_stock_page()
    elif page == "Sales":
        sales_page()
    elif page == "Credit Reminders":
        credit_reminders_page()
    elif page == "Client Orders":
        client_orders_page()
    elif page == "Expenses":
        expenses_page()
    elif page == "Corrections / Delete":
        corrections_page()
    elif page == "Reports":
        reports_page()
    elif page == "User Management":
        user_management_page()


if __name__ == "__main__":
    main()
