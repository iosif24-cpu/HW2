import asyncio
import logging

from aiogram import Bot, Dispatcher, F
from aiogram.filters import CommandStart
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.types import (
    Message,
    CallbackQuery,
    InlineKeyboardMarkup,
    InlineKeyboardButton,
    ReplyKeyboardMarkup,
    KeyboardButton,
)

# ==========================================================
# НАСТРОЙКИ
# ==========================================================

BOT_TOKEN = "8994715532:AAGte-MAminPkUyABNNl2sN7xVOPEk745vc"

# Оператор, который принимает заявки на медиа
MEDIA_OPERATOR_ID = 8173491400

# Оператор технической поддержки
SUPPORT_OPERATOR_ID = 8173491400

# ==========================================================
# ЗАПУСК
# ==========================================================

logging.basicConfig(level=logging.INFO)

bot = Bot(token=BOT_TOKEN)
dp = Dispatcher()

# ==========================================================
# СОСТОЯНИЯ ПОЛЬЗОВАТЕЛЕЙ
# ==========================================================

class Form(StatesGroup):
    waiting_media = State()
    waiting_support = State()

# ==========================================================
# АКТИВНЫЕ ЧАТЫ
# ==========================================================

# client_id -> operator_id
active_chats = {}

# operator_id -> client_id
operator_chats = {}

# ==========================================================
# ГЛАВНАЯ КЛАВИАТУРА
# ==========================================================

def main_keyboard():
    return ReplyKeyboardMarkup(
        keyboard=[
            [
                KeyboardButton(text="🔰 Заявка на медиа")
            ],
            [
                KeyboardButton(text="🛠 Тех поддержка")
            ]
        ],
        resize_keyboard=True
    )

# ==========================================================
# КНОПКИ МЕДИА
# ==========================================================

def media_keyboard(client_id: int):
    return InlineKeyboardMarkup(
        inline_keyboard=[
            [
                InlineKeyboardButton(
                    text="✅ Принять",
                    callback_data=f"media_accept:{client_id}"
                ),
                InlineKeyboardButton(
                    text="❌ Отклонить",
                    callback_data=f"media_reject:{client_id}"
                )
            ]
        ]
    )

# ==========================================================
# КНОПКА НАЧАЛА ТЕХПОДДЕРЖКИ
# ==========================================================

def support_keyboard(client_id: int):
    return InlineKeyboardMarkup(
        inline_keyboard=[
            [
                InlineKeyboardButton(
                    text="💬 Начать чат",
                    callback_data=f"support_start:{client_id}"
                )
            ]
        ]
    )

# ==========================================================
# КЛАВИАТУРА ОПЕРАТОРА
# ==========================================================

def operator_keyboard(client_id: int):
    return ReplyKeyboardMarkup(
        keyboard=[
            [
                KeyboardButton(
                    text=f"❌ Завершить чат с {client_id}"
                )
            ]
        ],
        resize_keyboard=True
    )

# ==========================================================
# START
# ==========================================================

@dp.message(CommandStart())
async def command_start(message: Message, state: FSMContext):
    await state.clear()
    await message.answer(
        "👋 Добро пожаловать!\n\nВыберите нужный раздел:",
        reply_markup=main_keyboard(),
        parse_mode='HTML'
    )

# ==========================================================
# КНОПКА "ЗАЯВКА НА МЕДИА"
# ==========================================================

@dp.message(F.text == "🔰 Заявка на медиа")
async def media_button(message: Message, state: FSMContext):
    if message.from_user.id in active_chats:
        await message.answer("❌ У вас уже есть активный чат или заявка.")
        return

    await state.set_state(Form.waiting_media)
    await message.answer(
        "📤 Пожалуйста, отправьте материалы (текст, фото или видео) для заявки на медиа:",
        reply_markup=ReplyKeyboardMarkup(
            keyboard=[[KeyboardButton(text="🔙 Отмена")]],
            resize_keyboard=True
        )
    )

@dp.message(F.text == "🔙 Отмена")
async def cancel_handler(message: Message, state: FSMContext):
    await state.clear()
    await message.answer("Действие отменено.", reply_markup=main_keyboard())

@dp.message(Form.waiting_media)
async def process_media_request(message: Message, state: FSMContext):
    client_id = message.from_user.id
    user_name = f"@{message.from_user.username}" if message.from_user.username else f"ID: {client_id}"
    text = message.text or message.caption or "Без описания"
    
    await state.clear()
    
    operator_message = (
        "📺 НОВАЯ ЗАЯВКА НА МЕДИА\n"
        "━━━━━━━━━━━━━━━━━━\n\n"
        f"👤 Пользователь: {user_name}\n"
        f"🆔 Telegram ID: {client_id}\n\n"
        f"📝 Заявка:\n{text}"
    )
    
    # Если пользователь прикрепил фото или видео, отправляем их с единым текстом
    if message.photo:
        await bot.send_photo(
            chat_id=MEDIA_OPERATOR_ID,
            photo=message.photo[-1].file_id,
            caption=operator_message,
            reply_markup=media_keyboard(client_id)
        )
    elif message.video:
        await bot.send_video(
            chat_id=MEDIA_OPERATOR_ID,
            video=message.video.file_id,
            caption=operator_message,
            reply_markup=media_keyboard(client_id)
        )
    else:
        await bot.send_message(
            chat_id=MEDIA_OPERATOR_ID,
            text=operator_message,
            reply_markup=media_keyboard(client_id)
        )
    
    await message.answer("✅ Ваша заявка отправлена оператору!", reply_markup=main_keyboard())

# ==========================================================
# КНОПКА "ТЕХ ПОДДЕРЖКА"
# ==========================================================

@dp.message(F.text == "🛠 Тех поддержка")
async def support_button(message: Message, state: FSMContext):
    if message.from_user.id in active_chats:
        await message.answer("❌ У вас уже открыт активный чат.")
        return

    await state.set_state(Form.waiting_support)
    await message.answer(
        "🛠 Опишите вашу проблему или вопрос, и мы передадим его в техподдержку:",
        reply_markup=ReplyKeyboardMarkup(
            keyboard=[[KeyboardButton(text="🔙 Отмена")]],
            resize_keyboard=True
        )
    )

@dp.message(Form.waiting_support)
async def process_support_request(message: Message, state: FSMContext):
    client_id = message.from_user.id
    user_name = f"@{message.from_user.username}" if message.from_user.username else f"ID: {client_id}"
    text = message.text or message.caption or "Без описания"
    
    await state.clear()

    operator_message = (
        "🛠 ЗАПРОС В ТЕХПОДДЕРЖКУ\n"
        "━━━━━━━━━━━━━━━━━━\n\n"
        f"👤 Пользователь: {user_name}\n"
        f"🆔 Telegram ID: {client_id}\n\n"
        f"📝 Вопрос:\n{text}"
    )

    if message.photo:
        await bot.send_photo(
            chat_id=SUPPORT_OPERATOR_ID,
            photo=message.photo[-1].file_id,
            caption=operator_message,
            reply_markup=support_keyboard(client_id)
        )
    elif message.video:
        await bot.send_video(
            chat_id=SUPPORT_OPERATOR_ID,
            video=message.video.file_id,
            caption=operator_message,
            reply_markup=support_keyboard(client_id)
        )
    else:
        await bot.send_message(
            chat_id=SUPPORT_OPERATOR_ID,
            text=operator_message,
            reply_markup=support_keyboard(client_id)
        )
    
    await message.answer("✅ Ваше сообщение отправлено в техподдержку. Ожидайте ответа.", reply_markup=main_keyboard())

# ==========================================================
# ОБРАБОТКА CALLBACK
# ==========================================================

@dp.callback_query(F.data.startswith("media_accept:"))
async def media_accept(callback: CallbackQuery):
    client_id = int(callback.data.split(":")[1])
    # Проверяем, было ли это фото/видео (у них caption вместо text)
    if callback.message.caption:
        await callback.message.edit_caption(caption=callback.message.caption + "\n\n✅ <b>Принято</b>", parse_mode="HTML")
    else:
        await callback.message.edit_text(callback.message.text + "\n\n✅ <b>Принято</b>", parse_mode="HTML")
        
    await bot.send_message(client_id, "✅ Ваша заявка на медиа была принята оператором!")
    await callback.answer()

@dp.callback_query(F.data.startswith("media_reject:"))
async def media_reject(callback: CallbackQuery):
    client_id = int(callback.data.split(":")[1])
    if callback.message.caption:
        await callback.message.edit_caption(caption=callback.message.caption + "\n\n❌ <b>Отклонено</b>", parse_mode="HTML")
    else:
        await callback.message.edit_text(callback.message.text + "\n\n❌ <b>Отклонено</b>", parse_mode="HTML")
        
    await bot.send_message(client_id, "❌ К сожалению, ваша заявка на медиа была отклонена.")
    await callback.answer()

@dp.callback_query(F.data.startswith("support_start:"))
async def support_start_chat(callback: CallbackQuery):
    client_id = int(callback.data.split(":")[1])
    operator_id = callback.from_user.id

    active_chats[client_id] = operator_id
    operator_chats[operator_id] = client_id

    if callback.message.caption:
        await callback.message.edit_caption(caption=callback.message.caption + "\n\n💬 <b>Чат начат</b>", parse_mode="HTML")
    else:
        await callback.message.edit_text(callback.message.text + "\n\n💬 <b>Чат начат</b>", parse_mode="HTML")
    
    await bot.send_message(
        client_id,
        "💬 Оператор подключился к чату! Теперь вы можете писать сюда сообщения.",
        reply_markup=ReplyKeyboardMarkup(
            keyboard=[[KeyboardButton(text="❌ Завершить чат")]],
            resize_keyboard=True
        )
    )
    
    await bot.send_message(
        operator_id,
        f"💬 Чат с пользователем {client_id} начат.",
        reply_markup=operator_keyboard(client_id)
    )
    await callback.answer()

# ==========================================================
# АКТИВНЫЙ ДИАЛОГ МЕЖДУ ПОЛЬЗОВАТЕЛЕМ И ОПЕРАТОРОМ
# ==========================================================

@dp.message(F.text == "❌ Завершить чат")
async def user_close_chat(message: Message):
    client_id = message.from_user.id
    if client_id in active_chats:
        operator_id = active_chats[client_id]
        
        del active_chats[client_id]
        if operator_id in operator_chats:
            del operator_chats[operator_id]
            
        await message.answer("❌ Чат завершен.", reply_markup=main_keyboard())
        await bot.send_message(operator_id, f"❌ Пользователь {client_id} завершил чат.", reply_markup=main_keyboard())
    else:
        await message.answer("У вас нет активных чатов.", reply_markup=main_keyboard())

@dp.message(F.text.startswith("❌ Завершить чат с "))
async def operator_close_chat(message: Message):
    operator_id = message.from_user.id
    if operator_id in operator_chats:
        client_id = operator_chats[operator_id]
        
        del operator_chats[operator_id]
        if client_id in active_chats:
            del active_chats[client_id]
            
        await message.answer(f"❌ Чат с пользователем {client_id} завершен.", reply_markup=main_keyboard())
        await bot.send_message(client_id, "❌ Оператор завершил чат.", reply_markup=main_keyboard())
    else:
        await message.answer("У вас нет активных чатов с пользователями.", reply_markup=main_keyboard())

@dp.message()
async def chat_message_forwarder(message: Message):
    user_id = message.from_user.id
    
    if user_id in active_chats:
        operator_id = active_chats[user_id]
        await message.copy_to(operator_id)
    elif user_id in operator_chats:
        client_id = operator_chats[user_id]
        await message.copy_to(client_id)

# ==========================================================
# ЗАПУСК БОТА
# ==========================================================

async def main():
    await bot.delete_webhook(drop_pending_updates=True)
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
