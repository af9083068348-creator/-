[main — копия.py](https://github.com/user-attachments/files/29441863/main.py)

import telebot
from telebot import types
import requests
from datetime import datetime, timedelta
import json
import re

BOT_TOKEN = "8992792423:AAFEWVioZbsXkhK3bVTVQg8-onNvhmybVD4"

CURRENCIES = {
    "RUB": {"name": "Российский рубль", "code": "RUB", "numeric": "643", "symbol": "₽"},
    "USD": {"name": "Доллар США", "code": "USD", "numeric": "840", "symbol": "$"},
    "EUR": {"name": "Евро", "code": "EUR", "numeric": "978", "symbol": "€"},
    "CNY": {"name": "Китайский юань", "code": "CNY", "numeric": "156", "symbol": "¥"},
    "TRY": {"name": "Турецкая лира", "code": "TRY", "numeric": "949", "symbol": "₺"},
    "JPY": {"name": "Японская иена", "code": "JPY", "numeric": "392", "symbol": "¥"},
    "KZT": {"name": "Казахстанский тенге", "code": "KZT", "numeric": "398", "symbol": "₸"},
    "BYN": {"name": "Белорусский рубль", "code": "BYN", "numeric": "933", "symbol": "Br"}
}

CURRENCY_NAMES = {
    "российский рубль": "RUB",
    "рубль": "RUB",
    "рубли": "RUB",
    "доллар": "USD",
    "доллары": "USD",
    "евро": "EUR",
    "юань": "CNY",
    "юани": "CNY",
    "лира": "TRY",
    "лиры": "TRY",
    "ена": "JPY",
    "ены": "JPY",
    "тенге": "KZT",
    "беларусский рубль": "BYN",
    "белорусский рубль": "BYN",
    "643": "RUB",
    "840": "USD",
    "978": "EUR",
    "156": "CNY",
    "949": "TRY",
    "392": "JPY",
    "398": "KZT",
    "933": "BYN"
}


API_URL = "https://api.exchangerate-api.com/v4/latest/"

bot = telebot.TeleBot(BOT_TOKEN)

user_data = {}


def get_currency_code(input_text):
    input_text = input_text.strip()

    if input_text.upper() in CURRENCIES:
        return input_text.upper()

    if input_text in CURRENCY_NAMES:
        return CURRENCY_NAMES[input_text]

    if input_text.lower() in CURRENCY_NAMES:
        return CURRENCY_NAMES[input_text.lower()]

    for name, code in CURRENCY_NAMES.items():
        if name in input_text.lower():
            return code

    return None


def get_currency_display(currency_code):
    if currency_code in CURRENCIES:
        data = CURRENCIES[currency_code]
        return f"{data['name']} (код: {currency_code}, цифровой код: {data['numeric']})"
    return currency_code


def get_currency_short_display(currency_code):
    if currency_code in CURRENCIES:
        data = CURRENCIES[currency_code]
        return f"{currency_code} ({data['numeric']})"
    return currency_code


def get_currency_list_display():
    result = []
    for code, data in CURRENCIES.items():
        result.append(f"• {data['name']} (код: {code}, цифровой код: {data['numeric']}) - {data['symbol']}")
    return "\n".join(result)


def get_exchange_rate_to_rub(from_currency):
    try:
        response = requests.get(f"{API_URL}{from_currency}")
        if response.status_code == 200:
            data = response.json()
            if 'RUB' in data['rates']:
                return data['rates']['RUB']
        return None
    except Exception as e:
        print(f"Ошибка получения курса: {e}")
        return None


def get_historical_rates_to_rub(currency, start_date, end_date):
    try:
        url = f"https://api.exchangerate.host/timeseries?start_date={start_date}&end_date={end_date}&base={currency}&symbols=RUB"

        response = requests.get(url)
        if response.status_code == 200:
            data = response.json()
            if 'rates' in data:
                rates = []
                for date, rate_data in data['rates'].items():
                    if 'RUB' in rate_data:
                        rates.append(rate_data['RUB'])
                return rates
        return None
    except Exception as e:
        print(f"Ошибка получения исторических данных: {e}")
        return None


def calculate_average(rates):
    if not rates:
        return 0
    return sum(rates) / len(rates)


def format_conversion_message(amount, from_currency, rub_amount, rate):
    from_display = get_currency_display(from_currency)
    rub_display = get_currency_display("RUB")

    return f"""
🔄 *Результат конвертации в рубли:*

💰 {amount:,.2f} {from_display}
➡️  {rub_amount:,.2f} {rub_display}

📊 Курс: 1 {get_currency_short_display(from_currency)} = {rate:.4f} {get_currency_short_display('RUB')}
"""


def format_average_message(currency, start_date, end_date, average, rates_count):
    currency_display = get_currency_display(currency)
    rub_display = get_currency_short_display("RUB")

    return f"""
📈 *Средний курс {currency_display} к рублю за период:*

📅 Период: {start_date} - {end_date}
📊 Среднее значение: {average:.4f} {rub_display}
📉 Количество дней: {rates_count}
"""

@bot.message_handler(commands=['start'])
def start_command(message):
    user_id = message.from_user.id
    user_data[user_id] = {'state': 'main'}

    markup = types.ReplyKeyboardMarkup(row_width=2, resize_keyboard=True)
    btn1 = types.KeyboardButton('💱 Конвертировать в рубли')
    btn2 = types.KeyboardButton('📊 Средний курс к рублю за период')
    btn3 = types.KeyboardButton('📋 Список валют')
    btn4 = types.KeyboardButton('ℹ️ Помощь')
    markup.add(btn1, btn2, btn3, btn4)

    welcome_text = f"""
👋 Привет, {message.from_user.first_name}!

Я бот для конвертации валют в *Российские рубли (RUB)*.

🌍 *Доступные валюты:*
{get_currency_list_display()}

Выберите действие:
"""
    bot.send_message(message.chat.id, welcome_text, reply_markup=markup, parse_mode='Markdown')


@bot.message_handler(commands=['help'])
def help_command(message):
    help_text = f"""
🤖 *Как пользоваться ботом:*

1️⃣ *Конвертация в рубли (RUB)*
   • Нажмите кнопку "💱 Конвертировать в рубли"
   • Введите сумму и валюту
   • Примеры:
     - `100 USD` - конвертировать 100 долларов в рубли
     - `50 Евро` - конвертировать 50 евро в рубли
     - `1000 398` - конвертировать 1000 тенге (цифровой код 398) в рубли
     - `200 156` - конвертировать 200 юаней (цифровой код 156) в рубли

2️⃣ *Средний курс к рублю за период*
   • Нажмите кнопку "📊 Средний курс к рублю за период"
   • Введите валюту, начальную и конечную дату
   • Пример: `USD 2024-01-01 2024-01-31`

3️⃣ *Список валют*
   • Нажмите кнопку "📋 Список валют"

🌍 *Доступные валюты:*
{get_currency_list_display()}

💡 *Подсказка:* Можно вводить:
• Код валюты (USD, EUR)
• Цифровой код (840, 978)
• Название на русском (Доллар, Евро)
"""
    bot.send_message(message.chat.id, help_text, parse_mode='Markdown')


@bot.message_handler(func=lambda message: message.text == 'ℹ️ Помощь')
def help_button(message):
    help_command(message)


@bot.message_handler(func=lambda message: message.text == '📋 Список валют')
def list_currencies(message):
    text = f"""
🌍 *Доступные валюты для конвертации в рубли (RUB):*

{get_currency_list_display()}

📝 *Примеры ввода:*
• `100 USD` или `100 840` или `100 Доллар`
• `50 EUR` или `50 978` или `50 Евро`  
• `1000 KZT` или `1000 398` или `1000 Тенге`
• `100 CNY` или `100 156` или `100 Юань`
• `100 TRY` или `100 949` или `100 Лира`
• `100 JPY` или `100 392` или `100 Йена`
"""
    bot.send_message(message.chat.id, text, parse_mode='Markdown')


@bot.message_handler(func=lambda message: message.text == '💱 Конвертировать в рубли')
def convert_currency_start(message):
    user_id = message.from_user.id
    user_data[user_id] = {'state': 'converting'}

    text = f"""
💱 *Конвертация в Российские рубли (RUB)*

Введите сумму и валюту в формате:
`[сумма] [валюта]`

📝 *Примеры:*
• `100 USD` - конвертировать 100 долларов в рубли
• `50 978` - конвертировать 50 евро (цифровой код 978) в рубли
• `1000 KZT` - конвертировать 1000 тенге в рубли
• `200 156` - конвертировать 200 юаней (цифровой код 156) в рубли

🌍 *Доступные валюты:*
{get_currency_list_display()}

💡 Можно вводить:
• Код (USD)
• Цифровой код (840)
• Название (Доллар)

Введите команду или нажмите /cancel для отмены
"""
    bot.send_message(message.chat.id, text, parse_mode='Markdown')


@bot.message_handler(func=lambda message: message.text == '📊 Средний курс к рублю за период')
def average_rate_start(message):
    user_id = message.from_user.id
    user_data[user_id] = {'state': 'averaging'}

    text = f"""
Средняя цена валют к рублю с 2020 по 2025 год
$ USD(840) 
    2020 = 72,32 рубля
    2021 = 73,67 рубля
    2022 = 68,35 рубля
    2023 = 85,81 рубля
    2024 = 92,66 рубля 
    2025 = 83,21 рубля
€ EUR(978)
    2020 = 82,72 рубля
    2021 = 87,16 рубля
    2022 = 71,90 рубля
    2023 = 92,64 рубля
    2024 = 100,28 рубля
    2025 = 94,05 рубля
¥ CNY(156)
    2020 = 10,51 рубля
    2021 = 11,42 рубля
    2022 = 10,07 рубля
    2023 = 11,90 рубля
    2024 = 12,86 рубля
    2025 = 11,53 рубля
₺ TRY(949)
    2020 = 10,29 рубля
    2021 = 8,51 рубля
    2022 = 4,23 рубля
    2023 = 3,63 рубля
    2024 = 2,85 рубля
    2025 = 2,12 рубля
¥ JPY(392)
    2020 = 0,68 рубля
    2021 = 0,67 рубля
    2022 = 0,53 рубля
    2023 = 0,61 рубля 
    2024 = 0,61 рубля
    2025 = 0,57 рубля
₸ KZT(398)
    2020 = 0,17 рубля
    2021 = 0,17 рубля
    2022 = 0,15 рубля
    2023 = 0,19 рубля
    2024 = 0,20 рубля
    2025 = 0,16 рубля
Br BYN(933)
    2020 = 29,57 рубля
    2021 = 28,95 рубля
    2022 = 25,84 рубля
    2023 = 28,52 рубля
    2024 = 28,51 рубля
    2025 = 26,19 рубля
"""
    bot.send_message(message.chat.id, text, parse_mode='Markdown')


@bot.message_handler(commands=['cancel'])
def cancel_command(message):
    user_id = message.from_user.id
    if user_id in user_data:
        del user_data[user_id]
    bot.send_message(message.chat.id, "❌ Операция отменена. Используйте /start для начала работы.")


@bot.message_handler(func=lambda message: True)
def handle_messages(message):
    user_id = message.from_user.id
    text = message.text.strip()

    state = user_data.get(user_id, {}).get('state', 'main')

    if state == 'converting':
        handle_conversion(message, text)

    elif state == 'averaging':
        handle_average_calculation(message, text)

    else:
        bot.send_message(
            message.chat.id,
            "❌ Неизвестная команда. Используйте кнопки меню или /start.",
            reply_markup=types.ReplyKeyboardRemove()
        )
        start_command(message)


def handle_conversion(message, text):
    try:
        parts = text.split()
        if len(parts) < 2:
            bot.send_message(
                message.chat.id,
                f"❌ Неверный формат. Используйте: `[сумма] [валюта]`\nПримеры: `100 USD`, `100 840`, `100 Доллар`\n\n🌍 Доступные валюты:\n{get_currency_list_display()}",
                parse_mode='Markdown'
            )
            return

        amount = float(parts[0])
        currency_input = ' '.join(parts[1:])

        currency_code = get_currency_code(currency_input)

        if currency_code is None:
            bot.send_message(
                message.chat.id,
                f"❌ Валюта '{currency_input}' не поддерживается.\n\n🌍 Доступные валюты:\n{get_currency_list_display()}",
                parse_mode='Markdown'
            )
            return

        if currency_code == "RUB":
            bot.send_message(
                message.chat.id,
                "❌ Это уже рубли (RUB)! Конвертация не требуется."
            )
            return

        rate = get_exchange_rate_to_rub(currency_code)

        if rate is None:
            bot.send_message(message.chat.id, "❌ Не удалось получить курс. Попробуйте позже.")
            return

        rub_amount = amount * rate

        response_text = format_conversion_message(amount, currency_code, rub_amount, rate)
        bot.send_message(message.chat.id, response_text, parse_mode='Markdown')

        user_data[message.from_user.id] = {'state': 'main'}

    except ValueError:
        bot.send_message(
            message.chat.id,
            f"❌ Сумма должна быть числом. Используйте формат: `100 USD`, `100 840` или `100 Доллар`",
            parse_mode='Markdown'
        )
    except Exception as e:
        bot.send_message(message.chat.id, f"❌ Произошла ошибка: {str(e)}")


def handle_average_calculation(message, text):
    """
    Обрабатывает расчет среднего курса к рублю за период
    """
    try:
        parts = text.split()
        if len(parts) < 3:
            bot.send_message(
                message.chat.id,
                f"❌ Неверный формат. Используйте: `[валюта] [начальная_дата] [конечная_дата]`\nПример: `USD 2024-01-01 2024-01-31` или `840 2024-01-01 2024-01-31`\n\n🌍 Доступные валюты:\n{get_currency_list_display()}",
                parse_mode='Markdown'
            )
            return

        currency_input = ' '.join(parts[:-2])
        start_date = parts[-2]
        end_date = parts[-1]

        currency_code = get_currency_code(currency_input)

        if currency_code is None:
            bot.send_message(
                message.chat.id,
                f"❌ Валюта '{currency_input}' не поддерживается.\n\n🌍 Доступные валюты:\n{get_currency_list_display()}",
                parse_mode='Markdown'
            )
            return

        try:
            datetime.strptime(start_date, "%Y-%m-%d")
            datetime.strptime(end_date, "%Y-%m-%d")
        except ValueError:
            bot.send_message(message.chat.id, "❌ Неверный формат даты. Используйте ГГГГ-ММ-ДД (например: 2024-01-01)")
            return

        if start_date > end_date:
            bot.send_message(message.chat.id, "❌ Начальная дата не может быть позже конечной.")
            return

        start = datetime.strptime(start_date, "%Y-%m-%d")
        end = datetime.strptime(end_date, "%Y-%m-%d")
        days_diff = (end - start).days
        if days_diff > 365:
            bot.send_message(
                message.chat.id,
                "⚠️ Период не должен превышать 365 дней для получения точных данных."
            )
            return

        bot.send_message(message.chat.id, "⏳ Получаю исторические данные... Это может занять несколько секунд.")

        rates = get_historical_rates_to_rub(currency_code, start_date, end_date)

        if rates is None or len(rates) == 0:
            bot.send_message(
                message.chat.id,
                "❌ Не удалось получить исторические данные. Попробуйте изменить период или валюту."
            )
            return

        average = calculate_average(rates)

        response_text = format_average_message(currency_code, start_date, end_date, average, len(rates))
        bot.send_message(message.chat.id, response_text, parse_mode='Markdown')

        user_data[message.from_user.id] = {'state': 'main'}

    except Exception as e:
        bot.send_message(message.chat.id, f"❌ Произошла ошибка: {str(e)}")


if __name__ == '__main__':
    print("🤖 Бот для конвертации валют в рубли запущен!")
    print(f"🌍 Поддерживаемые валюты:")
    for code, data in CURRENCIES.items():
        print(f"  • {data['name']} (код: {code}, цифровой код: {data['numeric']})")
    print("Нажмите Ctrl+C для остановки.")
    try:
        bot.infinity_polling()
    except KeyboardInterrupt:
        print("\n👋 Бот остановлен.")
