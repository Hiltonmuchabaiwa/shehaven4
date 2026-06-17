"""
SHEHaven Bedding Client Order Website
Streamlit + Supabase/PostgreSQL

Purpose:
- Allows clients to browse available products and place orders online.
- Saves orders in Supabase.
- Includes a staff/admin order management area.
- Does NOT automatically reduce stock when a client places an order.
  Admin should confirm the order first, then record the actual sale in the main stock system.

Required Streamlit secret:
SUPABASE_DB_URL = "postgresql://..." or "postgresql+psycopg://..."
"""

from __future__ import annotations

import hashlib
import os
import re
from datetime import date, datetime
from decimal import Decimal, InvalidOperation
from typing import Any, Optional
from urllib.parse import urlparse, urlunparse

import pandas as pd
import streamlit as st
from sqlalchemy import create_engine, text
from sqlalchemy.engine import Engine


APP_NAME = "SHEHaven Bedding Orders"
APP_TAGLINE = "Order bedding online. SHEHaven will confirm availability and payment details."

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


# -----------------------------
# Page setup and styling
# -----------------------------
st.set_page_config(page_title=APP_NAME, page_icon="🛏️", layout="wide", initial_sidebar_state="collapsed")

st.markdown(
    """
    <style>
        .block-container {padding-top: 1.2rem; padding-bottom: 2rem; max-width: 1180px;}
        div[data-testid="stMetric"] {background: #ffffff; border: 1px solid #eee; border-radius: 16px; padding: 12px; box-shadow: 0 2px 8px rgba(0,0,0,0.035);}
        .she-card {background: #ffffff; border: 1px solid #eee; border-radius: 18px; padding: 16px; margin-bottom: 14px; box-shadow: 0 2px 10px rgba(0,0,0,0.04);}
        .muted {color: #6b7280; font-size: 0.92rem;}
        .price {font-size: 1.2rem; font-weight: 700;}
        .pill {display: inline-block; padding: 3px 10px; border-radius: 999px; background: #f3f4f6; margin: 2px; font-size: 0.84rem;}
        .hero {background: linear-gradient(135deg, #111827 0%, #374151 100%); color: white; padding: 22px 24px; border-radius: 22px; margin-bottom: 18px;}
        .hero h1 {margin-bottom: 6px;}
        .success-box {background: #ecfdf5; border: 1px solid #10b981; color: #064e3b; border-radius: 14px; padding: 14px;}
        .warn-box {background: #fffbeb; border: 1px solid #f59e0b; color: #78350f; border-radius: 14px; padding: 14px;}
        @media (max-width: 760px) {
            .block-container {padding-left: 0.75rem; padding-right: 0.75rem;}
            .hero {padding: 16px; border-radius: 16px;}
            .hero h1 {font-size: 1.5rem;}
        }
    </style>
    """,
    unsafe_allow_html=True,
)


# -----------------------------
# Database helpers
# -----------------------------
def clean_db_url(raw_url: str) -> str:
    if not raw_url:
        return raw_url
    url = raw_url.strip().strip('"').strip("'")
    if url.startswith("postgres://"):
        url = "postgresql://" + url[len("postgres://") :]
    if url.startswith("postgresql+psycopg2://"):
        url = "postgresql+psycopg://" + url[len("postgresql+psycopg2://") :]
    elif url.startswith("postgresql://"):
        url = "postgresql+psycopg://" + url[len("postgresql://") :]

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
        raise RuntimeError("Missing SUPABASE_DB_URL in Streamlit Secrets.")
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


def hash_password(password: str) -> str:
    return hashlib.sha256(password.encode("utf-8")).hexdigest()


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

    CREATE INDEX IF NOT EXISTS idx_client_orders_status ON client_orders(status);
    CREATE INDEX IF NOT EXISTS idx_client_orders_mobile ON client_orders(mobile_number);
    CREATE INDEX IF NOT EXISTS idx_client_orders_deleted ON client_orders(is_deleted);
    CREATE INDEX IF NOT EXISTS idx_client_order_items_order ON client_order_items(order_id);
    CREATE INDEX IF NOT EXISTS idx_items_active_stock ON items(active, current_stock);
    """
    with get_engine().begin() as conn:
        for statement in schema.split(";"):
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
                {"full_name": "System Administrator", "username": "admin", "password_hash": hash_password("admin123")},
            )


# -----------------------------
# Formatting and parsing
# -----------------------------
def parse_decimal(value: Any, field_name: str = "value", allow_blank: bool = False) -> Decimal:
    raw = str(value or "").replace(",", "").strip()
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


def item_label(row: pd.Series | dict[str, Any]) -> str:
    parts = [
        str(row.get("brand") or "").strip(),
        str(row.get("stock_type") or "").strip(),
        str(row.get("size") or "").strip(),
        str(row.get("colour") or "").strip(),
    ]
    label = " ".join([p for p in parts if p and p.lower() != "none"])
    return label or str(row.get("item_name") or row.get("item_code") or "Product")


def clean_mobile(mobile: str) -> str:
    return re.sub(r"\s+", "", mobile or "").strip()


def clear_cached_data() -> None:
    try:
        st.cache_data.clear()
    except Exception:
        pass


# -----------------------------
# Cached reads
# -----------------------------
@st.cache_data(ttl=90, show_spinner=False)
def get_public_products(search: str = "") -> pd.DataFrame:
    search = f"%{search.strip().lower()}%"
    return fetch_df(
        """
        SELECT item_id, item_code, item_name, stock_type, brand, size, colour,
               selling_price, current_stock
        FROM items
        WHERE active = TRUE
          AND current_stock > 0
          AND (
                :search = '%%'
                OR LOWER(COALESCE(item_code,'')) LIKE :search
                OR LOWER(COALESCE(item_name,'')) LIKE :search
                OR LOWER(COALESCE(stock_type,'')) LIKE :search
                OR LOWER(COALESCE(brand,'')) LIKE :search
                OR LOWER(COALESCE(size,'')) LIKE :search
                OR LOWER(COALESCE(colour,'')) LIKE :search
          )
        ORDER BY brand, stock_type, size, colour, item_id
        LIMIT 250
        """,
        {"search": search},
    )


@st.cache_data(ttl=45, show_spinner=False)
def get_products_by_ids(ids: tuple[int, ...]) -> pd.DataFrame:
    if not ids:
        return pd.DataFrame()
    placeholders = ",".join([f":id{i}" for i, _ in enumerate(ids)])
    params = {f"id{i}": int(item_id) for i, item_id in enumerate(ids)}
    return fetch_df(
        f"""
        SELECT item_id, item_code, item_name, stock_type, brand, size, colour,
               selling_price, current_stock
        FROM items
        WHERE item_id IN ({placeholders}) AND active = TRUE
        """,
        params,
    )


@st.cache_data(ttl=30, show_spinner=False)
def order_stats() -> dict[str, Any]:
    row = fetch_one(
        """
        SELECT
            COUNT(*) FILTER (WHERE is_deleted = FALSE) AS total_orders,
            COUNT(*) FILTER (WHERE is_deleted = FALSE AND status = 'Pending') AS pending_orders,
            COUNT(*) FILTER (WHERE is_deleted = FALSE AND order_date = CURRENT_DATE) AS orders_today,
            COALESCE(SUM(order_total) FILTER (WHERE is_deleted = FALSE), 0) AS total_order_value
        FROM client_orders
        """
    )
    return row or {}


# -----------------------------
# Cart helpers
# -----------------------------
def get_cart() -> dict[int, str]:
    if "cart" not in st.session_state:
        st.session_state.cart = {}
    return st.session_state.cart


def add_to_cart(item_id: int, qty_text: str) -> None:
    qty = parse_decimal(qty_text, "quantity")
    if qty <= 0:
        raise ValueError("Quantity must be greater than zero.")
    cart = get_cart()
    existing = parse_decimal(cart.get(item_id, "0"), "quantity", allow_blank=True)
    cart[item_id] = str(existing + qty)


def remove_from_cart(item_id: int) -> None:
    cart = get_cart()
    if item_id in cart:
        del cart[item_id]


def empty_cart() -> None:
    st.session_state.cart = {}


def cart_dataframe() -> pd.DataFrame:
    cart = get_cart()
    ids = tuple(cart.keys())
    products = get_products_by_ids(ids)
    if products.empty:
        return pd.DataFrame()
    rows = []
    for _, p in products.iterrows():
        qty = parse_decimal(cart.get(int(p["item_id"]), "0"), "quantity", allow_blank=True)
        price = Decimal(str(p.get("selling_price") or 0))
        line_total = qty * price
        rows.append(
            {
                "Item ID": int(p["item_id"]),
                "Product": item_label(p),
                "Code": p.get("item_code"),
                "Available": p.get("current_stock"),
                "Qty Ordered": qty,
                "Unit Price": price,
                "Line Total": line_total,
            }
        )
    return pd.DataFrame(rows)


# -----------------------------
# Write operations
# -----------------------------
def place_order(customer_name: str, mobile_number: str, order_method: str, delivery_address: str,
                preferred_payment: str, notes: str) -> str:
    customer_name = customer_name.strip()
    mobile_number = clean_mobile(mobile_number)
    if not customer_name:
        raise ValueError("Please enter your full name.")
    if not mobile_number:
        raise ValueError("Please enter your mobile number.")

    cart_df = cart_dataframe()
    if cart_df.empty:
        raise ValueError("Your order is empty. Please add at least one product.")

    # Recheck available stock at order time. This does not reduce stock; it only prevents impossible orders.
    for _, row in cart_df.iterrows():
        qty = Decimal(str(row["Qty Ordered"]))
        available = Decimal(str(row["Available"] or 0))
        if qty <= 0:
            raise ValueError("Quantity must be greater than zero.")
        if qty > available:
            raise ValueError(f"Only {qty_fmt(available)} available for {row['Product']}.")

    order_total = sum(Decimal(str(x)) for x in cart_df["Line Total"].tolist())

    with get_engine().begin() as conn:
        order_id = conn.execute(
            text(
                """
                INSERT INTO client_orders(
                    order_no, order_date, customer_name, mobile_number, order_method,
                    delivery_address, preferred_payment, notes, order_total, status,
                    created_at, updated_at, is_deleted
                )
                VALUES(
                    'TEMP', CURRENT_DATE, :customer_name, :mobile_number, :order_method,
                    :delivery_address, :preferred_payment, :notes, :order_total, 'Pending',
                    NOW(), NOW(), FALSE
                )
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
                "order_total": order_total,
            },
        ).scalar()

        order_no = f"SHO-{datetime.now():%Y%m%d}-{int(order_id):04d}"
        conn.execute(text("UPDATE client_orders SET order_no = :order_no WHERE order_id = :order_id"), {"order_no": order_no, "order_id": order_id})

        for _, row in cart_df.iterrows():
            conn.execute(
                text(
                    """
                    INSERT INTO client_order_items(
                        order_id, item_id, item_code, item_description, qty,
                        unit_price, line_total, created_at
                    )
                    VALUES(:order_id, :item_id, :item_code, :item_description, :qty,
                           :unit_price, :line_total, NOW())
                    """
                ),
                {
                    "order_id": order_id,
                    "item_id": int(row["Item ID"]),
                    "item_code": row["Code"],
                    "item_description": row["Product"],
                    "qty": Decimal(str(row["Qty Ordered"])),
                    "unit_price": Decimal(str(row["Unit Price"])),
                    "line_total": Decimal(str(row["Line Total"])),
                },
            )

        conn.execute(
            text(
                """
                INSERT INTO order_status_history(order_id, old_status, new_status, notes, changed_by, created_at)
                VALUES(:order_id, NULL, 'Pending', 'Client placed order online', :changed_by, NOW())
                """
            ),
            {"order_id": order_id, "changed_by": customer_name},
        )

    clear_cached_data()
    empty_cart()
    return order_no


def update_order_status(order_id: int, new_status: str, notes: str, changed_by: str) -> None:
    old = fetch_one("SELECT status FROM client_orders WHERE order_id = :order_id", {"order_id": order_id})
    old_status = old["status"] if old else None
    with get_engine().begin() as conn:
        conn.execute(
            text("UPDATE client_orders SET status = :status, updated_at = NOW() WHERE order_id = :order_id"),
            {"status": new_status, "order_id": order_id},
        )
        conn.execute(
            text(
                """
                INSERT INTO order_status_history(order_id, old_status, new_status, notes, changed_by, created_at)
                VALUES(:order_id, :old_status, :new_status, :notes, :changed_by, NOW())
                """
            ),
            {"order_id": order_id, "old_status": old_status, "new_status": new_status, "notes": notes, "changed_by": changed_by},
        )
    clear_cached_data()


def delete_order(order_id: int, reason: str, deleted_by: str) -> None:
    if not reason.strip():
        raise ValueError("Please enter a deletion reason.")
    with get_engine().begin() as conn:
        conn.execute(
            text(
                """
                UPDATE client_orders
                SET is_deleted = TRUE, deleted_at = NOW(), deleted_by = :deleted_by,
                    delete_reason = :reason, updated_at = NOW()
                WHERE order_id = :order_id
                """
            ),
            {"order_id": order_id, "deleted_by": deleted_by, "reason": reason.strip()},
        )
        conn.execute(
            text(
                """
                INSERT INTO order_status_history(order_id, old_status, new_status, notes, changed_by, created_at)
                VALUES(:order_id, NULL, 'Deleted', :notes, :changed_by, NOW())
                """
            ),
            {"order_id": order_id, "notes": reason.strip(), "changed_by": deleted_by},
        )
    clear_cached_data()


# -----------------------------
# Authentication
# -----------------------------
def login_staff(username: str, password: str) -> bool:
    row = fetch_one(
        """
        SELECT user_id, full_name, username, role
        FROM users
        WHERE username = :username
          AND password_hash = :password_hash
          AND active = TRUE
        """,
        {"username": username.strip(), "password_hash": hash_password(password)},
    )
    if row:
        st.session_state.staff_user = row
        return True
    return False


def staff_user() -> Optional[dict[str, Any]]:
    return st.session_state.get("staff_user")


def require_staff_login() -> bool:
    user = staff_user()
    if user:
        c1, c2 = st.columns([3, 1])
        with c1:
            st.caption(f"Logged in as {user['full_name']} ({user['role']})")
        with c2:
            if st.button("Log out"):
                st.session_state.pop("staff_user", None)
                st.rerun()
        return True

    st.subheader("Staff Login")
    with st.form("staff_login"):
        username = st.text_input("Username")
        password = st.text_input("Password", type="password")
        submitted = st.form_submit_button("Login")
    if submitted:
        if login_staff(username, password):
            st.success("Logged in successfully.")
            st.rerun()
        else:
            st.error("Invalid username or password.")
    return False


# -----------------------------
# Client pages
# -----------------------------
def render_header() -> None:
    st.markdown(
        f"""
        <div class="hero">
            <h1>🛏️ {APP_NAME}</h1>
            <div>{APP_TAGLINE}</div>
        </div>
        """,
        unsafe_allow_html=True,
    )


def shop_page() -> None:
    render_header()

    st.info("Place your order here. SHEHaven will contact you to confirm availability, delivery/collection and payment.")

    top1, top2 = st.columns([2, 1])
    with top1:
        search = st.text_input("Search products", placeholder="Example: Queen blanket, Mooi mooi, blue")
    with top2:
        st.write("")
        if st.button("Refresh products"):
            clear_cached_data()
            st.rerun()

    products = get_public_products(search)
    if products.empty:
        st.warning("No available products found. Try a different search.")
    else:
        st.subheader("Available Products")
        for _, p in products.iterrows():
            with st.container():
                st.markdown('<div class="she-card">', unsafe_allow_html=True)
                c1, c2, c3 = st.columns([3, 1, 1])
                product_name = item_label(p)
                with c1:
                    st.markdown(f"### {product_name}")
                    st.markdown(
                        f"<span class='pill'>{p.get('stock_type') or 'Product'}</span> "
                        f"<span class='pill'>{p.get('size') or 'Size not set'}</span> "
                        f"<span class='pill'>{p.get('colour') or 'Colour not set'}</span>",
                        unsafe_allow_html=True,
                    )
                    st.caption(f"Code: {p.get('item_code')}")
                with c2:
                    st.markdown(f"<div class='price'>{money(p.get('selling_price'))}</div>", unsafe_allow_html=True)
                    st.caption(f"Available: {qty_fmt(p.get('current_stock'))}")
                with c3:
                    with st.form(f"add_{int(p['item_id'])}"):
                        qty_text = st.text_input("Qty", value="1", key=f"qty_{int(p['item_id'])}")
                        add = st.form_submit_button("Add to order")
                    if add:
                        try:
                            add_to_cart(int(p["item_id"]), qty_text)
                            st.success("Added.")
                            st.rerun()
                        except Exception as e:
                            st.error(str(e))
                st.markdown("</div>", unsafe_allow_html=True)

    st.divider()
    cart_section()


def cart_section() -> None:
    st.subheader("Your Order")
    cart_df = cart_dataframe()
    if cart_df.empty:
        st.caption("No products added yet.")
        return

    display = cart_df.copy()
    display["Available"] = display["Available"].apply(qty_fmt)
    display["Qty Ordered"] = display["Qty Ordered"].apply(qty_fmt)
    display["Unit Price"] = display["Unit Price"].apply(money)
    display["Line Total"] = display["Line Total"].apply(money)
    st.dataframe(display[["Product", "Available", "Qty Ordered", "Unit Price", "Line Total"]], use_container_width=True, hide_index=True)

    total = sum(Decimal(str(x)) for x in cart_df["Line Total"].tolist())
    st.metric("Order Total", money(total))

    rem_cols = st.columns([2, 1])
    with rem_cols[0]:
        labels = {f"{row['Product']} | Qty {qty_fmt(row['Qty Ordered'])}": int(row["Item ID"]) for _, row in cart_df.iterrows()}
        selected = st.selectbox("Remove item from order", list(labels.keys()))
    with rem_cols[1]:
        st.write("")
        if st.button("Remove selected"):
            remove_from_cart(labels[selected])
            st.rerun()

    if st.button("Clear whole order"):
        empty_cart()
        st.rerun()

    st.divider()
    st.subheader("Customer Details")
    with st.form("place_order_form"):
        customer_name = st.text_input("Full Name")
        mobile_number = st.text_input("Mobile Number")
        order_method = st.selectbox("Order Method", ORDER_METHODS)
        delivery_address = ""
        if order_method == "Delivery":
            delivery_address = st.text_area("Delivery Address")
        preferred_payment = st.selectbox("Preferred Payment", CLIENT_PAYMENT_OPTIONS)
        notes = st.text_area("Notes / Special request", placeholder="Example: Please call before delivery")
        submitted = st.form_submit_button("Place Order")
    if submitted:
        try:
            order_no = place_order(customer_name, mobile_number, order_method, delivery_address, preferred_payment, notes)
            st.markdown(
                f"""
                <div class="success-box">
                    <b>Order placed successfully.</b><br>
                    Your order number is <b>{order_no}</b>.<br>
                    SHEHaven will contact you to confirm your order.
                </div>
                """,
                unsafe_allow_html=True,
            )
        except Exception as e:
            st.error(str(e))


def track_order_page() -> None:
    render_header()
    st.subheader("Track Your Order")
    st.caption("Enter your order number and mobile number.")
    with st.form("track_order"):
        order_no = st.text_input("Order Number", placeholder="Example: SHO-20260617-0001")
        mobile = st.text_input("Mobile Number")
        submitted = st.form_submit_button("Check Order")
    if submitted:
        row = fetch_one(
            """
            SELECT order_id, order_no, order_date, customer_name, mobile_number,
                   order_method, preferred_payment, order_total, status, notes
            FROM client_orders
            WHERE LOWER(order_no) = LOWER(:order_no)
              AND mobile_number = :mobile
              AND is_deleted = FALSE
            """,
            {"order_no": order_no.strip(), "mobile": clean_mobile(mobile)},
        )
        if not row:
            st.warning("Order not found. Check the order number and mobile number.")
            return
        st.success(f"Order found: {row['order_no']}")
        m1, m2, m3 = st.columns(3)
        m1.metric("Status", row["status"])
        m2.metric("Order Total", money(row["order_total"]))
        m3.metric("Method", row["order_method"])
        items = fetch_df(
            """
            SELECT item_description AS product, qty, unit_price, line_total
            FROM client_order_items
            WHERE order_id = :order_id
            ORDER BY order_item_id
            """,
            {"order_id": row["order_id"]},
        )
        if not items.empty:
            d = items.copy()
            d["qty"] = d["qty"].apply(qty_fmt)
            d["unit_price"] = d["unit_price"].apply(money)
            d["line_total"] = d["line_total"].apply(money)
            st.dataframe(d, use_container_width=True, hide_index=True)

        history = fetch_df(
            """
            SELECT new_status AS status, notes, changed_by, created_at
            FROM order_status_history
            WHERE order_id = :order_id
            ORDER BY created_at DESC
            """,
            {"order_id": row["order_id"]},
        )
        if not history.empty:
            st.subheader("Order Updates")
            st.dataframe(history, use_container_width=True, hide_index=True)


# -----------------------------
# Staff pages
# -----------------------------
def staff_orders_page() -> None:
    render_header()
    if not require_staff_login():
        return

    user = staff_user() or {}
    st.subheader("Order Management")

    stats = order_stats()
    c1, c2, c3, c4 = st.columns(4)
    c1.metric("Total Orders", int(stats.get("total_orders") or 0))
    c2.metric("Pending", int(stats.get("pending_orders") or 0))
    c3.metric("Orders Today", int(stats.get("orders_today") or 0))
    c4.metric("Order Value", money(stats.get("total_order_value")))

    filters = st.columns([1, 1, 2])
    with filters[0]:
        status_filter = st.selectbox("Status", ["All"] + ORDER_STATUSES)
    with filters[1]:
        limit = st.selectbox("Rows", [50, 100, 200, 500], index=1)
    with filters[2]:
        search = st.text_input("Search order/customer/mobile")

    where = ["is_deleted = FALSE"]
    params: dict[str, Any] = {"limit": int(limit)}
    if status_filter != "All":
        where.append("status = :status")
        params["status"] = status_filter
    if search.strip():
        where.append("(LOWER(order_no) LIKE :search OR LOWER(customer_name) LIKE :search OR mobile_number LIKE :search)")
        params["search"] = f"%{search.lower().strip()}%"
    where_sql = " AND ".join(where)

    orders = fetch_df(
        f"""
        SELECT order_id, order_no, order_date, customer_name, mobile_number,
               order_method, preferred_payment, order_total, status, created_at
        FROM client_orders
        WHERE {where_sql}
        ORDER BY created_at DESC
        LIMIT :limit
        """,
        params,
    )

    if orders.empty:
        st.info("No orders found.")
        return

    display = orders.copy()
    display["order_total"] = display["order_total"].apply(money)
    st.dataframe(
        display[["order_no", "order_date", "customer_name", "mobile_number", "order_method", "preferred_payment", "order_total", "status"]],
        use_container_width=True,
        hide_index=True,
    )

    order_options = {f"{r.order_no} | {r.customer_name} | {money(r.order_total)} | {r.status}": int(r.order_id) for r in orders.itertuples()}
    selected_label = st.selectbox("Select order to manage", list(order_options.keys()))
    order_id = order_options[selected_label]

    order = fetch_one(
        """
        SELECT * FROM client_orders
        WHERE order_id = :order_id AND is_deleted = FALSE
        """,
        {"order_id": order_id},
    )
    if not order:
        st.warning("Order no longer exists.")
        return

    st.markdown("### Selected Order")
    s1, s2, s3 = st.columns(3)
    s1.metric("Order No", order["order_no"])
    s2.metric("Status", order["status"])
    s3.metric("Total", money(order["order_total"]))
    st.write(f"**Customer:** {order['customer_name']} | **Mobile:** {order['mobile_number']}")
    st.write(f"**Method:** {order['order_method']} | **Payment:** {order.get('preferred_payment') or ''}")
    if order.get("delivery_address"):
        st.write(f"**Delivery address:** {order['delivery_address']}")
    if order.get("notes"):
        st.write(f"**Notes:** {order['notes']}")

    items = fetch_df(
        """
        SELECT item_description AS product, item_code, qty, unit_price, line_total
        FROM client_order_items
        WHERE order_id = :order_id
        ORDER BY order_item_id
        """,
        {"order_id": order_id},
    )
    if not items.empty:
        d = items.copy()
        d["qty"] = d["qty"].apply(qty_fmt)
        d["unit_price"] = d["unit_price"].apply(money)
        d["line_total"] = d["line_total"].apply(money)
        st.dataframe(d, use_container_width=True, hide_index=True)

    tab1, tab2, tab3 = st.tabs(["Update Status", "Order History", "Delete / Correction"])

    with tab1:
        with st.form("status_update_form"):
            new_status = st.selectbox("New Status", ORDER_STATUSES, index=ORDER_STATUSES.index(order["status"]) if order["status"] in ORDER_STATUSES else 0)
            notes = st.text_area("Status notes", placeholder="Example: Customer contacted and confirmed order")
            submit = st.form_submit_button("Save Status")
        if submit:
            try:
                update_order_status(order_id, new_status, notes, user.get("username", "staff"))
                st.success("Order status updated.")
                st.rerun()
            except Exception as e:
                st.error(str(e))

        st.markdown(
            """
            <div class="warn-box">
            <b>Stock control note:</b> Client orders do not reduce stock automatically. After confirming the order, record the actual sale in the main SHEHaven stock system. This protects stock from fake or abandoned orders.
            </div>
            """,
            unsafe_allow_html=True,
        )

    with tab2:
        hist = fetch_df(
            """
            SELECT old_status, new_status, notes, changed_by, created_at
            FROM order_status_history
            WHERE order_id = :order_id
            ORDER BY created_at DESC
            """,
            {"order_id": order_id},
        )
        st.dataframe(hist, use_container_width=True, hide_index=True)

    with tab3:
        st.warning("Only delete orders that were entered incorrectly, duplicated, or are fake/spam.")
        with st.form("delete_order_form"):
            reason = st.text_area("Reason for deleting this order")
            confirm = st.checkbox("I confirm this order should be deleted/cancelled from active orders")
            delete_btn = st.form_submit_button("Delete Order")
        if delete_btn:
            if not confirm:
                st.error("Please tick the confirmation box.")
            else:
                try:
                    delete_order(order_id, reason, user.get("username", "staff"))
                    st.success("Order deleted from active orders.")
                    st.rerun()
                except Exception as e:
                    st.error(str(e))


# -----------------------------
# Main
# -----------------------------
def main() -> None:
    try:
        init_db()
    except Exception as e:
        st.error("Database connection failed.")
        st.exception(e)
        st.stop()

    page = st.sidebar.radio("Menu", ["Shop", "Track Order", "Staff Orders"])
    if page == "Shop":
        shop_page()
    elif page == "Track Order":
        track_order_page()
    else:
        staff_orders_page()

    st.caption("© SHEHaven Bedding")


if __name__ == "__main__":
    main()
