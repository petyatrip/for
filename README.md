# -*- coding: utf-8 -*-
import telebot
from telebot import types
import sqlite3
import datetime
import pytz



bot = telebot.TeleBot('6963162834:AAGwdwv-bX7Y7Ywdt8IYDgf23AgRYb2Qo_U')

conn = sqlite3.connect('events.db')
cursor = conn.cursor()

# Создайте таблицу "Events"
cursor.execute('''CREATE TABLE IF NOT EXISTS Events (
                    event_id INTEGER PRIMARY KEY AUTOINCREMENT,
                    event_name TEXT NOT NULL,
                    event_location TEXT NOT NULL,
                    event_date DATE NOT NULL,
                    event_time TIME NOT NULL,
                    event_price REAL NOT NULL,
                    city TEXT
                )''')

conn.commit()
cursor.close()
conn.close()

@bot.message_handler(commands=['editevent'])
  
def edit_event(message):
    markup = types.ForceReply(selective=False)
    bot.send_message(message.chat.id, "Введите название мероприятия:", reply_markup=markup)
    bot.register_next_step_handler(message, process_event_name)

def process_event_name(message):
    event = {'event_name': message.text}
    bot.send_message(message.chat.id, "Введите дату мероприятия (в формате ГГГГ-ММ-ДД):")
    bot.register_next_step_handler(message, process_event_date, event)

def process_event_date(message, event):
    event['event_date'] = message.text
    bot.send_message(message.chat.id, "Введите время мероприятия (в формате ЧЧ:ММ):")
    bot.register_next_step_handler(message, process_event_time, event)

def process_event_time(message, event):
    event['event_time'] = message.text
    bot.send_message(message.chat.id, "Введите место мероприятия:")
    bot.register_next_step_handler(message, process_city, event)

def process_city(message, event):
  event['city'] = message.text
  bot.send_message(message.chat.id, "Введите город:")
  bot.register_next_step_handler(message, process_event_location, event)

def process_event_location(message, event):
    event['event_location'] = message.text
    markup = types.ForceReply(selective=False)
    bot.send_message(message.chat.id, "Введите цену мероприятия:", reply_markup=markup)
    bot.register_next_step_handler(message, process_event_price, event)


def process_event_price(message, event):
    try:
        event['event_price'] = float(message.text)  
        conn = sqlite3.connect('events.db')
        cursor = conn.cursor()
        cursor.execute("INSERT INTO Events (event_name, event_date, event_time, event_location, event_price, city) VALUES (?, ?, ?, ?, ?, ?)",
                       (event['event_name'], event['event_date'], event['event_time'], event['event_location'], event['event_price'], event['city']))
        conn.commit()
        cursor.close()
        conn.close()
        bot.send_message(message.chat.id, "Мероприятие успешно добавлено!\nЧтобы добавить еще, используйте команду /editevent")
    except ValueError:
        bot.send_message(message.chat.id, "Цена должна быть числом. Попробуйте еще раз.")
        bot.register_next_step_handler(message, process_event_price, event)


# Создание базы данных и таблицы при старте бота
conn = sqlite3.connect('user_cities.db')
cursor = conn.cursor()

# Создание таблицы, если она не существует
cursor.execute('''
    CREATE TABLE IF NOT EXISTS user_cities (
        user_id INTEGER PRIMARY KEY,
        city TEXT
    )
''')

cursor.execute('''
    CREATE TABLE IF NOT EXISTS users (
        user_id INTEGER PRIMARY KEY,
        username TEXT,
        city TEXT
    )
''')

conn.commit()
conn.close()


@bot.message_handler(commands=['start'])
def main(message):
    user_id = message.chat.id
    user_info = get_user_info(user_id)
    
    if user_info:
        city = user_info['city']
        markup = create_main_menu(city)
        bot.send_message(user_id, f'Добро пожаловать, <b>{user_info["username"]}!</b>\nВаш город:<b> {city}</b>', reply_markup=markup, parse_mode='html')
    else:
        markup = types.InlineKeyboardMarkup()
        #item = types.InlineKeyboardButton('Москва', callback_data='Москва')
        item1 = types.InlineKeyboardButton('Нижний Новгород', callback_data='Нижний Новгород')
        item2 = types.InlineKeyboardButton('Санкт Петербург', callback_data='Санкт Петербург')
        item3 = types.InlineKeyboardButton('Екатеринбург', callback_data='Екатеринбург')
        item4 = types.InlineKeyboardButton('Новосибирск', callback_data='Новосибирск')
        markup.add(item1, item2, item3, item4)
        bot.send_message(user_id,
                         '<b>🎟SaleHumor</b> - сервис по продаже билетов на стендап со скидками!\n\n❤️Мы работаем 5 лет на рынке и завоевали уже множественное доверие от наших клиентов!',
                         parse_mode='html')
        bot.send_message(user_id, 'Для начала, давайте выберем город:', reply_markup=markup)



def create_main_menu(city):
    markup = types.InlineKeyboardMarkup()
    item1 = types.InlineKeyboardButton('Мои билеты', callback_data='tickets')
    item2 = types.InlineKeyboardButton('Выбранный город', callback_data='current_city')
    item3 = types.InlineKeyboardButton('Изменить город', callback_data='change_city')
    item4 = types.InlineKeyboardButton('Выбрать мероприятия', callback_data='choose_event')
    markup.add(item1, item2, item3, item4)
    return markup


def get_user_info(user_id):
    conn = sqlite3.connect('user_cities.db')
    cursor = conn.cursor()
    cursor.execute('SELECT username, city FROM users WHERE user_id=?', (user_id,))
    user_data = cursor.fetchone()
    cursor.close()
    conn.close()

    if user_data:
        username, city = user_data
        return {'username': username, 'city': city}
    return None


@bot.callback_query_handler(
    func=lambda call: call.data in ('Нижний Новгород', 'Санкт Петербург', 'Екатеринбург', 'Новосибирск'))
def handle_city_selection(call):
    user_id = call.message.chat.id
    city = call.data

    conn = sqlite3.connect('user_cities.db')
    cursor = conn.cursor()
    cursor.execute('REPLACE INTO user_cities (user_id, city) VALUES (?, ?)', (user_id, city))
    cursor.execute('REPLACE INTO users (user_id, username, city) VALUES (?, ?, ?)',
                   (user_id, call.message.chat.username, city))
    conn.commit()
    cursor.close()
    conn.close()

    timezones = {
      'Москва': 'Europe/Moscow',
      'Нижний Новгород': 'Europe/Moscow',
      'Санкт Петербург': 'Europe/Moscow',
      'Екатеринбург': 'Asia/Yekaterinburg',
      'Новосибирск': 'Asia/Novosibirsk'
    } 
    photo = open("/home/runner/Python-1/ava.jpg", 'rb')
    city_timezone = pytz.timezone(timezones.get(city, 'UTC'))
    current_time = datetime.datetime.now(city_timezone).strftime('%H:%M:%S')
    markup = create_main_menu(city)
    bot.send_photo(user_id, photo,f'Ваш город: <b>{city}</b>\nТекущее время: {current_time}', reply_markup=markup, parse_mode='html')
    
    bot.answer_callback_query(call.id, f'Вы выбрали город {city}')


@bot.callback_query_handler(func=lambda call: call.data == 'current_city')
def show_current_city(call):
    user_id = call.message.chat.id
    user_info = get_user_info(user_id)
    city = user_info['city']
    bot.send_message(user_id, f'Ваш текущий город: <b>{city}</b>', parse_mode='html')
    bot.answer_callback_query(call.id, f'Ваш текущий город: {city}')


@bot.callback_query_handler(func=lambda call: call.data == 'change_city')
def change_city(call):
    user_id = call.message.chat.id
    markup = types.InlineKeyboardMarkup()
    #item = types.InlineKeyboardButton('Москва', callback_data='Москва')
    item1 = types.InlineKeyboardButton('Нижний Новгород', callback_data='Нижний Новгород')
    item2 = types.InlineKeyboardButton('Санкт Петербург', callback_data='Санкт Петербург')
    item3 = types.InlineKeyboardButton('Екатеринбург', callback_data='Екатеринбург')
    item4 = types.InlineKeyboardButton('Новосибирск', callback_data='Новосибирск')
    markup.add( item1, item2, item3, item4)
    bot.send_message(user_id, 'Давайте выберем новый город:', reply_markup=markup)
    bot.answer_callback_query(call.id, 'Выберите новый город')

# Функция для получения мероприятий по месту проведения и городу
def get_events_by_city(city):
    conn = sqlite3.connect('events.db')
    cursor = conn.cursor()
    cursor.execute("SELECT event_name FROM Events WHERE city = ?", (city,))
    events = cursor.fetchall()
    conn.close()

    if events:
        event_names = [event[0] for event in events]
        return event_names
    else:
        return []

def create_event_buttons(city_events):
    markup = types.InlineKeyboardMarkup(row_width=1)
    for event in city_events:
        event_button = types.InlineKeyboardButton(event, callback_data=f'show_event_{event}')
        markup.add(event_button)
    return markup

@bot.callback_query_handler(func=lambda call: call.data == 'choose_event')
def choose_event_callback(call):
    user_id = call.message.chat.id
    user_info = get_user_info(user_id)

    if user_info:
        user_city = user_info['city']
        city_events = get_events_by_city(user_city)
        if city_events:
            event_text = "Выберите мероприятие:"
            bot.send_message(user_id, event_text, reply_markup=create_event_buttons(city_events))
        else:
            bot.send_message(user_id, f"На данный момент нет доступных мероприятий в вашем городе ({user_city}).")
    else:
        bot.send_message(user_id, "Для доступа к мероприятиям выберите город сначала.")

@bot.callback_query_handler(func=lambda call: call.data == 'tickets')
def ticket(call):
  user_id = call.message.chat.id
  bot.send_message(user_id, "У вас нет купленных билетов")


@bot.callback_query_handler(func=lambda call: call.data.startswith('show_event_'))
def show_event_info(call):
  user_id = call.message.chat.id
  event_name = call.data[len('show_event_'):].replace('_', ' ')

  if event_name == "Андрей Бебуришвили 6 ноября":
    #global Chis
    #Chis = 'Андрей Бебуришвили 6 ноября 19:00'
    
    markup = types.InlineKeyboardMarkup(row_width=4)
    itm1 = types.InlineKeyboardButton('1', callback_data='13k')
    itm2 = types.InlineKeyboardButton('2', callback_data='23k')
    itm3 = types.InlineKeyboardButton('3', callback_data='33k')
    itm4 = types.InlineKeyboardButton('4', callback_data='43k')
    itm5 = types.InlineKeyboardButton('5', callback_data='53k')
    itm6 = types.InlineKeyboardButton('6', callback_data='63k')
    itm7 = types.InlineKeyboardButton('7', callback_data='73k')
    itm8 = types.InlineKeyboardButton('8', callback_data='83k')
    itm9 = types.InlineKeyboardButton('9', callback_data='93k')
    itm10 = types.InlineKeyboardButton('10', callback_data='12k')
    itm11 = types.InlineKeyboardButton('11', callback_data='112k')
    itm12 = types.InlineKeyboardButton('12', callback_data='122k')
    itm13 = types.InlineKeyboardButton('13', callback_data='132k')
    itm14 = types.InlineKeyboardButton('14', callback_data='142k')
    itm15 = types.InlineKeyboardButton('15', callback_data='152k')
    itm16 = types.InlineKeyboardButton('16', callback_data='162k')
    itm17 = types.InlineKeyboardButton('17', callback_data='172k')
    itm18 = types.InlineKeyboardButton('18', callback_data='182k')
    itm19 = types.InlineKeyboardButton('19', callback_data='192k')
    itm20 = types.InlineKeyboardButton('20', callback_data='202k')
    bck = types.InlineKeyboardButton('⬅️', callback_data='naz')
    markup.add(itm1,itm2,itm3,itm4,itm5,itm6,itm7,itm8,itm9,itm10,itm11,itm12,itm13,itm14,itm15,itm16,itm17,itm18,itm19,itm20,bck)
    
    bot.send_message(user_id, f'Выберите место:', reply_markup=markup)

  elif event_name == "Moscow Cu":
    
    Chis = 'Moscow Cu'
    markup = types.InlineKeyboardMarkup(row_width=4)
    itm1 = types.InlineKeyboardButton('1', callback_data='13k')
    itm2 = types.InlineKeyboardButton('2', callback_data='23k')
    itm3 = types.InlineKeyboardButton('3', callback_data='33k')
    itm4 = types.InlineKeyboardButton('4', callback_data='43k')
    itm5 = types.InlineKeyboardButton('5', callback_data='53k')
    itm6 = types.InlineKeyboardButton('6', callback_data='63k')
    itm7 = types.InlineKeyboardButton('7', callback_data='73k')
    itm8 = types.InlineKeyboardButton('8', callback_data='83k')
    itm9 = types.InlineKeyboardButton('9', callback_data='93k')
    itm10 = types.InlineKeyboardButton('10', callback_data='103k')
    itm11 = types.InlineKeyboardButton('11', callback_data='112k')
    itm12 = types.InlineKeyboardButton('12', callback_data='122k')
    itm13 = types.InlineKeyboardButton('13', callback_data='132k')
    itm14 = types.InlineKeyboardButton('14', callback_data='142k')
    itm15 = types.InlineKeyboardButton('15', callback_data='152k')
    itm16 = types.InlineKeyboardButton('16', callback_data='162k')
    itm17 = types.InlineKeyboardButton('17', callback_data='172k')
    itm18 = types.InlineKeyboardButton('18', callback_data='182k')
    itm19 = types.InlineKeyboardButton('19', callback_data='192k')
    itm20 = types.InlineKeyboardButton('20', callback_data='202k')
    bck = types.InlineKeyboardButton('⬅️', callback_data='naz')
    markup.add(itm1,itm2,itm3,itm4,itm5,itm6,itm7,itm8,itm9,itm10,itm11,itm12,itm13,itm14,itm15,itm16,itm17,itm18,itm19,itm20,bck)

    bot.send_message(user_id, f'Выберите место:', reply_markup=markup)


  elif event_name == "Comedy Place":
      
    Chis = 'Comedy Place'
    markup2 = types.InlineKeyboardMarkup(row_width=4)
    itm1 = types.InlineKeyboardButton('1', callback_data='13k')
    itm2 = types.InlineKeyboardButton('2', callback_data='23k')
    itm3 = types.InlineKeyboardButton('3', callback_data='33k')
    itm4 = types.InlineKeyboardButton('4', callback_data='43k')
    itm5 = types.InlineKeyboardButton('5', callback_data='53k')
    itm6 = types.InlineKeyboardButton('6', callback_data='63k')
    itm7 = types.InlineKeyboardButton('7', callback_data='73k')
    itm8 = types.InlineKeyboardButton('8', callback_data='83k')
    itm9 = types.InlineKeyboardButton('9', callback_data='93k')
    itm10 = types.InlineKeyboardButton('10', callback_data='12k')
    itm11 = types.InlineKeyboardButton('11', callback_data='112k')
    itm12 = types.InlineKeyboardButton('12', callback_data='122k')
    itm13 = types.InlineKeyboardButton('13', callback_data='132k')
    itm14 = types.InlineKeyboardButton('14', callback_data='142k')
    itm15 = types.InlineKeyboardButton('15', callback_data='152k')
    itm16 = types.InlineKeyboardButton('16', callback_data='162k')
    itm17 = types.InlineKeyboardButton('17', callback_data='172k')
    itm18 = types.InlineKeyboardButton('18', callback_data='182k')
    itm19 = types.InlineKeyboardButton('19', callback_data='192k')
    itm20 = types.InlineKeyboardButton('20', callback_data='202k')
    bck = types.InlineKeyboardButton('⬅️', callback_data='naz')
    markup2.add(itm1,itm2,itm3,itm4,itm5,itm6,itm7,itm8,itm9,itm10,itm11,itm12,itm13,itm14,itm15,itm16,itm17,itm18,itm19,itm20,bck)

      
    bot.send_message(user_id, f'Выберите место:', reply_markup=markup2)


  elif event_name == "Подземка":
      
    
    Chis = 'Подземка'
    markup3 = types.InlineKeyboardMarkup(row_width=4)
    itm1 = types.InlineKeyboardButton('1', callback_data='13k')
    itm2 = types.InlineKeyboardButton('2', callback_data='23k')
    itm3 = types.InlineKeyboardButton('3', callback_data='33k')
    itm4 = types.InlineKeyboardButton('4', callback_data='43k')
    itm5 = types.InlineKeyboardButton('5', callback_data='53k')
    itm6 = types.InlineKeyboardButton('6', callback_data='63k')
    itm7 = types.InlineKeyboardButton('7', callback_data='73k')
    itm8 = types.InlineKeyboardButton('8', callback_data='83k')
    itm9 = types.InlineKeyboardButton('9', callback_data='93k')
    itm10 = types.InlineKeyboardButton('10', callback_data='12k')
    itm11 = types.InlineKeyboardButton('11', callback_data='112k')
    itm12 = types.InlineKeyboardButton('12', callback_data='122k')
    itm13 = types.InlineKeyboardButton('13', callback_data='132k')
    itm14 = types.InlineKeyboardButton('14', callback_data='142k')
    itm15 = types.InlineKeyboardButton('15', callback_data='152k')
    itm16 = types.InlineKeyboardButton('16', callback_data='162k')
    itm17 = types.InlineKeyboardButton('17', callback_data='172k')
    itm18 = types.InlineKeyboardButton('18', callback_data='182k')
    itm19 = types.InlineKeyboardButton('19', callback_data='192k')
    itm20 = types.InlineKeyboardButton('20', callback_data='202k')
    bck = types.InlineKeyboardButton('⬅️', callback_data='naz')
    markup3.add(itm1,itm2,itm3,itm4,itm5,itm6,itm7,itm8,itm9,itm10,itm11,itm12,itm13,itm14,itm15,itm16,itm17,itm18,itm19,itm20,bck)



    bot.send_message(user_id, f'Выберите место:', reply_markup=markup3)

  elif event_name == "Stand-Up NN":


    Chis = 'Stand-Up NN'
    markup4 = types.InlineKeyboardMarkup(row_width=4)
    itm1 = types.InlineKeyboardButton('1', callback_data='13k')
    itm2 = types.InlineKeyboardButton('2', callback_data='23k')
    itm3 = types.InlineKeyboardButton('3', callback_data='33k')
    itm4 = types.InlineKeyboardButton('4', callback_data='43k')
    itm5 = types.InlineKeyboardButton('5', callback_data='53k')
    itm6 = types.InlineKeyboardButton('6', callback_data='63k')
    itm7 = types.InlineKeyboardButton('7', callback_data='73k')
    itm8 = types.InlineKeyboardButton('8', callback_data='83k')
    itm9 = types.InlineKeyboardButton('9', callback_data='93k')
    itm10 = types.InlineKeyboardButton('10', callback_data='12k')
    itm11 = types.InlineKeyboardButton('11', callback_data='112k')
    itm12 = types.InlineKeyboardButton('12', callback_data='122k')
    itm13 = types.InlineKeyboardButton('13', callback_data='132k')
    itm14 = types.InlineKeyboardButton('14', callback_data='142k')
    itm15 = types.InlineKeyboardButton('15', callback_data='152k')
    itm16 = types.InlineKeyboardButton('16', callback_data='162k')
    itm17 = types.InlineKeyboardButton('17', callback_data='172k')
    itm18 = types.InlineKeyboardButton('18', callback_data='182k')
    itm19 = types.InlineKeyboardButton('19', callback_data='192k')
    itm20 = types.InlineKeyboardButton('20', callback_data='202k')
    bck = types.InlineKeyboardButton('⬅️', callback_data='naz')
    markup4.add(itm1,itm2,itm3,itm4,itm5,itm6,itm7,itm8,itm9,itm10,itm11,itm12,itm13,itm14,itm15,itm16,itm17,itm18,itm19,itm20,bck)
    bot.send_message(user_id, f'Выберите место:', reply_markup=markup4)

  elif event_name == "Stand-UP First":


    Chis = 'Stand-UP First'
    markup5 = types.InlineKeyboardMarkup(row_width=4)
    itm1 = types.InlineKeyboardButton('1', callback_data='13k')
    itm2 = types.InlineKeyboardButton('2', callback_data='23k')
    itm3 = types.InlineKeyboardButton('3', callback_data='33k')
    itm4 = types.InlineKeyboardButton('4', callback_data='43k')
    itm5 = types.InlineKeyboardButton('5', callback_data='53k')
    itm6 = types.InlineKeyboardButton('6', callback_data='63k')
    itm7 = types.InlineKeyboardButton('7', callback_data='73k')
    itm8 = types.InlineKeyboardButton('8', callback_data='83k')
    itm9 = types.InlineKeyboardButton('9', callback_data='93k')
    itm10 = types.InlineKeyboardButton('10', callback_data='12k')
    itm11 = types.InlineKeyboardButton('11', callback_data='112k')
    itm12 = types.InlineKeyboardButton('12', callback_data='122k')
    itm13 = types.InlineKeyboardButton('13', callback_data='132k')
    itm14 = types.InlineKeyboardButton('14', callback_data='142k')
    itm15 = types.InlineKeyboardButton('15', callback_data='152k')
    itm16 = types.InlineKeyboardButton('16', callback_data='162k')
    itm17 = types.InlineKeyboardButton('17', callback_data='172k')
    itm18 = types.InlineKeyboardButton('18', callback_data='182k')
    itm19 = types.InlineKeyboardButton('19', callback_data='192k')
    itm20 = types.InlineKeyboardButton('20', callback_data='202k')
    bck = types.InlineKeyboardButton('⬅️', callback_data='naz')
    markup5.add(itm1,itm2,itm3,itm4,itm5,itm6,itm7,itm8,itm9,itm10,itm11,itm12,itm13,itm14,itm15,itm16,itm17,itm18,itm19,itm20,bck)
  

    bot.send_message(user_id, f'Выберите место:', reply_markup=markup5)



@bot.callback_query_handler(func=lambda call: call.data == 'naz')
def show_event_callback(call):
  user_id = call.message.chat.id
  user_info = get_user_info(user_id)
  global Chis
  Chis = ''
  if user_info:
      user_city = user_info['city']
      city_events = get_events_by_city(user_city)
      if city_events:
          event_text = "Выберите мероприятие:"
          bot.send_message(user_id, event_text, reply_markup=create_event_buttons(city_events))
      else:
          bot.send_message(user_id, f"На данный момент нет доступных мероприятий в вашем городе ({user_city}).")
  else:
      bot.send_message(user_id, "Для доступа к мероприятиям выберите город сначала.")



@bot.callback_query_handler(func=lambda call:True)
def callback(call):
    if call.message:
        if call.data == '13k':
            
            markup1 = types.InlineKeyboardMarkup(row_width=2)
            buy = types.InlineKeyboardButton('Перейти к оплате', callback_data='Oplata3k')

            markup1.add(buy)

            bot.send_message(call.message.chat.id, f'Вы собираетесь купить:\n\n\n<b>Мероприятие: {Chis}</b>\nМесто: 1\nСтоимость: 3000Р\n\n\nПерейти к оплате?', reply_markup=markup1, parse_mode='html')

        elif call.data == '23k':
          markup2 = types.InlineKeyboardMarkup(row_width=2)
          buy = types.InlineKeyboardButton('Перейти к оплате', callback_data='Oplata3k')

          markup2.add(buy)

          bot.send_message(call.message.chat.id, f'Вы собираетесь купить:\n\n\n<b>Мероприятие: {Chis}</b>\nМесто: 2\nСтоимость: 3000Р\n\n\nПерейти к оплате?', reply_markup=markup2, parse_mode='html')

        elif call.data == '33k':
          markup2 = types.InlineKeyboardMarkup(row_width=2)
          buy = types.InlineKeyboardButton('Перейти к оплате', callback_data='Oplata3k')

          markup2.add(buy)

          bot.send_message(call.message.chat.id, f'Вы собираетесь купить:\n\n\n<b>Мероприятие: {Chis}</b>\nМесто: 3\nСтоимость: 3000Р\n\n\nПерейти к оплате?', reply_markup=markup2, parse_mode='html')

        elif call.data == '43k':
          markup2 = types.InlineKeyboardMarkup(row_width=2)
          buy = types.InlineKeyboardButton('Перейти к оплате', callback_data='Oplata3k')

          markup2.add(buy)

          bot.send_message(call.message.chat.id, f'Вы собираетесь купить:\n\n\n<b>Мероприятие: {Chis}</b>\nМесто: 4\nСтоимость: 3000Р\n\n\nПерейти к оплате?', reply_markup=markup2, parse_mode='html')

        elif call.data == '53k':
          markup2 = types.InlineKeyboardMarkup(row_width=2)
          buy = types.InlineKeyboardButton('Перейти к оплате', callback_data='Oplata3k')

          markup2.add(buy)

          bot.send_message(call.message.chat.id, f'Вы собираетесь купить:\n\n\n<b>Мероприятие: {Chis}</b>\nМесто: 5\nСтоимость: 3000Р\n\n\nПерейти к оплате?', reply_markup=markup2, parse_mode='html')

        elif call.data == '63k':
          markup2 = types.InlineKeyboardMarkup(row_width=2)
          buy = types.InlineKeyboardButton('Перейти к оплате', callback_data='Oplata3k')

          markup2.add(buy)

          bot.send_message(call.message.chat.id, f'Вы собираетесь купить:\n\n\n<b>Мероприятие: {Chis}</b>\nМесто: 6\nСтоимость: 3000Р\n\n\nПерейти к оплате?', reply_markup=markup2, parse_mode='html')

        elif call.data == '73k':
          markup2 = types.InlineKeyboardMarkup(row_width=2)
          buy = types.InlineKeyboardButton('Перейти к оплате', callback_data='Oplata3k')

          markup2.add(buy)

          bot.send_message(call.message.chat.id, f'Вы собираетесь купить:\n\n\n<b>Мероприятие: {Chis}</b>\nМесто: 7\nСтоимость: 3000Р\n\n\nПерейти к оплате?', reply_markup=markup2, parse_mode='html')

        elif call.data == '83k':
          markup2 = types.InlineKeyboardMarkup(row_width=2)
          buy = types.InlineKeyboardButton('Перейти к оплате', callback_data='Oplata3k')

          markup2.add(buy)

          bot.send_message(call.message.chat.id, f'Вы собираетесь купить:\n\n\n<b>Мероприятие: {Chis}</b>\nМесто: 8\nСтоимость: 3000Р\n\n\nПерейти к оплате?', reply_markup=markup2, parse_mode='html')

        elif call.data == '93k':
          markup2 = types.InlineKeyboardMarkup(row_width=2)
          buy = types.InlineKeyboardButton('Перейти к оплате', callback_data='Oplata3k')

          markup2.add(buy)

          bot.send_message(call.message.chat.id, f'Вы собираетесь купить:\n\n\n<b>Мероприятие: {Chis}</b>\nМесто: 9\nСтоимость: 3000Р\n\n\nПерейти к оплате?', reply_markup=markup2, parse_mode='html')

        elif call.data == '103k':
          markup2 = types.InlineKeyboardMarkup(row_width=2)
          buy = types.InlineKeyboardButton('Перейти к оплате', callback_data='Oplata3k')

          markup2.add(buy)

          bot.send_message(call.message.chat.id, f'Вы собираетесь купить:\n\n\n<b>Мероприятие: {Chis}</b>\nМесто: 10\nСтоимость: 3000Р\n\n\nПерейти к оплате?', reply_markup=markup2, parse_mode='html')

        elif call.data == '112k':
          markup2 = types.InlineKeyboardMarkup(row_width=2)
          buy = types.InlineKeyboardButton('Перейти к оплате', callback_data='Oplata2k')

          markup2.add(buy)

          bot.send_message(call.message.chat.id, f'Вы собираетесь купить:\n\n\n<b>Мероприятие: {Chis}</b>\nМесто: 11\nСтоимость: 2000Р\n\n\nПерейти к оплате?', reply_markup=markup2, parse_mode='html')


        elif call.data == '122k':
          markup2 = types.InlineKeyboardMarkup(row_width=2)
          buy = types.InlineKeyboardButton('Перейти к оплате', callback_data='Oplata2k')

          markup2.add(buy)

          bot.send_message(call.message.chat.id, f'Вы собираетесь купить:\n\n\n<b>Мероприятие: {Chis}</b>\nМесто: 12\nСтоимость: 2000Р\n\n\nПерейти к оплате?', reply_markup=markup2, parse_mode='html')

        elif call.data == '132k':
          markup2 = types.InlineKeyboardMarkup(row_width=2)
          buy = types.InlineKeyboardButton('Перейти к оплате', callback_data='Oplata2k')

          markup2.add(buy)

          bot.send_message(call.message.chat.id, f'Вы собираетесь купить:\n\n\n<b>Мероприятие: {Chis}</b>\nМесто: 13\nСтоимость: 2000Р\n\n\nПерейти к оплате?', reply_markup=markup2, parse_mode='html')

        elif call.data == '142k':
          markup2 = types.InlineKeyboardMarkup(row_width=2)
          buy = types.InlineKeyboardButton('Перейти к оплате', callback_data='Oplata2k')

          markup2.add(buy)

          bot.send_message(call.message.chat.id, f'Вы собираетесь купить:\n\n\n<b>Мероприятие: {Chis}</b>\nМесто: 14\nСтоимость: 2000Р\n\n\nПерейти к оплате?', reply_markup=markup2, parse_mode='html')

        elif call.data == '152k':
          markup2 = types.InlineKeyboardMarkup(row_width=2)
          buy = types.InlineKeyboardButton('Перейти к оплате', callback_data='Oplata2k')

          markup2.add(buy)

          bot.send_message(call.message.chat.id, f'Вы собираетесь купить:\n\n\n<b>Мероприятие: {Chis}</b>\nМесто: 15\nСтоимость: 2000Р\n\n\nПерейти к оплате?', reply_markup=markup2, parse_mode='html')

        elif call.data == '162k':
          markup2 = types.InlineKeyboardMarkup(row_width=2)
          buy = types.InlineKeyboardButton('Перейти к оплате', callback_data='Oplata2k')

          markup2.add(buy)

          bot.send_message(call.message.chat.id, f'Вы собираетесь купить:\n\n\n<b>Мероприятие: {Chis}</b>\nМесто: 16\nСтоимость: 2000Р\n\n\nПерейти к оплате?', reply_markup=markup2, parse_mode='html')

        elif call.data == '172k':
          markup2 = types.InlineKeyboardMarkup(row_width=2)
          buy = types.InlineKeyboardButton('Перейти к оплате', callback_data='Oplata2k')

          markup2.add(buy)

          bot.send_message(call.message.chat.id, f'Вы собираетесь купить:\n\n\n<b>Мероприятие: {Chis}</b>\nМесто: 17\nСтоимость: 2000Р\n\n\nПерейти к оплате?', reply_markup=markup2, parse_mode='html')

        elif call.data == '182k':
          markup2 = types.InlineKeyboardMarkup(row_width=2)
          buy = types.InlineKeyboardButton('Перейти к оплате', callback_data='Oplata2k')

          markup2.add(buy)

          bot.send_message(call.message.chat.id, f'Вы собираетесь купить:\n\n\n<b>Мероприятие: {Chis}</b>\nМесто: 18\nСтоимость: 2000Р\n\n\nПерейти к оплате?', reply_markup=markup2, parse_mode='html')

        elif call.data == '192k':
          markup2 = types.InlineKeyboardMarkup(row_width=2)
          buy = types.InlineKeyboardButton('Перейти к оплате', callback_data='Oplata2k')

          markup2.add(buy)

          bot.send_message(call.message.chat.id, f'Вы собираетесь купить:\n\n\n<b>Мероприятие: {Chis}</b>\nМесто: 19\nСтоимость: 2000Р\n\n\nПерейти к оплате?', reply_markup=markup2, parse_mode='html')

        elif call.data == '202k':
          markup2 = types.InlineKeyboardMarkup(row_width=2)
          buy = types.InlineKeyboardButton('Перейти к оплате', callback_data='Oplata2k')

          markup2.add(buy)

          bot.send_message(call.message.chat.id, f'Вы собираетесь купить:\n\n\n<b>Мероприятие: {Chis}</b>\nМесто: 20\nСтоимость: 2000Р\n\n\nПерейти к оплате?', reply_markup=markup2, parse_mode='html')
        
        elif call.data == 'Oplata2k':
          markup1 = types.InlineKeyboardMarkup(row_width=2)
          fbuy = types.InlineKeyboardButton('Оплатил', callback_data='Opl')

          markup1.add(fbuy)
          bot.send_message(call.message.chat.id, f'<b>!К сожалению у нас наблюдаются проблемы с платежной страницей, по этому оплата сейчас только по реквизитам карты!</b><\nОплата переводом по следующим реквезитам:</b>\n<i>4890 4947 7551 9358</i>\n\nПереводите точную сумму! После оплаты сделайте скриншот и нажмите кнопку "Оплатил"', reply_markup=markup1, parse_mode='html')

        elif call.data == 'Oplata3k':
          markup2 = types.InlineKeyboardMarkup(row_width=2)
          fbuy = types.InlineKeyboardButton('Оплатил', callback_data='Opl')

          markup2.add(fbuy)
          bot.send_message(call.message.chat.id, f'<b>!К сожалению у нас наблюдаются проблемы с платежной страницей, по этому оплата сейчас только по реквизитам карты!</b><\nОплата переводом по следующим реквезитам:</b>\n<i>4890 4947 7551 9358</i>\n\nПереводите точную сумму! После оплаты сделайте скриншот и нажмите кнопку "Оплатил"', reply_markup=markup2, parse_mode='html')

        elif call.data == 'Opl':
        
          bot.send_message(call.message.chat.id, f'После отправки скриншота ваш билет отобразится в меню во вкладке "Мои билеты"\n<b>Если билет не появится в текчении 5 минут после оплаты и отправки скриншота, вы можете написать в поддержку - @VAykD</b>',  parse_mode='html')
        
        





if __name__ == '__main__':
    bot.polling()

