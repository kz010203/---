# sudny-bot
Мой первый репозитори на гитхаб

from aiogram import Bot, Dispatcher, types
from aiogram.types import ReplyKeyboardMarkup, KeyboardButton, InlineKeyboardMarkup, InlineKeyboardButton
from aiogram.utils import executor
import asyncio
import random

API_TOKEN = '7806267460:AAEYkxaR_VcFeiK56gJSjU13JgO1bcgQJ-s'

bot = Bot(token=API_TOKEN)
dp = Dispatcher(bot)

# Игровые данные
players = {}
game_started = False
apocalypse = None
votes = {}

APOCALYPSES = {
    "вирус": {
        "good": ["Врач", "Биолог", "Медсестра", "Психолог"],
        "bad": ["СПИД", "Диабет", "Клаустрофобия", "Гематофобия"],
        "useless": ["Артист", "Блогер"]
    },
    "ядерная война": {
        "good": ["Инженер", "Электрик", "Солдат", "Химик"],
        "bad": ["Онкология", "Астма", "Никтофобия"],
        "useless": ["Флорист", "Юрист"]
    }
}

PROFESSIONS = ["Врач", "Биолог", "Инженер", "Электрик", "Солдат", "Химик", "Юрист", "Флорист"]
DISEASES = ["СПИД", "Диабет", "Клаустрофобия", "Гематофобия", "Онкология", "Астма", "Никтофобия"]
FEARS = ["Агорафобия", "Социофобия", "Акрофобия", "Аэрофобия", "Технофобия"]
ITEMS = ["Топор", "Рация", "Аптечка", "Оружие", "Радио", "Библия", "Пауэрбанк"]
TRAITS = ["Лидер", "Трус", "Манипулятор", "Добряк", "Психопат", "Рационал"]

@dp.message_handler(commands=['startgame'])
async def start_game(message: types.Message):
    global players, game_started, apocalypse, votes
    players = {}
    votes = {}
    game_started = True
    apocalypse = random.choice(list(APOCALYPSES.keys()))
    await message.answer(f"🎮 Игра началась! Апокалипсис: {apocalypse.upper()}\nИгроки, напишите /join, чтобы участвовать")

@dp.message_handler(commands=['join'])
async def join_game(message: types.Message):
    global players
    if not game_started:
        await message.answer("Сначала нужно запустить игру командой /startgame")
        return
    user_id = message.from_user.id
    if user_id in players:
        await message.answer("Вы уже в игре!")
        return
    players[user_id] = {"name": message.from_user.full_name}
    await message.answer(f"{message.from_user.full_name} присоединился к игре!")

@dp.message_handler(commands=['deal'])
async def deal_cards(message: types.Message):
    if not game_started or not players:
        await message.answer("Нет активной игры или игроков")
        return
    for user_id in players:
        prof = random.choice(PROFESSIONS)
        disease = random.choice(DISEASES)
        fear = random.choice(FEARS)
        item = random.choice(ITEMS)
        trait = random.choice(TRAITS)

        usefulness = "❌ Бесполезен"
        if prof in APOCALYPSES[apocalypse]["good"]:
            usefulness = "✅ Полезен"
        elif prof in APOCALYPSES[apocalypse]["bad"]:
            usefulness = "⚠️ Проблемный"

        players[user_id].update({
            "profession": prof,
            "disease": disease,
            "fear": fear,
            "item": item,
            "trait": trait,
            "usefulness": usefulness
        })
        await bot.send_message(user_id,
            f"Ваша карточка:\n👤 Профессия: {prof}\n⚕️ Болезнь: {disease}\n😨 Фобия: {fear}\n🎒 Предмет: {item}\n🧠 Черта характера: {trait}\n📊 Оценка: {usefulness}")
    await message.answer("Все карточки розданы. Обсуждение началось!")

@dp.message_handler(commands=['mycard'])
async def my_card(message: types.Message):
    user_id = message.from_user.id
    if user_id not in players or 'profession' not in players[user_id]:
        await message.answer("Вам ещё не раздавали карточку")
        return
    data = players[user_id]
    await message.answer(
        f"Ваша карточка:\n👤 Профессия: {data['profession']}\n⚕️ Болезнь: {data['disease']}\n😨 Фобия: {data['fear']}\n🎒 Предмет: {data['item']}\n🧠 Черта: {data['trait']}\n📊 Оценка: {data['usefulness']}")

@dp.message_handler(commands=['vote'])
async def vote_handler(message: types.Message):
    if not players:
        await message.answer("Нет игроков для голосования")
        return

    keyboard = InlineKeyboardMarkup()
    for user_id, data in players.items():
        keyboard.add(InlineKeyboardButton(text=data['name'], callback_data=f"vote_{user_id}"))

    await message.answer("🗳️ За кого голосуешь?", reply_markup=keyboard)

@dp.callback_query_handler(lambda c: c.data.startswith('vote_'))
async def process_vote(callback_query: types.CallbackQuery):
    voter = callback_query.from_user.id
    voted_id = int(callback_query.data.split('_')[1])
    votes[voter] = voted_id
    await bot.answer_callback_query(callback_query.id, text="Голос принят")
    await bot.send_message(callback_query.from_user.id, "Вы проголосовали")

@dp.message_handler(commands=['results'])
async def show_results(message: types.Message):
    summary = "\n".join([
        f"{data['name']} — {data.get('profession', '❓')} / {data.get('disease', '❓')} / {data.get('fear', '❓')} / {data.get('item', '❓')} / {data.get('trait', '❓')} / {data.get('usefulness', '❓')}"
        for data in players.values()
    ])
    await message.answer(f"🧾 Игроки:\n{summary}")

@dp.message_handler(commands=['eliminate'])
async def eliminate_player(message: types.Message):
    if not votes:
        await message.answer("Голосов нет")
        return
    tally = {}
    for voted_id in votes.values():
        tally[voted_id] = tally.get(voted_id, 0) + 1
    eliminated_id = max(tally, key=tally.get)
    eliminated_name = players[eliminated_id]['name']
    del players[eliminated_id]
    await message.answer(f"🚫 По итогам голосования выбыл: {eliminated_name}")
    votes.clear()

if __name__ == '__main__':
    executor.start_polling(dp, skip_updates=True)
