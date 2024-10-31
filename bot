from telegram import Update, ReplyKeyboardMarkup, KeyboardButton
from telegram.ext import Updater, CommandHandler, MessageHandler, Filters, CallbackContext, ConversationHandler
import logging
import sqlite3
import datetime
from apscheduler.schedulers.background import BackgroundScheduler

# Настройки логгера
logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
                    level=logging.INFO)
logger = logging.getLogger(__name__)

# Состояние диалога
NAME, SURNAME, START_WORK, END_WORK, STORE_VISIT, PROFIT, PURCHASE, COMMISSION, ADDITIONAL_WORK = range(9)

# Функция для получения соединения с базой данных
def get_db_connection():
    return sqlite3.connect('employees.db')

# Команда /start
def start(update: Update, context: CallbackContext):
    user = update.message.from_user
    logger.info("Пользователь %s начал разговор.", user.first_name)

    # Запрашиваем имя и фамилию
    update.message.reply_text('Здравствуйте! Пожалуйста, введите Ваше имя и фамилию через пробел.')

    return NAME

# Обработка имени и фамилии
def handle_name(update: Update, context: CallbackContext):
    user = update.message.from_user
    text = update.message.text.split()

    if len(text) != 2:
        update.message.reply_text('Пожалуйста, введите Ваше имя и фамилию через пробел.')
        return NAME

    name = text[0]
    surname = text[1]

    db_conn = get_db_connection()
    cursor = db_conn.cursor()

    # Проверяем наличие сотрудника в базе данных
    cursor.execute('SELECT * FROM employees WHERE name=? AND surname=?', (name, surname))
    row = cursor.fetchone()

    if row is None:
        # Если нет, добавляем нового сотрудника
        cursor.execute('INSERT INTO employees (name, surname) VALUES (?, ?)', (name, surname))
        db_conn.commit()

        # Получаем ID добавленного сотрудника
        employee_id = cursor.lastrowid
        context.user_data['employee_id'] = employee_id

        update.message.reply_text(f'Добро пожаловать, {name} {surname}!')
    else:
        # Если сотрудник уже зарегистрирован
        employee_id = row[0]
        context.user_data['employee_id'] = employee_id

        update.message.reply_text(f'{name} {surname}, Вы уже зарегистрированы.')

    db_conn.close()

    # Переход к началу рабочего дня
    return start_work(update, context)

# Начало рабочего дня
def start_work(update: Update, context: CallbackContext):
    keyboard = [[KeyboardButton('Начать рабочий день')]]
    reply_markup = ReplyKeyboardMarkup(keyboard, one_time_keyboard=True)

    update.message.reply_text('Нажмите кнопку, чтобы отметить начало рабочего дня.', reply_markup=reply_markup)

    return START_WORK

# Конец рабочего дня
def end_work(update: Update, context: CallbackContext):
    keyboard = [[KeyboardButton('Завершить рабочий день')]]
    reply_markup = ReplyKeyboardMarkup(keyboard, one_time_keyboard=True)

    update.message.reply_text('Нажмите кнопку, чтобы отметить окончание рабочего дня.', reply_markup=reply_markup)

    return END_WORK

# Посещение магазина
def store_visit(update: Update, context: CallbackContext):
    update.message.reply_text('Введите название магазина, время нахождения и выполненную работу через запятую.')

    return STORE_VISIT

# Отчет по прибыли
def report_profit(update: Update, context: CallbackContext):
    update.message.reply_text('Укажите прибыль в процентах.')

    return PROFIT

# Отчет по закупке
def report_purchase(update: Update, context: CallbackContext):
    update.message.reply_text('Укажите закупку в процентах.')

    return PURCHASE

# Отчет по комиссии
def report_commission(update: Update, context: CallbackContext):
    update.message.reply_text('Укажите комиссию в процентах.')

    return COMMISSION

# Дополнительная работа
def additional_work(update: Update, context: CallbackContext):
    update.message.reply_text('Опишите выполненную дополнительную работу.')

    return ADDITIONAL_WORK

# Завершение отчета
def finish_report(update: Update, context: CallbackContext):
    user = update.message.from_user
    logger.info("Пользователь %s завершил отчет.", user.first_name)

    # Собираем все данные из контекста
    employee_id = context.user_data['employee_id']
    start_time = context.user_data['start_time']
    end_time = context.user_data['end_time']
    profit = context.user_data['profit']
    purchase = context.user_data['purchase']
    commission = context.user_data['commission']
    additional_work = context.user_data['additional_work']

    # Формируем отчет
    report = f'Отчёт от {update.message.from_user.full_name}:\n' \
             f'- Начало рабочего дня: {start_time}\n' \
             f'- Окончание рабочего дня: {end_time}\n' \
             f'- Прибыль: {profit}%\n' \
             f'- Закупка: {purchase}%\n' \
             f'- Комиссия: {commission}%\n' \
             f'- Дополнительная работа: {additional_work}'

    # Отправляем отчет в группу
    group_chat_id = -1001726681436  # Замените на ID вашей группы
    context.bot.send_message(group_chat_id, report)

    update.message.reply_text('Ваш отчет отправлен. Хорошего Вам вечера!')

    return ConversationHandler.END

# Обработка ошибок
def error(update: Update, context: CallbackContext):
    logger.warning('Ошибка: %s', context.error)

# Функция для отправки напоминаний
def send_reminder(context: CallbackContext):
    job = context.job
    chat_id = job.context['chat_id']

    if job.name == 'morning_reminder':
        context.bot.send_message(chat_id, 'Доброе утро! Не забудьте начать свой рабочий день.')
    elif job.name == 'evening_reminder':
        context.bot.send_message(chat_id, 'Добрый вечер! Пора завершить рабочий день и отправить отчет.')

# Главная функция запуска бота
def main():
    # Ваш токен бота
    TOKEN = '7925358242:AAH2tcx0kbc4zFXtNGJwAYxwY5Wfr18QJLM'

    updater = Updater(TOKEN)
    dp = updater.dispatcher

    # Конверсационный хендлер
    conv_handler = ConversationHandler(
        entry_points=[CommandHandler('start', start)],
        states={
            NAME: [MessageHandler(Filters.text & ~Filters.command, handle_name)],
            START_WORK: [MessageHandler(Filters.regex('^Начать рабочий день$'), start_work)],
            END_WORK: [MessageHandler(Filters.regex('^Завершить рабочий день$'), end_work)],
            STORE_VISIT: [MessageHandler(Filters.text & ~Filters.command, store_visit)],
            PROFIT: [MessageHandler(Filters.text & ~Filters.command, report_profit)],
            PURCHASE: [MessageHandler(Filters.text & ~Filters.command, report_purchase)],
            COMMISSION: [MessageHandler(Filters.text & ~Filters.command, report_commission)],
            ADDITIONAL_WORK: [MessageHandler(Filters.text & ~Filters.command, additional_work)]
        },
        fallbacks=[]
    )

    dp.add_handler(conv_handler)

    # Настройка планировщиков
    scheduler = BackgroundScheduler()
    scheduler.add_job(send_reminder, 'cron', hour=9, minute=0, args=[context], id='morning_reminder')
    scheduler.add_job(send_reminder, 'cron', hour=20, minute=0, args=[context], id='evening_reminder')
    scheduler.start()

    # Запуск бота
    updater.start_polling()
    updater.idle()

if __name__ == '__main__':
    main()
