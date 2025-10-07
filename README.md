# =============================================================================
# Bill Bharat POS - Streamlit + Firebase Integration
# =============================================================================
import os
import time
import json
import warnings
import requests
import streamlit as st
from firebase_admin import credentials, firestore, auth
import firebase_admin
from pyngrok import ngrok

warnings.filterwarnings("ignore")

# =============================================================================
# CONFIGURATION
# =============================================================================
CREDENTIALS_FILE = "billbharatfirebase.json"  # Firebase credentials JSON
CURRENCY = "₹"
PORT = 8501

# -----------------------------------------------------------------------------
# Initialize Firebase
# -----------------------------------------------------------------------------
if not firebase_admin._apps:
    cred = credentials.Certificate(CREDENTIALS_FILE)
    firebase_admin.initialize_app(cred)

DB = firestore.client()
AUTH = auth

# -----------------------------------------------------------------------------
# Streamlit Page Config
# -----------------------------------------------------------------------------
st.set_page_config(page_title="Bill Bharat POS", layout="wide")
st.title("🧾 Bill Bharat POS")

# -----------------------------------------------------------------------------
# Session State Defaults
# -----------------------------------------------------------------------------
if "page" not in st.session_state:
    st.session_state.page = "setup"
if "logged_in" not in st.session_state:
    st.session_state.logged_in = False
if "cart" not in st.session_state:
    st.session_state.cart = []
if "uid" not in st.session_state:
    st.session_state.uid = None


# =============================================================================
# PAGE FUNCTIONS
# =============================================================================

# -----------------------------------------------------------------------------
# 1️⃣ Setup Page
# -----------------------------------------------------------------------------
def setup_page():
    st.header("Welcome to Bill Bharat POS")
    st.write("Manage your sales, inventory, and reports in one place.")
    if st.button("Start"):
        st.session_state.page = "auth"


# -----------------------------------------------------------------------------
# 2️⃣ Authentication Page
# -----------------------------------------------------------------------------
def auth_page():
    st.header("Login / Register")

    option = st.radio("Select Action", ["Login", "Register"])

    email = st.text_input("Email")
    password = st.text_input("Password", type="password")

    if option == "Register":
        shop_name = st.text_input("Shop Name")
        phone = st.text_input("Phone")

        if st.button("Register"):
            try:
                user = AUTH.create_user(email=email, password=password)
                DB.collection("users").document(user.uid).set({
                    "email": email,
                    "shop_name": shop_name,
                    "phone": phone,
                })
                st.success("✅ Registration successful! You can now login.")
            except Exception as e:
                st.error(f"Registration failed: {e}")

    else:  # Login
        if st.button("Login"):
            try:
                # Use Firebase REST API for password verification
                api_key = st.secrets["FIREBASE_API_KEY"]
                url = f"https://identitytoolkit.googleapis.com/v1/accounts:signInWithPassword?key={api_key}"
                payload = {"email": email, "password": password, "returnSecureToken": True}
                res = requests.post(url, data=payload)
                data = res.json()

                if "idToken" in data:
                    st.session_state.logged_in = True
                    user = AUTH.get_user_by_email(email)
                    st.session_state.uid = user.uid
                    st.session_state.page = "dashboard"
                    st.success("✅ Login successful!")
                else:
                    st.error("Invalid credentials.")
            except Exception as e:
                st.error(f"Login failed: {e}")


# -----------------------------------------------------------------------------
# 3️⃣ Dashboard
# -----------------------------------------------------------------------------
def dashboard():
    st.sidebar.title("Dashboard")
    menu = st.sidebar.radio("Menu", ["Sales", "Inventory", "Reports", "Logout"])

    if menu == "Sales":
        sales_tab()
    elif menu == "Inventory":
        inventory_tab()
    elif menu == "Reports":
        reports_tab()
    elif menu == "Logout":
        st.session_state.clear()
        st.session_state.page = "setup"
        st.success("Logged out successfully!")


# -----------------------------------------------------------------------------
# Sales Tab
# -----------------------------------------------------------------------------
def sales_tab():
    st.subheader("🛒 Sales")

    products_ref = DB.collection("users").document(st.session_state.uid).collection("products")
    products = [p.to_dict() for p in products_ref.stream()]

    if not products:
        st.warning("No products found. Please add some in Inventory.")
        return

    for prod in products:
        col1, col2, col3 = st.columns([4, 2, 2])
        with col1:
            st.write(f"**{prod['name']}**")
        with col2:
            qty = st.number_input(f"Qty for {prod['name']}", min_value=0, step=1, key=prod['name'])
        with col3:
            if st.button("Add to Cart", key=f"add_{prod['name']}"):
                st.session_state.cart.append({
                    "sku": prod["sku"],
                    "name": prod["name"],
                    "price": prod["price"],
                    "qty": qty
                })
                st.success(f"Added {qty} x {prod['name']}")

    if st.session_state.cart:
        st.write("### Cart Items")
        total = 0
        for item in st.session_state.cart:
            st.write(f"{item['qty']} x {item['name']} @ {CURRENCY}{item['price']}")
            total += item["qty"] * float(item["price"])
        st.write(f"**Total: {CURRENCY}{total}**")

        if st.button("Checkout"):
            sale_id = str(int(time.time()))
            for item in st.session_state.cart:
                DB.collection("users").document(st.session_state.uid).collection("sales").document(sale_id).set({
                    **item,
                    "timestamp": time.time()
                })
            st.session_state.cart = []
            st.success("✅ Sale completed!")


# -----------------------------------------------------------------------------
# Inventory Tab
# -----------------------------------------------------------------------------
def inventory_tab():
    st.subheader("📦 Inventory Management")

    name = st.text_input("Product Name")
    sku = st.text_input("SKU")
    price = st.number_input("Price", min_value=0.0)
    qty = st.number_input("Quantity", min_value=0, step=1)

    if st.button("Add Product"):
        try:
            DB.collection("users").document(st.session_state.uid).collection("products").document(sku).set({
                "name": name,
                "sku": sku,
                "price": price,
                "qty": qty
            })
            st.success("Product added successfully!")
        except Exception as e:
            st.error(f"Error adding product: {e}")

    st.write("### Existing Products")
    products_ref = DB.collection("users").document(st.session_state.uid).collection("products")
    products = [p.to_dict() for p in products_ref.stream()]
    if products:
        st.table(products)
    else:
        st.info("No products found.")


# -----------------------------------------------------------------------------
# Reports Tab
# -----------------------------------------------------------------------------
def reports_tab():
    st.subheader("📊 Sales Reports")

    sales_ref = DB.collection("users").document(st.session_state.uid).collection("sales")
    sales = [s.to_dict() for s in sales_ref.stream()]
    if not sales:
        st.info("No sales data found.")
        return

    total_sales = sum(float(s["price"]) * s["qty"] for s in sales)
    st.metric("Total Revenue", f"{CURRENCY}{total_sales:.2f}")
    st.write("### Detailed Sales Data")
    st.table(sales)


# =============================================================================
# MAIN APP FLOW
# =============================================================================
if st.session_state.page == "setup":
    setup_page()
elif st.session_state.page == "auth":
    auth_page()
elif st.session_state.page == "dashboard":
    dashboard()

# =============================================================================
# NGROK (Optional, for local testing only)
# =============================================================================
if os.getenv("RUN_WITH_NGROK") == "1":
    from pyngrok import ngrok
    ngrok.set_auth_token(st.secrets["NGROK_AUTHTOKEN"])
    public_url = ngrok.connect(PORT)
    st.write(f"App running at: {public_url}")
