import sqlite3
from datetime import datetime, timedelta
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    Application,
    CommandHandler,
    CallbackQueryHandler,
    ContextTypes,
    ConversationHandler,
    MessageHandler,
    filters,
)

# ============================================================
# SETTINGS
# ============================================================

BOT_TOKEN = "YOUR_BOT_TOKEN_HERE"

# Your Telegram Chat/User ID
YOUR_CHAT_ID = 7779241353

# Australian time
TIMEZONE = "Australia/Hobart"

DB_FILE = "money_bot.db"

# Conversation states
(
    INCOME_AMOUNT,
    BILL_NAME,
    BILL_AMOUNT,
    BILL_DATE,
    BILL_FREQUENCY,
    SAVINGS_NAME,
    SAVINGS_TARGET,
    SAVINGS_AMOUNT,
) = range(8)


# ============================================================
# DATABASE
# ============================================================

def db():
    return sqlite3.connect(DB_FILE)


def setup_database():
    conn = db()
    cur = conn.cursor()

    cur.execute("""
        CREATE TABLE IF NOT EXISTS income (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            amount REAL NOT NULL,
            description TEXT,
            date TEXT NOT NULL
        )
    """)

    cur.execute("""
        CREATE TABLE IF NOT EXISTS bills (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            amount REAL NOT NULL,
            due_date TEXT NOT NULL,
            frequency TEXT DEFAULT 'once',
            paid INTEGER DEFAULT 0
        )
    """)

    cur.execute("""
        CREATE TABLE IF NOT EXISTS savings (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            target REAL NOT NULL,
            amount REAL DEFAULT 0
        )
    """)

    cur.execute("""
        CREATE TABLE IF NOT EXISTS payments (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            description TEXT NOT NULL,
            amount REAL NOT NULL,
            date TEXT NOT NULL
        )
    """)

    conn.commit()
    conn.close()


# ============================================================
# SECURITY
# ============================================================

def authorised(update: Update):
    return update.effective_user.id == YOUR_CHAT_ID


async def private_only(update: Update):
    if not authorised(update):
        if update.message:
            await update.message.reply_text(
                "🔒 Sorry, this is a private money bot."
            )
        elif update.callback_query:
            await update.callback_query.answer(
                "🔒 Private bot.",
                show_alert=True
            )
        return False

    return True


# ============================================================
# MAIN MENU
# ============================================================

def main_menu():
    keyboard = [
        [
            InlineKeyboardButton("💵 Income", callback_data="income"),
            InlineKeyboardButton("📅 Bills", callback_data="bills"),
        ],
        [
            InlineKeyboardButton("🏦 Savings", callback_data="savings"),
            InlineKeyboardButton("💳 Payments", callback_data="payments"),
        ],
        [
            InlineKeyboardButton("📊 Overview", callback_data="overview"),
        ],
    ]

    return InlineKeyboardMarkup(keyboard)


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):

    if not await private_only(update):
        return

    await update.message.reply_text(
        "💰 *HOUSEHOLD MONEY BOT*\n\n"
        "What would you like to do?",
        reply_markup=main_menu(),
        parse_mode="Markdown",
    )


# ============================================================
# OVERVIEW
# ============================================================

async def overview(update: Update, context: ContextTypes.DEFAULT_TYPE):

    if not await private_only(update):
        return

    conn = db()
    cur = conn.cursor()

    cur.execute("SELECT COALESCE(SUM(amount), 0) FROM income")
    income = cur.fetchone()[0]

    cur.execute(
        "SELECT COALESCE(SUM(amount), 0) FROM bills WHERE paid = 0"
    )
    unpaid_bills = cur.fetchone()[0]

    cur.execute("SELECT COALESCE(SUM(amount), 0) FROM payments")
    payments = cur.fetchone()[0]

    cur.execute("SELECT name, amount, target FROM savings")
    savings = cur.fetchall()

    conn.close()

    savings_total = sum(x[1] for x in savings)
    savings_targets = sum(x[2] for x in savings)

    available = income - payments - unpaid_bills

    text = (
        "📊 *MONEY OVERVIEW*\n\n"
        f"💵 Income recorded: ${income:,.2f}\n"
        f"💳 Payments: ${payments:,.2f}\n"
        f"📅 Unpaid bills: ${unpaid_bills:,.2f}\n"
        f"💰 Available after bills: ${available:,.2f}\n\n"
        f"🏦 Savings: ${savings_total:,.2f}"
    )

    if savings_targets > 0:
        text += f" / ${savings_targets:,.2f}"

    if savings:
        text += "\n\n🎯 *Savings Goals*"

        for name, amount, target in savings:
            percent = (amount / target * 100) if target else 0

            text += (
                f"\n\n{name}"
                f"\n${amount:,.2f} / ${target:,.2f}"
                f"\nProgress: {percent:.0f}%"
            )

    keyboard = [
        [InlineKeyboardButton("⬅️ Main Menu", callback_data="menu")]
    ]

    await update.callback_query.edit_message_text(
        text,
        reply_markup=InlineKeyboardMarkup(keyboard),
        parse_mode="Markdown",
    )


# ============================================================
# INCOME
# ============================================================

async def income_menu(update: Update, context: ContextTypes.DEFAULT_TYPE):

    if not await private_only(update):
        return ConversationHandler.END

    keyboard = [
        [InlineKeyboardButton("➕ Add Income", callback_data="add_income")],
        [InlineKeyboardButton("⬅️ Main Menu", callback_data="menu")],
    ]

    await update.callback_query.edit_message_text(
        "💵 *INCOME*",
        reply_markup=InlineKeyboardMarkup(keyboard),
        parse_mode="Markdown",
    )

    return ConversationHandler.END


async def add_income_start(update: Update, context: ContextTypes.DEFAULT_TYPE):

    if not await private_only(update):
        return ConversationHandler.END

    await update.callback_query.answer()

    await update.callback_query.edit_message_text(
        "💵 How much income did you receive?\n\n"
        "Example: `1200`",
        parse_mode="Markdown",
    )

    return INCOME_AMOUNT


async def receive_income(update: Update, context: ContextTypes.DEFAULT_TYPE):

    if not authorised(update):
        return ConversationHandler.END

    try:
        amount = float(update.message.text.replace("$", "").replace(",", ""))
    except ValueError:
        await update.message.reply_text(
            "❌ Please enter just the amount.\nExample: 1200"
        )
        return INCOME_AMOUNT

    conn = db()
    cur = conn.cursor()

    cur.execute(
        "INSERT INTO income (amount, description, date) VALUES (?, ?, ?)",
        (amount, "Income", datetime.now().isoformat()),
    )

    conn.commit()
    conn.close()

    await update.message.reply_text(
        f"✅ Income recorded: ${amount:,.2f}",
        reply_markup=main_menu(),
    )

    return ConversationHandler.END


# ============================================================
# BILLS
# ============================================================

async def bills_menu(update: Update, context: ContextTypes.DEFAULT_TYPE):

    if not await private_only(update):
        return

    conn = db()
    cur = conn.cursor()

    cur.execute("""
        SELECT id, name, amount, due_date, paid
        FROM bills
        ORDER BY due_date
    """)

    bills = cur.fetchall()
    conn.close()

    text = "📅 *BILLS*\n\n"

    if not bills:
        text += "No bills added yet."
    else:
        for bill_id, name, amount, due_date, paid in bills:

            status = "✅ PAID" if paid else "🔴 UNPAID"

            text += (
                f"*{name}*\n"
                f"${amount:,.2f} — due {due_date}\n"
                f"{status}\n\n"
            )

    keyboard = [
        [InlineKeyboardButton("➕ Add Bill", callback_data="add_bill")],
        [InlineKeyboardButton("⬅️ Main Menu", callback_data="menu")],
    ]

    await update.callback_query.edit_message_text(
        text,
        reply_markup=InlineKeyboardMarkup(keyboard),
        parse_mode="Markdown",
    )


async def add_bill_start(update: Update, context: ContextTypes.DEFAULT_TYPE):

    if not await private_only(update):
        return ConversationHandler.END

    await update.callback_query.answer()

    await update.callback_query.edit_message_text(
        "📅 What is the bill called?\n\n"
        "Example: Electricity"
    )

    return BILL_NAME


async def receive_bill_name(update: Update, context: ContextTypes.DEFAULT_TYPE):

    if not authorised(update):
        return ConversationHandler.END

    context.user_data["bill_name"] = update.message.text

    await update.message.reply_text(
        "💰 How much is the bill?\n\n"
        "Example: 180"
    )

    return BILL_AMOUNT


async def receive_bill_amount(update: Update, context: ContextTypes.DEFAULT_TYPE):

    if not authorised(update):
        return ConversationHandler.END

    try:
        amount = float(update.message.text.replace("$", "").replace(",", ""))
    except ValueError:
        await update.message.reply_text("❌ Enter the amount, e.g. 180")
        return BILL_AMOUNT

    context.user_data["bill_amount"] = amount

    await update.message.reply_text(
        "📅 What date is it due?\n\n"
        "Use DD/MM/YYYY\n"
        "Example: 10/09/2026"
    )

    return BILL_DATE


async def receive_bill_date(update: Update, context: ContextTypes.DEFAULT_TYPE):

    if not authorised(update):
        return ConversationHandler.END

    try:
        date = datetime.strptime(
            update.message.text,
            "%d/%m/%Y"
        ).strftime("%Y-%m-%d")
    except ValueError:
        await update.message.reply_text(
            "❌ Please use DD/MM/YYYY\nExample: 10/09/2026"
        )
        return BILL_DATE

    context.user_data["bill_date"] = date

    keyboard = [
        [
            InlineKeyboardButton("Once", callback_data="freq_once"),
            InlineKeyboardButton("Weekly", callback_data="freq_weekly"),
        ],
        [
            InlineKeyboardButton("Fortnightly", callback_data="freq_fortnightly"),
            InlineKeyboardButton("Monthly", callback_data="freq_monthly"),
        ],
        [
            InlineKeyboardButton("Yearly", callback_data="freq_yearly"),
        ],
    ]

    await update.message.reply_text(
        "🔄 How often does this bill occur?",
        reply_markup=InlineKeyboardMarkup(keyboard),
    )

    return BILL_FREQUENCY


async def receive_bill_frequency(
    update: Update,
    context: ContextTypes.DEFAULT_TYPE
):

    if not await private_only(update):
        return ConversationHandler.END

    query = update.callback_query
    await query.answer()

    frequencies = {
        "freq_once": "once",
        "freq_weekly": "weekly",
        "freq_fortnightly": "fortnightly",
        "freq_monthly": "monthly",
        "freq_yearly": "yearly",
    }

    frequency = frequencies.get(query.data, "once")

    conn = db()
    cur = conn.cursor()

    cur.execute("""
        INSERT INTO bills
        (name, amount, due_date, frequency, paid)
        VALUES (?, ?, ?, ?, 0)
    """, (
        context.user_data["bill_name"],
        context.user_data["bill_amount"],
        context.user_data["bill_date"],
        frequency,
    ))

    conn.commit()
    conn.close()

    await query.edit_message_text(
        f"✅ *Bill added!*\n\n"
        f"📅 {context.user_data['bill_name']}\n"
        f"💰 ${context.user_data['bill_amount']:,.2f}\n"
        f"📆 Due: {context.user_data['bill_date']}\n"
        f"🔄 {frequency.title()}",
        parse_mode="Markdown",
    )

    context.user_data.clear()

    return ConversationHandler.END


# ============================================================
# SAVINGS
# ============================================================

async def savings_menu(update: Update, context: ContextTypes.DEFAULT_TYPE):

    if not await private_only(update):
        return

    conn = db()
    cur = conn.cursor()

    cur.execute("SELECT id, name, amount, target FROM savings")
    goals = cur.fetchall()

    conn.close()

    text = "🏦 *SAVINGS GOALS*\n\n"

    if not goals:
        text += "No savings goals yet."
    else:
        for goal_id, name, amount, target in goals:
            percent = (amount / target * 100) if target else 0

            text += (
                f"*{name}*\n"
                f"${amount:,.2f} / ${target:,.2f}\n"
                f"{percent:.0f}% complete\n\n"
            )

    keyboard = [
        [InlineKeyboardButton("➕ New Savings Goal", callback_data="add_savings")],
        [InlineKeyboardButton("⬅️ Main Menu", callback_data="menu")],
    ]

    await update.callback_query.edit_message_text(
        text,
        reply_markup=InlineKeyboardMarkup(keyboard),
        parse_mode="Markdown",
    )


async def add_savings_start(update: Update, context: ContextTypes.DEFAULT_TYPE):

    if not await private_only(update):
        return ConversationHandler.END

    await update.callback_query.answer()

    await update.callback_query.edit_message_text(
        "🏦 What are you saving for?\n\n"
        "Example: Kids Activities"
    )

    return SAVINGS_NAME


async def receive_savings_name(update: Update, context: ContextTypes.DEFAULT_TYPE):

    if not authorised(update):
        return ConversationHandler.END

    context.user_data["savings_name"] = update.message.text

    await update.message.reply_text(
        "🎯 What is your savings target?\n\n"
        "Example: 1000"
    )

    return SAVINGS_TARGET


async def receive_savings_target(update: Update, context: ContextTypes.DEFAULT_TYPE):

    if not authorised(update):
        return ConversationHandler.END

    try:
        target = float(update.message.text.replace("$", "").replace(",", ""))
    except ValueError:
        await update.message.reply_text("❌ Enter a number, e.g. 1000")
        return SAVINGS_TARGET

    context.user_data["savings_target"] = target

    conn = db()
    cur = conn.cursor()

    cur.execute(
        "INSERT INTO savings (name, target, amount) VALUES (?, ?, 0)",
        (
            context.user_data["savings_name"],
            target,
        )
    )

    conn.commit()
    conn.close()

    await update.message.reply_text(
        f"✅ Savings goal created!\n\n"
        f"🏦 {context.user_data['savings_name']}\n"
        f"🎯 Target: ${target:,.2f}",
        reply_markup=main_menu(),
    )

    context.user_data.clear()

    return ConversationHandler.END


# ============================================================
# PAYMENTS
# ============================================================

async def payments_menu(update: Update, context: ContextTypes.DEFAULT_TYPE):

    if not await private_only(update):
        return

    conn = db()
    cur = conn.cursor()

    cur.execute("""
        SELECT description, amount, date
        FROM payments
        ORDER BY date DESC
        LIMIT 10
    """)

    payments = cur.fetchall()
    conn.close()

    text = "💳 *RECENT PAYMENTS*\n\n"

    if not payments:
        text += "No payments recorded yet."
    else:
        for description, amount, date in payments:
            date_display = date[:10]
            text += f"• {description}: ${amount:,.2f} ({date_display})\n"

    keyboard = [
        [InlineKeyboardButton("⬅️ Main Menu", callback_data="menu")]
    ]

    await update.callback_query.edit_message_text(
        text,
        reply_markup=InlineKeyboardMarkup(keyboard),
        parse_mode="Markdown",
    )


# ============================================================
# CALLBACK ROUTER
# ============================================================

async def button_router(update: Update, context: ContextTypes.DEFAULT_TYPE):

    query = update.callback_query

    if not await private_only(update):
        return

    await query.answer()

    choice = query.data

    if choice == "menu":
        await query.edit_message_text(
            "💰 *HOUSEHOLD MONEY BOT*\n\n"
            "What would you like to do?",
            reply_markup=main_menu(),
            parse_mode="Markdown",
        )

    elif choice == "overview":
        await overview(update, context)

    elif choice == "income":
        await income_menu(update, context)

    elif choice == "bills":
        await bills_menu(update, context)

    elif choice == "savings":
        await savings_menu(update, context)

    elif choice == "payments":
        await payments_menu(update, context)


# ============================================================
# BILL REMINDERS
# ============================================================

async def bill_reminder_job(context: ContextTypes.DEFAULT_TYPE):

    conn = db()
    cur = conn.cursor()

    today = datetime.now().date()

    cur.execute("""
        SELECT id, name, amount, due_date
        FROM bills
        WHERE paid = 0
    """)

    bills = cur.fetchall()

    for bill_id, name, amount, due_date in bills:

        try:
            due = datetime.strptime(
                due_date,
                "%Y-%m-%d"
            ).date()
        except ValueError:
            continue

        days = (due - today).days

        if days in [7, 3, 1, 0]:

            if days == 0:
                timing = "🚨 DUE TODAY"
            elif days == 1:
                timing = "⚠️ DUE TOMORROW"
            else:
                timing = f"🔔 DUE IN {days} DAYS"

            await context.bot.send_message(
                chat_id=YOUR_CHAT_ID,
                text=(
                    f"{timing}\n\n"
                    f"📅 {name}\n"
                    f"💰 ${amount:,.2f}\n"
                    f"📆 Due: {due.strftime('%d/%m/%Y')}"
                ),
            )

    conn.close()


# ============================================================
# COMMANDS
# ============================================================

async def cancel(update: Update, context: ContextTypes.DEFAULT_TYPE):

    if authorised(update):
        context.user_data.clear()

        await update.message.reply_text(
            "❌ Cancelled.",
            reply_markup=main_menu(),
        )

    return ConversationHandler.END


# ============================================================
# MAIN
# ============================================================

def main():

    setup_database()

    app = (
        Application.builder()
        .token(BOT_TOKEN)
        .build()
    )

    # Income conversation
    income_conversation = ConversationHandler(
        entry_points=[
            CallbackQueryHandler(
                add_income_start,
                pattern="^add_income$"
            )
        ],
        states={
            INCOME_AMOUNT: [
                MessageHandler(
                    filters.TEXT & ~filters.COMMAND,
                    receive_income
                )
            ],
        },
        fallbacks=[
            CommandHandler("cancel", cancel)
        ],
    )

    # Bills conversation
    bill_conversation = ConversationHandler(
        entry_points=[
            CallbackQueryHandler(
                add_bill_start,
                pattern="^add_bill$"
            )
        ],
        states={
            BILL_NAME: [
                MessageHandler(
                    filters.TEXT & ~filters.COMMAND,
                    receive_bill_name
                )
            ],
            BILL_AMOUNT: [
                MessageHandler(
                    filters.TEXT & ~filters.COMMAND,
                    receive_bill_amount
                )
            ],
            BILL_DATE: [
                MessageHandler(
                    filters.TEXT & ~filters.COMMAND,
                    receive_bill_date
                )
            ],
            BILL_FREQUENCY: [
                CallbackQueryHandler(
                    receive_bill_frequency,
                    pattern="^freq_"
                )
            ],
        },
        fallbacks=[
            CommandHandler("cancel", cancel)
        ],
    )

    # Savings conversation
    savings_conversation = ConversationHandler(
        entry_points=[
            CallbackQueryHandler(
                add_savings_start,
                pattern="^add_savings$"
            )
        ],
        states={
            SAVINGS_NAME: [
                MessageHandler(
                    filters.TEXT & ~filters.COMMAND,
                    receive_savings_name
                )
            ],
            SAVINGS_TARGET: [
                MessageHandler(
                    filters.TEXT & ~filters.COMMAND,
                    receive_savings_target
                )
            ],
        },
        fallbacks=[
            CommandHandler("cancel", cancel)
        ],
    )

    app.add_handler(CommandHandler("start", start))

    app.add_handler(income_conversation)
    app.add_handler(bill_conversation)
    
