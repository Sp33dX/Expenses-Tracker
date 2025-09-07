import streamlit as st
import pandas as pd
import altair as alt
from datetime import datetime

# File to store data
FILE = "expenses.csv"

# Load data
@st.cache_data
def load_data():
    try:
        return pd.read_csv(FILE, parse_dates=["Date"])
    except FileNotFoundError:
        return pd.DataFrame(columns=["Date", "Type", "Category", "Description", "Amount", "Balance"])

# Save data
def save_data(df):
    df.to_csv(FILE, index=False)

# Update balances after new entry
def update_balance(df, starting_balance):
    df = df.sort_values("Date").reset_index(drop=True)
    balance = starting_balance
    balances = []
    for _, row in df.iterrows():
        if row["Type"] == "Expense":
            balance -= row["Amount"]
        else:  # Salary or Income
            balance += row["Amount"]
        balances.append(balance)
    df["Balance"] = balances
    return df

# App title
st.title("💰 Expenses & Balance Tracker")

# Load existing data
df = load_data()

# Sidebar: Setup
st.sidebar.header("Settings")
starting_balance = st.sidebar.number_input("Starting Balance", min_value=0.0, step=0.01, value=0.0)
salary = st.sidebar.number_input("Monthly Salary", min_value=0.0, step=0.01, value=0.0)

# Sidebar: Add Expense
st.sidebar.header("Add Transaction")
with st.sidebar.form("transaction_form"):
    date = st.date_input("Date", datetime.today())
    trans_type = st.selectbox("Type", ["Expense", "Income"])
    category = st.selectbox("Category", ["Food", "Transport", "Entertainment", "Bills", "Other"])
    description = st.text_input("Description")
    amount = st.number_input("Amount", min_value=0.0, step=0.01)
    submitted = st.form_submit_button("Add Transaction")

    if submitted and amount > 0:
        new_entry = pd.DataFrame([[date, trans_type, category, description, amount, 0]],
                                 columns=["Date", "Type", "Category", "Description", "Amount", "Balance"])
        df = pd.concat([df, new_entry], ignore_index=True)
        df = update_balance(df, starting_balance)
        save_data(df)
        st.sidebar.success("Transaction added!")

# Salary button
if st.sidebar.button("Salary Received"):
    new_entry = pd.DataFrame([[datetime.today(), "Income", "Salary", "Monthly Salary", salary, 0]],
                             columns=["Date", "Type", "Category", "Description", "Amount", "Balance"])
    df = pd.concat([df, new_entry], ignore_index=True)
    df = update_balance(df, starting_balance)
    save_data(df)
    st.sidebar.success("Salary added!")

# Reload updated data
df = load_data()
df = update_balance(df, starting_balance)

# Display transactions
st.subheader("All Transactions")
st.dataframe(df)

# Summary section
if not df.empty:
    st.subheader("Summary")
    total_expenses = df[df["Type"] == "Expense"]["Amount"].sum()
    total_income = df[df["Type"] == "Income"]["Amount"].sum()
    current_balance = df["Balance"].iloc[-1]

    col1, col2, col3 = st.columns(3)
    col1.metric("Total Expenses", f"${total_expenses:,.2f}")
    col2.metric("Total Income", f"${total_income:,.2f}")
    col3.metric("Current Balance", f"${current_balance:,.2f}")

    # Expense chart
    chart = alt.Chart(df[df["Type"] == "Expense"]).mark_bar().encode(
        x="Category",
        y="Amount",
        color="Category"
    )
    st.altair_chart(chart, use_container_width=True)

    # Filter by date range
    st.subheader("Filter by Date Range")
    start_date = st.date_input("Start Date", df["Date"].min().date())
    end_date = st.date_input("End Date", df["Date"].max().date())
    filtered = df[(df["Date"] >= pd.to_datetime(start_date)) & (df["Date"] <= pd.to_datetime(end_date))]
    st.dataframe(filtered)

    # Download option
    st.download_button("Download Transactions CSV", df.to_csv(index=False), "transactions.csv", "text/csv")
