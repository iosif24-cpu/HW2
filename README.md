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
        "<a href='https://i.yapx.ru/eNX9d.png'>👋</a><b>Добро пожаловать!</b>\n\n"
        "<b>Выберите нужный раздел:</b>",
        reply_markup=main_keyboard(),parse_mode='HTML'

    )


# ==========================================================
# КНОПКА "ЗАЯВКА НА МЕДИА"
# ==========================================================

@dp.message(F.text == "🔰 Заявка на медиа")
async def media_button(message: Message, state: FSMContext):

    # Если человек уже находится в чате
    if message.from_user.id in active_chats:
        await message.answer(
            "❌ <b>Сначала завершите текущий чат с оператором.</b>"
        )
        return

    await state.set_state(Form.waiting_media)

    await message.answer(
        "🔰<b>Заявка на медиа</b>\n\n"
        "<b>Напишите следующим сообщением вашу заявку.</b>\n\n"
        "<b>1.Ссылка на ваш канал.</b>\n"
        "<b>2.Количество подписчиков на данный момент.</b>\n"
        "<b>3.Среднее количество просмотров под роликами/стримами.</b>\n"
        "<b>4.Никнейм в игре.</b>\n"
        "<b>5.Настоящее имя.</b>\n"
        "<b>6.В каком формате планируете снимать? (стримы, шортсы или длинные ролики).</b>\n"
        "<b>7.Готовы ли вы оставить ссылку на сервер в описании/закрепе?</b>\n"
        "<b>8.Ссылка на ваш VK / Telegram для связи.</b>\n\n"
        "<b>После отправки она будет передана оператору.</b>",
        parse_mode = 'HTML'
    )


# ==========================================================
# ПОЛУЧЕНИЕ ЗАЯВКИ НА МЕДИА
# ==========================================================

@dp.message(Form.waiting_media)
async def receive_media(message: Message, state: FSMContext):
    client_id = message.from_user.id

    username = message.from_user.username

    if username:
        user_name = f"@{username}"
    else:
        user_name = message.from_user.full_name

    text = message.text

    if not text:
        await message.answer(
            "<b>❌ Пожалуйста, отправьте заявку обычным текстовым сообщением.</b>"
        )
        return

    operator_message = (
        "📺 НОВАЯ ЗАЯВКА НА МЕДИА\n"
        "━━━━━━━━━━━━━━━━━━\n\n"
        f"👤 Пользователь: {user_name}\n"
        f"🆔 Telegram ID: {client_id}\n\n"
        f"📝 Заявка:\n{text}"
    )

    await bot.send_message(
        chat_id=MEDIA_OPERATOR_ID,
        text=operator_message,
        reply_markup=media_keyboard(client_id)
    )

    await message.answer(
        "<b>✅ Ваша заявка отправлена оператору.\n\n</b>"
        "<b>Ожидайте решения.</b>"
    )

    await state.clear()


# ==========================================================
# ПРИНЯТИЕ МЕДИА-ЗАЯВКИ
# ==========================================================

@dp.callback_query(F.data.startswith("media_accept:"))
async def accept_media(callback: CallbackQuery):
    # Проверяем оператора
    if callback.from_user.id != MEDIA_OPERATOR_ID:
        await callback.answer(
            "❌ У вас нет доступа к этой кнопке.",
            show_alert=True
        )
        return

    client_id = int(callback.data.split(":")[1])

    try:
        await bot.send_message(
            chat_id=client_id,
            text=(
                "<b>✅ Ваша заявка на медиа принята!\n\n</b>"
                "<b>Оператор рассмотрел вашу заявку.</b>"
            )
        )

        await callback.message.edit_reply_markup(
            reply_markup=None
        )

        await callback.answer(
            "Заявка принята."
        )

    except Exception as error:

        logging.error(
            f"Ошибка при принятии заявки: {error}"
        )

        await callback.answer(
            "❌ Не удалось отправить сообщение пользователю.",
            show_alert=True
        )


# ==========================================================
# ОТКЛОНЕНИЕ МЕДИА-ЗАЯВКИ
# ==========================================================

@dp.callback_query(F.data.startswith("media_reject:"))
async def reject_media(callback: CallbackQuery):
    if callback.from_user.id != MEDIA_OPERATOR_ID:
        await callback.answer(
            "❌ У вас нет доступа к этой кнопке.",
            show_alert=True
        )
        return

    client_id = int(callback.data.split(":")[1])

    try:

        await bot.send_message(
            chat_id=client_id,
            text=(
                "<b>❌ Ваша заявка на медиа отклонена.</b>"
            )
        )

        await callback.message.edit_reply_markup(
            reply_markup=None
        )

        await callback.answer(
            "Заявка отклонена."
        )

    except Exception as error:

        logging.error(
            f"Ошибка при отклонении заявки: {error}"
        )

        await callback.answer(
            "❌ Не удалось отправить сообщение пользователю.",
            show_alert=True
        )


# ==========================================================
# КНОПКА "ТЕХ ПОДДЕРЖКА"
# ==========================================================

@dp.message(F.text == "🛠 Тех поддержка")
async def support_button(message: Message, state: FSMContext):
    client_id = message.from_user.id

    if client_id in active_chats:
        await message.answer(
            "<b>❌ У вас уже есть активный чат с оператором.</b>"
        )
        return

    await state.set_state(Form.waiting_support)

    await message.answer(
        "<b>🛠 Техническая поддержка\n\n</b>"
        "Опишите вашу проблему следующим сообщением.\n\n"
        "<b>1.Ваш игровой никнейм.</b>\n"
        "<b>2.Когда был найден баг/произошла проблема (примерное время).</b>\n"
        "<b>3.Суть бага/проблемы.</b>\n\n"
        "<b>Ваше сообщение будет передано оператору.</b>"
    )


# ==========================================================
# ПОЛУЧЕНИЕ ПЕРВОГО СООБЩЕНИЯ В ТЕХПОДДЕРЖКУ
# ==========================================================

@dp.message(Form.waiting_support)
async def receive_support(message: Message, state: FSMContext):
    client_id = message.from_user.id

    username = message.from_user.username

    if username:
        user_name = f"@{username}"
    else:
        user_name = message.from_user.full_name

    text = message.text

    if not text:
        await message.answer(
            "❌ Пожалуйста, опишите проблему обычным текстовым сообщением."
        )
        return

    operator_message = (
        "🛠 НОВОЕ ОБРАЩЕНИЕ В ТЕХПОДДЕРЖКУ\n"
        "━━━━━━━━━━━━━━━━━━\n\n"
        f"👤 Пользователь: {user_name}\n"
        f"🆔 Telegram ID: {client_id}\n\n"
        f"📝 Сообщение клиента:\n{text}"
    )

    await bot.send_message(
        chat_id=SUPPORT_OPERATOR_ID,
        text=operator_message,
        reply_markup=support_keyboard(client_id)
    )

    await message.answer(
        "✅ Ваше сообщение отправлено оператору.\n\n"
        "Ожидайте подключения."
    )

    await state.clear()


# ==========================================================
# ОПЕРАТОР НАЖИМАЕТ "НАЧАТЬ ЧАТ"
# ==========================================================

@dp.callback_query(F.data.startswith("support_start:"))
async def start_support_chat(callback: CallbackQuery):
    operator_id = callback.from_user.id

    # Только оператор поддержки
    if operator_id != SUPPORT_OPERATOR_ID:
        await callback.answer(
            "❌ У вас нет доступа к этой кнопке.",
            show_alert=True
        )
        return

    client_id = int(callback.data.split(":")[1])

    # Проверяем, занят ли оператор
    if operator_id in operator_chats:
        await callback.answer(
            "❌ У вас уже есть активный чат.",
            show_alert=True
        )
        return

    # Проверяем, занят ли клиент
    if client_id in active_chats:
        await callback.answer(
            "❌ Этот клиент уже общается с оператором.",
            show_alert=True
        )
        return

    # Создаём чат
    active_chats[client_id] = operator_id
    operator_chats[operator_id] = client_id

    # Сообщение клиенту
    await bot.send_message(
        chat_id=client_id,
        text=(
            "🟢 Оператор подключился к чату.\n\n"
            "Теперь вы можете отправлять сообщения."
        )
    )

    # Убираем кнопку "Начать чат"
    await callback.message.edit_reply_markup(
        reply_markup=None
    )

    # Сообщение оператору
    await bot.send_message(
        chat_id=operator_id,
        text=(
            f"🟢 Чат с клиентом {client_id} начат.\n\n"
            "Все ваши сообщения будут отправляться клиенту."
        ),
        reply_markup=operator_keyboard(client_id)
    )

    await callback.answer(
        "Чат начат."
    )


# ==========================================================
# ЗАВЕРШЕНИЕ ЧАТА
# ==========================================================

@dp.message(F.text.startswith("❌ Завершить чат с "))
async def finish_chat(message: Message):
    operator_id = message.from_user.id

    # Проверяем, является ли пользователь оператором
    if operator_id != SUPPORT_OPERATOR_ID:
        await message.answer(
            "❌ У вас нет активного чата."
        )
        return

    # Проверяем, есть ли активный чат
    if operator_id not in operator_chats:
        await message.answer(
            "❌ У вас нет активного чата."
        )
        return
    client_id = operator_chats[operator_id]

    # Сообщение клиенту
    try:

        await bot.send_message(
            chat_id=client_id,
            text=(
                "🔴 Чат с оператором завершён.\n\n"
                "Если вам снова понадобится помощь, "
                "нажмите «🛠 Тех поддержка»."
            )
        )

    except Exception as error:

        logging.error(
            f"Ошибка отправки сообщения клиенту: {error}"
        )

    # Удаляем чат
    active_chats.pop(client_id, None)
    operator_chats.pop(operator_id, None)

    # Возвращаем оператору главное меню
    await message.answer(
        "✅ Чат успешно завершён.",
        reply_markup=main_keyboard()
    )


# ==========================================================
# ОБЩИЙ ОБРАБОТЧИК СООБЩЕНИЙ
# ==========================================================

@dp.message()
async def chat_messages(message: Message, state: FSMContext):
    user_id = message.from_user.id

    # ------------------------------------------------------
    # ЕСЛИ ЭТО ОПЕРАТОР
    # ------------------------------------------------------

    if user_id == SUPPORT_OPERATOR_ID:

        # Если оператор ведёт чат
        if user_id in operator_chats:

            client_id = operator_chats[user_id]

            # Игнорируем кнопку завершения
            if (
                    message.text
                    and message.text.startswith("❌ Завершить чат с ")
            ):
                return

            # Только текст
            if message.text:

                await bot.send_message(
                    chat_id=client_id,
                    text=(
                        f"👨‍💻 Оператор:\n\n"
                        f"{message.text}"
                    )
                )

            else:

                await message.answer(
                    "⚠️ Сейчас бот поддерживает в чате только текстовые сообщения."
                )

        return

    # ------------------------------------------------------
    # ЕСЛИ ЭТО КЛИЕНТ
    # ------------------------------------------------------

    if user_id in active_chats:

        operator_id = active_chats[user_id]

        # Только текст
        if message.text:

            await bot.send_message(
                chat_id=operator_id,
                text=(
                    f"👤 Клиент {user_id}:\n\n"
                    f"{message.text}"
                )
            )

        else:

            await message.answer(
                "⚠️ Сейчас в чате поддерживаются только текстовые сообщения."
            )

        return


# ==========================================================
# ЗАПУСК БОТА
# ==========================================================

async def main():
    print("=================================")
    print("🤖 Telegram бот запущен")
    print("=================================")

    try:

        await dp.start_polling(bot)

    finally:

        await bot.session.close()


# ==========================================================
# MAIN
# ==========================================================

if __name__ == "__main__":
    asyncio.run(main())
