import os
import logging
import asyncio
from datetime import datetime
from aiogram import Bot, Dispatcher, executor, types
from aiogram.contrib.fsm_storage.memory import MemoryStorage
from aiogram.dispatcher import FSMContext
from aiogram.dispatcher.filters.state import State, StatesGroup
from aiogram.types import ReplyKeyboardMarkup, KeyboardButton, InlineKeyboardMarkup, InlineKeyboardButton
from aiogram.utils.exceptions import BotBlocked, ChatNotFound
import sqlite3
import json

# Настройки из переменных окружения
API_TOKEN = os.getenv('API_TOKEN', 'YOUR_BOT_TOKEN')
ADMIN_CHAT_IDS = os.getenv('ADMIN_CHAT_IDS', 'ADMIN_CHAT_ID_1,ADMIN_CHAT_ID_2').split(',')
DATABASE_NAME = 'questionnaires.db'

# Инициализация бота
bot = Bot(token=API_TOKEN)
storage = MemoryStorage()
dp = Dispatcher(bot, storage=storage)
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

# Инициализация базы данных
def init_db():
    conn = sqlite3.connect(DATABASE_NAME)
    cursor = conn.cursor()
    
    # Таблица анкет
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS questionnaires (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        username TEXT,
        bank TEXT,
        sbp TEXT,
        okved TEXT,
        reg_date TEXT,
        other_accounts TEXT,
        services TEXT,
        status TEXT DEFAULT 'new',
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )
    ''')
    
    # Таблица для уведомлений
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS notifications (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        message TEXT,
        sent_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )
    ''')
    
    # Таблица для шаблонов ответов
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS response_templates (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT,
        text TEXT
    )
    ''')
    
    # Добавляем базовые шаблоны
    cursor.execute('SELECT COUNT(*) FROM response_templates')
    if cursor.fetchone()[0] == 0:
        templates = [
            ('Приветствие', 'Добрый день! Мы рассмотрели вашу заявку и готовы предложить...'),
            ('Отказ', 'К сожалению, в данный момент мы не можем принять ваше предложение...'),
            ('Дополнительная информация', 'Для дальнейшего рассмотрения вашей заявки нам нужна дополнительная информация...')
        ]
        cursor.executemany('INSERT INTO response_templates (name, text) VALUES (?, ?)', templates)
    
    conn.commit()
    conn.close()

init_db()

# Класс состояний
class Questionnaire(StatesGroup):
    waiting_for_bank = State()
    waiting_for_sbp = State()
    waiting_for_okved = State()
    waiting_for_reg_date = State()
    waiting_for_other_accounts = State()
    waiting_for_services = State()
    waiting_for_confirmation = State()

# Утилиты для работы с БД
def db_connection():
    return sqlite3.connect(DATABASE_NAME)

# Клавиатуры
def make_row_keyboard(items: list[str]) -> ReplyKeyboardMarkup:
    row = [KeyboardButton(text=item) for item in items]
    return ReplyKeyboardMarkup(keyboard=[row], resize_keyboard=True)

def get_cancel_keyboard():
    return ReplyKeyboardMarkup(resize_keyboard=True).add(KeyboardButton('❌ Отмена'))

def get_main_keyboard():
    keyboard = ReplyKeyboardMarkup(resize_keyboard=True)
    keyboard.add(KeyboardButton('📊 Предложить счет'))
    keyboard.add(KeyboardButton('📞 Предложить корп. связь'))
    keyboard.add(KeyboardButton('ℹ️ О нас'))
    return keyboard

def get_admin_keyboard():
    keyboard = ReplyKeyboardMarkup(resize_keyboard=True)
    keyboard.add(KeyboardButton('📊 Статистика'))
    keyboard.add(KeyboardButton('📋 Список заявок'))
    keyboard.add(KeyboardButton('📢 Рассылка'))
    keyboard.add(KeyboardButton('📝 Шаблоны ответов'))
    return keyboard

# Проверка является ли пользователь админом
def is_admin(user_id):
    return str(user_id) in ADMIN_CHAT_IDS

# Сохранение анкеты в базу данных
def save_questionnaire(data):
    with db_connection() as conn:
        cursor = conn.cursor()
        cursor.execute('''
        INSERT INTO questionnaires (user_id, username, bank, sbp, okved, reg_date, other_accounts, services)
        VALUES (?, ?, ?, ?, ?, ?, ?, ?)
        ''', (
            data['user_id'],
            data['username'],
            data['bank'],
            data['sbp'],
            data['okved'],
            data['reg_date'],
            data['other_accounts'],
            data['services']
        ))

# Получение статистики
def get_stats():
    with db_connection() as conn:
        cursor = conn.cursor()
        cursor.execute('SELECT COUNT(*) FROM questionnaires')
        total = cursor.fetchone()[0]
        cursor.execute('SELECT COUNT(*) FROM questionnaires WHERE status = "new"')
        new = cursor.fetchone()[0]
        return total, new

# Получение списка заявок
def get_questionnaires(limit=10, status='new'):
    with db_connection() as conn:
        cursor = conn.cursor()
        if status == 'all':
            cursor.execute('SELECT * FROM questionnaires ORDER BY created_at DESC LIMIT ?', (limit,))
        else:
            cursor.execute('SELECT * FROM questionnaires WHERE status = ? ORDER BY created_at DESC LIMIT ?', (status, limit))
        return cursor.fetchall()

# Обновление статуса заявки
def update_questionnaire_status(q_id, status):
    with db_connection() as conn:
        cursor = conn.cursor()
        cursor.execute('UPDATE questionnaires SET status = ? WHERE id = ?', (status, q_id))

# Получение шаблонов ответов
def get_templates():
    with db_connection() as conn:
        cursor = conn.cursor()
        cursor.execute('SELECT * FROM response_templates')
        return cursor.fetchall()

# Добавление шаблона ответа
def add_template(name, text):
    with db_connection() as conn:
        cursor = conn.cursor()
        cursor.execute('INSERT INTO response_templates (name, text) VALUES (?, ?)', (name, text))

# Система уведомлений
async def send_notification_to_admins(message_text):
    for admin_id in ADMIN_CHAT_IDS:
        try:
            await bot.send_message(admin_id, f"🔔 Уведомление: {message_text}")
            # Сохраняем в базу
            with db_connection() as conn:
                cursor = conn.cursor()
                cursor.execute('INSERT INTO notifications (message) VALUES (?)', (message_text,))
        except (BotBlocked, ChatNotFound):
            logging.error(f"Не удалось отправить уведомление админу {admin_id}")

# Начало диалога
@dp.message_handler(commands='start')
async def cmd_start(message: types.Message):
    if is_admin(message.from_user.id):
        await message.answer("Панель администратора", reply_markup=get_admin_keyboard())
        return
        
    await message.answer(
        "👋 Добро пожаловать! Мы — надежный партнер по аренде бизнес-счетов.\n\n"
        "✨ Почему выбирают нас:\n"
        "• Самые высокие ставки на рынке\n"
        "• Мгновенные выплаты\n"
        "• Индивидуальный подход к каждому клиенту\n"
        "• Полная конфиденциальность\n\n"
        "Выберите опцию:",
        reply_markup=get_main_keyboard()
    )

# Обработка кнопки "Предложить счет"
@dp.message_handler(lambda message: message.text == '📊 Предложить счет')
async def offer_account(message: types.Message):
    await message.answer(
        "Отлично! Мы предлагаем лучшие условия по аренде бизнес-счетов.\n\n"
        "Для начала выберите банк:",
        reply_markup=make_row_keyboard(["РСХБ", "ВТБ", "Уралсиб", "Другой", "❌ Отмена"])
    )
    await Questionnaire.waiting_for_bank.set()

# Обработка выбора банка
@dp.message_handler(state=Questionnaire.waiting_for_bank)
async def process_bank(message: types.Message, state: FSMContext):
    if message.text == '❌ Отмена':
        await message.answer("Заполнение анкеты отменено.", reply_markup=get_main_keyboard())
        await state.finish()
        return
        
    await state.update_data(bank=message.text)
    await state.update_data(user_id=message.from_user.id)
    await state.update_data(username=message.from_user.username)
    
    await message.answer(
        "Есть ли подключение СБП?\n\n"
        "• C2B - прием платежей от физлиц\n"
        "• B2B - прием платежей от юрлиц\n"
        "Укажите тип подключения:",
        reply_markup=make_row_keyboard(["C2B", "B2B", "Оба варианта", "Нет", "❌ Отмена"])
    )
    await Questionnaire.next()

# Обработка СБП
@dp.message_handler(state=Questionnaire.waiting_for_sbp)
async def process_sbp(message: types.Message, state: FSMContext):
    if message.text == '❌ Отмена':
        await message.answer("Заполнение анкеты отменено.", reply_markup=get_main_keyboard())
        await state.finish()
        return
        
    await state.update_data(sbp=message.text)
    await message.answer("Укажите основные ОКВЭДы через запятую:", reply_markup=get_cancel_keyboard())
    await Questionnaire.next()

# Обработка ОКВЭД
@dp.message_handler(state=Questionnaire.waiting_for_okved)
async def process_okved(message: types.Message, state: FSMContext):
    if message.text == '❌ Отмена':
        await message.answer("Заполнение анкеты отменено.", reply_markup=get_main_keyboard())
        await state.finish()
        return
        
    await state.update_data(okved=message.text)
    await message.answer("Дата регистрации ИП/ООО (в формате ДД.ММ.ГГГГ):", reply_markup=get_cancel_keyboard())
    await Questionnaire.next()

# Обработка даты регистрации
@dp.message_handler(state=Questionnaire.waiting_for_reg_date)
async def process_reg_date(message: types.Message, state: FSMContext):
    if message.text == '❌ Отмена':
        await message.answer("Заполнение анкеты отменено.", reply_markup=get_main_keyboard())
        await state.finish()
        return
        
    # Простая валидация даты
    try:
        datetime.strptime(message.text, '%d.%m.%Y')
        await state.update_data(reg_date=message.text)
        await message.answer("Есть ли другие счета на это же лицо? Укажите какие:", reply_markup=get_cancel_keyboard())
        await Questionnaire.next()
    except ValueError:
        await message.answer("Неверный формат даты. Пожалуйста, укажите дату в формате ДД.ММ.ГГГГ:")

# Обработка других счетов
@dp.message_handler(state=Questionnaire.waiting_for_other_accounts)
async def process_other_accounts(message: types.Message, state: FSMContext):
    if message.text == '❌ Отмена':
        await message.answer("Заполнение анкеты отменено.", reply_markup=get_main_keyboard())
        await state.finish()
        return
        
    await state.update_data(other_accounts=message.text)
    
    service_keyboard = ReplyKeyboardMarkup(resize_keyboard=True)
    service_keyboard.add(KeyboardButton("АТС"))
    service_keyboard.add(KeyboardButton("SIP телефония"))
    service_keyboard.add(KeyboardButton("Корпоративная связь"))
    service_keyboard.add(KeyboardButton("Нет"))
    service_keyboard.add(KeyboardButton("❌ Отмена"))
    
    await message.answer(
        "Также мы покупаем корпоративную связь:\n"
        "• АТС - виртуальные номера\n"
        "• SIP телефонию\n"
        "• Корпоративные тарифы связи\n\n"
        "Хотите предложить что-то из этого?",
        reply_markup=service_keyboard
    )
    await Questionnaire.next()

# Обработка дополнительных услуг
@dp.message_handler(state=Questionnaire.waiting_for_services)
async def process_services(message: types.Message, state: FSMContext):
    if message.text == '❌ Отмена':
        await message.answer("Заполнение анкеты отменено.", reply_markup=get_main_keyboard())
        await state.finish()
        return
        
    await state.update_data(services=message.text)
    
    # Получаем данные
    user_data = await state.get_data()
    
    # Формируем текст для подтверждения
    confirmation_text = (
        "Пожалуйста, проверьте ваши данные:\n\n"
        f"🏦 Банк: {user_data.get('bank', 'Не указано')}\n"
        f"🔗 СБП: {user_data.get('sbp', 'Не указано')}\n"
        f"📋 ОКВЭД: {user_data.get('okved', 'Не указано')}\n"
        f"📅 Дата регистрации: {user_data.get('reg_date', 'Не указано')}\n"
        f"💳 Другие счета: {user_data.get('other_accounts', 'Не указано')}\n"
        f"📞 Доп. услуги: {message.text}\n\n"
        "Всё верно?"
    )
    
    await message.answer(confirmation_text, 
                         reply_markup=make_row_keyboard(["✅ Да, отправить", "❌ Нет, отменить"]))
    await Questionnaire.next()

# Подтверждение анкеты
@dp.message_handler(state=Questionnaire.waiting_for_confirmation)
async def process_confirmation(message: types.Message, state: FSMContext):
    if message.text == '✅ Да, отправить':
        user_data = await state.get_data()
        
        # Сохраняем в базу данных
        save_questionnaire(user_data)
        
        # Формируем анкету для админов
        questionnaire_text = (
            "📋 Новая анкета:\n\n"
            f"👤 Пользователь: @{user_data.get('username', 'Не указано')}\n"
            f"🏦 Банк: {user_data.get('bank', 'Не указано')}\n"
            f"🔗 СБП: {user_data.get('sbp', 'Не указано')}\n"
            f"📋 ОКВЭД: {user_data.get('okved', 'Не указано')}\n"
            f"📅 Дата регистрации: {user_data.get('reg_date', 'Не указано')}\n"
            f"💳 Другие счета: {user_data.get('other_accounts', 'Не указано')}\n"
            f"📞 Доп. услуги: {user_data.get('services', 'Не указано')}\n"
            f"🆔 ID: {user_data.get('user_id', 'Не указано')}"
        )
        
        # Создаем инлайн-кнопки для админов
        admin_keyboard = InlineKeyboardMarkup()
        admin_keyboard.add(InlineKeyboardButton("📞 Связаться", 
                                              url=f"tg://user?id={user_data.get('user_id')}"))
        admin_keyboard.add(InlineKeyboardButton("✅ Обработано", callback_data=f"processed_{user_data.get('user_id')}"))
        
        # Отправляем всем админам
        for admin_id in ADMIN_CHAT_IDS:
            try:
                await bot.send_message(admin_id, questionnaire_text, reply_markup=admin_keyboard)
            except (BotBlocked, ChatNotFound):
                logging.error(f"Не удалось отправить сообщение админу {admin_id}")
        
        # Отправляем уведомление
        await send_notification_to_admins(f"Новая заявка от @{user_data.get('username', 'Не указано')}")
        
        await message.answer(
            "✅ Спасибо! Ваша заявка принята. Наш менеджер свяжется с вами в течение 15 минут для предложения лучших условий!",
            reply_markup=get_main_keyboard()
        )
    else:
        await message.answer("Заполнение анкеты отменено.", reply_markup=get_main_keyboard())
    
    await state.finish()

# Обработка кнопки "О нас"
@dp.message_handler(lambda message: message.text == 'ℹ️ О нас')
async def about_us(message: types.Message):
    await message.answer(
        "Мы — лидер на рынке аренды бизнес-счетов.\n\n"
        "📊 Наши преимущества:\n"
        "• Более 20000+ довольных клиентов\n"
        "• Выкуп счетов в 15+ банках\n"
        "• Выплаты до 100 000 руб. в сутки\n"
        "• Техническая поддержка 24/7\n\n"
        "📞 Свяжитесь с нами прямо сейчас и получите персональное предложение!",
        reply_markup=get_main_keyboard()
    )

# Обработка кнопки "Предложить корп. связь"
@dp.message_handler(lambda message: message.text == '📞 Предложить корп. связь')
async def offer_corp_connection(message: types.Message):
    await message.answer(
        "Отлично! Мы заинтересованы в покупке корпоративной связи.\n\n"
        "Для быстрой обработки вашего предложения, пожалуйста, ответьте на несколько вопросов.\n\n"
        "Какой тип связи вы предлагаете?",
        reply_markup=make_row_keyboard(["АТС", "SIP телефония", "Корпоративные тарифы", "❌ Отмена"])
    )
    # Здесь можно добавить состояние для сбора информации о корпоративной связи

# Обработка админ-команд
@dp.message_handler(lambda message: is_admin(message.from_user.id) and message.text == '📊 Статистика')
async def admin_stats(message: types.Message):
    total, new = get_stats()
    await message.answer(
        f"📊 Статистика заявок:\n\n"
        f"• Всего заявок: {total}\n"
        f"• Новых заявок: {new}\n"
        f"• Обработанных: {total - new}",
        reply_markup=get_admin_keyboard()
    )

@dp.message_handler(lambda message: is_admin(message.from_user.id) and message.text == '📋 Список заявок')
async def admin_list(message: types.Message):
    questionnaires = get_questionnaires(limit=5, status='new')
    if not questionnaires:
        await message.answer("Новых заявок нет.", reply_markup=get_admin_keyboard())
        return
        
    response = "📋 Последние 5 новых заявок:\n\n"
    for q in questionnaires:
        response += f"🆔 {q[0]}: {q[3]} ({q[9]})\n"
    
    response += "\nДля просмотра деталей используйте команду /view_<id>"
    await message.answer(response, reply_markup=get_admin_keyboard())

@dp.message_handler(lambda message: is_admin(message.from_user.id) and message.text == '📝 Шаблоны ответов')
async def admin_templates(message: types.Message):
    templates = get_templates()
    if not templates:
        await message.answer("Шаблонов ответов нет.", reply_markup=get_admin_keyboard())
        return
        
    response = "📝 Доступные шаблоны ответов:\n\n"
    for i, template in enumerate(templates, 1):
        response += f"{i}. {template[1]} - /template_{template[0]}\n"
    
    response += "\nДля использования шаблона отправьте команду /template_<id> <user_id>"
    await message.answer(response, reply_markup=get_admin_keyboard())

# Просмотр деталей заявки
@dp.message_handler(lambda message: is_admin(message.from_user.id) and message.text.startswith('/view_'))
async def admin_view(message: types.Message):
    try:
        q_id = int(message.text.split('_')[1])
        with db_connection() as conn:
            cursor = conn.cursor()
            cursor.execute('SELECT * FROM questionnaires WHERE id = ?', (q_id,))
            q = cursor.fetchone()
        
        if q:
            response = (
                f"📋 Заявка #{q[0]}\n\n"
                f"👤 Пользователь: @{q[2]}\n"
                f"🏦 Банк: {q[3]}\n"
                f"🔗 СБП: {q[4]}\n"
                f"📋 ОКВЭД: {q[5]}\n"
                f"📅 Дата регистрации: {q[6]}\n"
                f"💳 Другие счета: {q[7]}\n"
                f"📞 Доп. услуги: {q[8]}\n"
                f"📊 Статус: {q[9]}\n"
                f"🕐 Создана: {q[10]}\n"
                f"🆔 User ID: {q[1]}"
            )
            
            keyboard = InlineKeyboardMarkup()
            keyboard.add(InlineKeyboardButton("📞 Связаться", url=f"tg://user?id={q[1]}"))
            keyboard.add(InlineKeyboardButton("✅ Отметить обработанной", callback_data=f"processed_{q[0]}"))
            
            await message.answer(response, reply_markup=keyboard)
        else:
            await message.answer("Заявка не найдена.")
    except (IndexError, ValueError):
        await message.answer("Используйте формат: /view_<id>")

# Использование шаблона ответа
@dp.message_handler(lambda message: is_admin(message.from_user.id) and message.text.startswith('/template_'))
async def use_template(message: types.Message):
    try:
        parts = message.text.split('_')
        template_id = int(parts[1])
        user_id = int(parts[2]) if len(parts) > 2 else None
        
        with db_connection() as conn:
            cursor = conn.cursor()
            cursor.execute('SELECT * FROM response_templates WHERE id = ?', (template_id,))
            template = cursor.fetchone()
        
        if template:
            if user_id:
                try:
                    await bot.send_message(user_id, template[2])
                    await message.answer(f"Шаблон отправлен пользователю {user_id}")
                except Exception as e:
                    await message.answer(f"Ошибка отправки: {e}")
            else:
                await message.answer(f"Шаблон: {template[2]}\n\nДля отправки используйте /template_{template_id} <user_id>")
        else:
            await message.answer("Шаблон не найден.")
    except (IndexError, ValueError):
        await message.answer("Используйте формат: /template_<id> <user_id>")

# Обработка инлайн-кнопок
@dp.callback_query_handler(lambda c: c.data.startswith('processed_'))
async def process_callback(callback_query: types.CallbackQuery):
    q_id = int(callback_query.data.split('_')[1])
    update_questionnaire_status(q_id, 'processed')
    await bot.answer_callback_query(callback_query.id, "Заявка отмечена как обработанная")
    await bot.edit_message_reply_markup(
        chat_id=callback_query.message.chat.id,
        message_id=callback_query.message.message_id,
        reply_markup=None
    )

# Рассылка сообщений (только для админов)
@dp.message_handler(lambda message: is_admin(message.from_user.id) and message.text == '📢 Рассылка')
async def start_broadcast(message: types.Message):
    await message.answer("Введите сообщение для рассылки (поддерживается разметка Markdown):", 
                         reply_markup=get_cancel_keyboard())
    # Здесь нужно добавить состояние для рассылки

# Обработка всех остальных сообщений
@dp.message_handler()
async def echo(message: types.Message):
    if is_admin(message.from_user.id):
        await message.answer("Используйте панель администратора", reply_markup=get_admin_keyboard())
    else:
        await message.answer("Используйте кнопки меню для навигации", reply_markup=get_main_keyboard())

if __name__ == '__main__':
    executor.start_polling(dp, skip_updates=True)
