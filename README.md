# referal-best-bot
import logging
import os
import sqlite3
from aiogram import Bot, Dispatcher, executor, types

TOKEN = os.getenv("BOT_TOKEN")
bot = Bot(token=TOKEN)
dp = Dispatcher(bot)

conn = sqlite3.connect("data.db")
cursor = conn.cursor()
cursor.execute("""
CREATE TABLE IF NOT EXISTS users (
    user_id INTEGER PRIMARY KEY,
    referrer INTEGER,
    count INTEGER DEFAULT 0,
    rewarded1 BOOLEAN DEFAULT 0,
    rewarded5 BOOLEAN DEFAULT 0,
    rewarded10 BOOLEAN DEFAULT 0
)
""")
conn.commit()

def add_user(user_id, referrer=None):
    cursor.execute("INSERT OR IGNORE INTO users (user_id, referrer) VALUES (?,?)", (user_id, referrer))
    if referrer:
        cursor.execute("UPDATE users SET count = count + 1 WHERE user_id = ?", (referrer,))
        conn.commit()

def check_rewards(user_id):
    cursor.execute("SELECT count, rewarded1, rewarded5, rewarded10 FROM users WHERE user_id = ?", (user_id,))
    row = cursor.fetchone()
    if not row: return None
    count, r1, r5, r10 = row
    messages = []
    if count >= 1 and not r1:
        messages.append("Поздравляем! 1 приглашённый — купон 10%. Напишите менеджеру.")
        cursor.execute("UPDATE users SET rewarded1=1 WHERE user_id=?", (user_id,))
    if count >= 5 and not r5:
        messages.append("Поздравляем! 5 приглашённых — купон 20% + очки. Напишите менеджеру.")
        cursor.execute("UPDATE users SET rewarded5=1 WHERE user_id=?", (user_id,))
    if count >= 10 and not r10:
        messages.append("Поздравляем! 10 приглашённых — купон 30% + очки + ремень. Напишите менеджеру.")
        cursor.execute("UPDATE users SET rewarded10=1 WHERE user_id=?", (user_id,))
    conn.commit()
    return messages

@dp.message_handler(commands=["start"])
async def cmd_start(message: types.Message):
    ref = None
    if message.get_args():
        try:
            ref = int(message.get_args())
        except: pass
    add_user(message.from_user.id, ref)
    link = f"https://t.me/Referal_best_bot?start={message.from_user.id}"
    await message.answer(f"Привет, {message.from_user.first_name}!\n"
                         f"Вот твоя реф. ссылка:\n{link}\n"
                         "Приглашай друзей и получай награды!")
    msgs = check_rewards(message.from_user.id)
    if msgs:
        for m in msgs:
            await message.answer(m)

@dp.message_handler(commands=["referrals"])
async def cmd_referrals(message: types.Message):
    cursor.execute("SELECT count FROM users WHERE user_id=?", (message.from_user.id,))
    row = cursor.fetchone()
    count = row[0] if row else 0
    await message.answer(f"У вас приглашено: {count} человек")

@dp.message_handler(commands=["help"])
async def cmd_help(message: types.Message):
    await message.answer("Реферальная программа:\n"
                         "1 друг → купон 10%\n"
                         "5 друзей → купон 20% + очки\n"
                         "10 друзей → купон 30% + очки + ремень\n\n"
                         "Команды: /referrals — статистика")

if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO)
    executor.start_polling(dp, skip_updates=True)
aiogram==2.25.1
# Referal_best_bot

## Установка и запуск

1. Создай репозиторий и загрузить файлы.
2. Добавь секрет `BOT_TOKEN` в Settings → Secrets.
3. На Render.com:
   - Новый Web Service → подключи репозиторий
   - Build Command: `pip install -r requirements.txt`
   - Start Command: `python bot.py`
   - Укажи `BOT_TOKEN` в Environment
4. Deploy.  
5. Проверь: напиши /start, отправь ссылку другому — проверь /referrals.
