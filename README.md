# rafflebot
raffle bot for telegram
import random
import sqlite3
from telegram import Update
from telegram.ext import Updater, CommandHandler, CallbackContext, Filters

# Configuration
TOTAL_TICKETS = 30
ADDRESS = "0x329633ed1ca833d0f1577aeffb8333bdb678b27f"
PAYMENT_ADDRESS = ADDRESS
prize = "Default Prize"
entry_fee = 10.0

ADMIN_USERNAMES = ["@Adam_HAAGA420_FAMS", "@R2R_Auwyne_DeFi_IRA", "@AG_R2R_IRA"]

# SQLite database setup for persistence
conn = sqlite3.connect('raffle.db', check_same_thread=False)
c = conn.cursor()

def setup_database():
    c.execute('''
        CREATE TABLE IF NOT EXISTS raffle_entries (
            address TEXT PRIMARY KEY
        )
    ''')
    c.execute('''
        CREATE TABLE IF NOT EXISTS previous_winners (
            winner_address TEXT,
            prize TEXT
        )
    ''')
    conn.commit()

setup_database()

def start_command(update: Update, context: CallbackContext):
    if update.effective_user.username in ADMIN_USERNAMES:
        update.message.reply_text("Welcome Admin. Use the /setup command to configure the raffle.")

def setup_command(update: Update, context: CallbackContext):
    if update.effective_user.username not in ADMIN_USERNAMES:
        return update.message.reply_text("You are not authorized to setup this bot.")

    try:
        new_fee, *new_prize = context.args
        global entry_fee, prize
        entry_fee = float(new_fee)
        prize = " ".join(new_prize)
        update.message.reply_text(f"Raffle setup complete.\nEntry Fee: {entry_fee}\nPrize: {prize}")
    except ValueError:
        update.message.reply_text("Invalid input. Use command: /setup <entry_fee> <prize>")

def sell_ticket(update: Update, context: CallbackContext):
    global TOTAL_TICKETS
    if update.effective_user.username not in ADMIN_USERNAMES:
        return update.message.reply_text("You are not authorized to sell tickets.")

    try:
        payment_address = context.args[0]
    except IndexError:
        return update.message.reply_text("Include the payment address. Use: /sell <address>")

    c.execute('SELECT * FROM raffle_entries WHERE address=?', (payment_address,))
    if c.fetchone() is not None:
        return update.message.reply_text("This address has already purchased a ticket.")
    
    c.execute('INSERT INTO raffle_entries (address) VALUES (?)', (payment_address,))
    conn.commit()

    ticket_count = c.execute('SELECT COUNT(*) FROM raffle_entries').fetchone()[0]
    remaining_tickets = TOTAL_TICKETS - ticket_count

    context.bot.send_message(chat_id="@DeFiIRA_Group", text=f"Ticket sold to {payment_address}! {remaining_tickets} tickets remaining.\nPlease send ${entry_fee} to {PAYMENT_ADDRESS}.")

    if ticket_count == TOTAL_TICKETS:
        announce_winner(context)

def announce_winner(context: CallbackContext):
    c.execute('SELECT address FROM raffle_entries')
    entries = c.fetchall()
    if not entries:
        return
    winner = random.choice(entries)[0]
    c.execute('INSERT INTO previous_winners (winner_address, prize) VALUES (?, ?)', (winner, prize))
    conn.commit()

    context.bot.send_message(chat_id="@DeFiIRA_Group", text=f"All tickets sold! The winner is {winner}.\nPrize: {prize}")

    reset_raffle()

def reset_raffle():
    global TOTAL_TICKETS
    
    c.execute('DELETE FROM raffle_entries')
    conn.commit()

def raffle_status(update: Update, context: CallbackContext):
    ticket_count = c.execute('SELECT COUNT(*) FROM raffle_entries').fetchone()[0]
    remaining_tickets = TOTAL_TICKETS - ticket_count
    
    entrants = c.execute('SELECT address FROM raffle_entries').fetchall()
    entry_list = ", ".join([address[0] for address in entrants])

    update.message.reply_text(
        f"Entries: {entry_list}\nSold: {ticket_count}/{TOTAL_TICKETS}\nRemaining: {remaining_tickets}\nEntry Fee: {entry_fee}\nPrize: {prize}\nPayment Address: {PAYMENT_ADDRESS}"
    )
    def winners_command(update: Update, context: CallbackContext):
    winners = c.execute('SELECT * FROM previous_winners').fetchall()
    if winners:
        winner_list = "\n".join([f"Winner: {w[0]}, Prize: {w[1]}" for w in winners])
        update.message.reply_text(f"Previous Winners:\n{winner_list}")
    else:
        update.message.reply_text("No previous winners recorded.")

def main():
    # Insert your bot's token here
    bot_token = "8111611527:AAG2LD04hLx0jezohHSEVKMNLLZqZ7lWezs"
    updater = Updater(bot_token, use_context=True)

    dispatcher = updater.dispatcher
    dispatcher.add_handler(CommandHandler("start", start_command))
    dispatcher.add_handler(CommandHandler("setup", setup_command, Filters.user(username=ADMIN_USERNAMES)))
    dispatcher.add_handler(CommandHandler("sell", sell_ticket, Filters.user(username=ADMIN_USERNAMES)))
    dispatcher.add_handler(CommandHandler("raffle", raffle_status))
    dispatcher.add_handler(CommandHandler("winners", winners_command))

    updater.start_polling()
    updater.idle()

if name == 'main':
    main()
    