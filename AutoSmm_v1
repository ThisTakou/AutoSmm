import json
import logging
import os
import re
import threading
import time
from datetime import datetime
from typing import TYPE_CHECKING, Optional, List, Dict

import requests
import telebot
from telebot import types
from telebot.types import InlineKeyboardMarkup, InlineKeyboardButton
from cardinal import Cardinal
if TYPE_CHECKING:
    from cardinal import Cardinal

from FunPayAPI.updater.events import *
from FunPayAPI.types import MessageTypes

from locales.localizer import Localizer
from tg_bot.utils import load_authorized_users


pending_confirmations = {}
logger = logging.getLogger("FPC.AutoSmm")
localizer = Localizer()
_ = localizer.translate

LOGGER_PREFIX = "AutoSmm Plugin"
NAME = "AutoSMM Retrived by Takouq - @llzzvvww"
VERSION = "5.0 AutoChanger" 
CREDITS = "@llzzvvww / @takouq "
UUID = "7aa412ab-0840-455d-9513-6f51bf83d43b"
SETTINGS_PAGE = False

ORDERS_FILE = f"storage/plugins/{UUID}/orders.json"
PAYORDERS_FILE = f"storage/plugins/{UUID}/payorders.json"
SETTINGS_FILE = f"storage/plugins/{UUID}/settings.json"
CASHLIST_FILE = f"storage/plugins/{UUID}/cashlist.json"
REFILL_FILE = f"storage/plugins/{UUID}/refill.json"

DEFAULT_SETTINGS = {
    "api_url": "",
    "api_key": "",
    "set_alert_neworder": True,
    "set_alert_errororder": True,
    "set_alert_smmbalance_new": False,
    "set_alert_smmbalance": True,
    "set_refund_smm": True,
    "set_start_mess": True,
    "set_auto_refill": False,
    "set_tg_private": False
}

# список для чекера заказов
def load_orders() -> dict:
    if os.path.exists(ORDERS_FILE):
        with open(ORDERS_FILE, "r") as file:
            return json.load(file)
    return {}

def save_orders(orders: dict) -> None:
    os.makedirs(f"storage/plugins/{UUID}", exist_ok=True)
    with open(ORDERS_FILE, "w") as file:
        json.dump(orders, file, indent=4)


# список для новых оплаченных заказов
def load_payorders() -> List[Dict]:
    if os.path.exists(PAYORDERS_FILE):
        with open(PAYORDERS_FILE, "r") as file:
            return json.load(file)
    return []

def save_payorders(orders: List[Dict]) -> None:
    os.makedirs(f"storage/plugins/{UUID}", exist_ok=True)
    with open(PAYORDERS_FILE, "w") as file:
        json.dump(orders, file, indent=4)
        
        
# временный список для пересозданных заказов
def load_cashlist() -> dict:
    if os.path.exists(CASHLIST_FILE):
        with open(CASHLIST_FILE, "r") as file:
            return json.load(file)
    return {}

def save_cashlist(orders: dict) -> None:
    os.makedirs(f"storage/plugins/{UUID}", exist_ok=True)
    with open(CASHLIST_FILE, "w") as file:
        json.dump(orders, file, indent=4)
        
        
# список для рефилла
def load_refill() -> dict:
    if os.path.exists(REFILL_FILE):
        with open(REFILL_FILE, "r") as file:
            return json.load(file)
    return {}

def save_refill(orders: dict) -> None:
    os.makedirs(f"storage/plugins/{UUID}", exist_ok=True)
    with open(REFILL_FILE, "w") as file:
        json.dump(orders, file, indent=4)




# Загружает настройки из файла или создает новые, если файла нет.
def load_settings():
    if not os.path.exists(SETTINGS_FILE):
        # Файла не существует, создаем новый
        settings = DEFAULT_SETTINGS.copy()
        save_settings(settings)
    
    with open(SETTINGS_FILE, 'r') as file:
        return json.load(file)

def save_settings(settings):
    os.makedirs(f"storage/plugins/{UUID}", exist_ok=True)
    with open(SETTINGS_FILE, 'w') as file:
        json.dump(settings, file, indent=4)
        
def get_api_url(type=None):
    """Возвращает URL API из настроек."""
    settings = load_settings()
    if type:
        return settings.get("api_url_2", "")
    return settings.get("api_url", "")

def get_api_key(type=None):
    """Возвращает ключ API из настроек."""
    settings = load_settings()
    if type:
        return settings.get("api_key_2", "")
    return settings.get("api_key", "")


def extract_links(text: str) -> List[str]:
    """
    Парсит ссылку из сообщения
    """
    link_pattern = r'http[s]?://(?:[a-zA-Z]|[0-9]|[$-_@.&+]|[!*\\(\\),]|(?:%[0-9a-fA-F][0-9a-fA-F]))+'
    return re.findall(link_pattern, text)


def find_order_by_buyer(orders: List[Dict], buyer: str) -> Optional[Dict]:
    """
    Ищем ордер по имени покупателя
    """
    for order in orders:
        if order['buyer'] == buyer:
            return order
    return None


def bind_to_new_order(c: Cardinal, e: NewOrderEvent) -> None:
    try:
        _element_data = e.order
        _order_id = _element_data.id
        _element_full_data = c.account.get_order(_order_id)
        _full_disc = _element_full_data.full_description
        _buyer_uz = _element_full_data.buyer_username
        
        settings = load_settings()
        if settings.get("set_alert_smmbalance_new", False):
            send_smm_balance_info(c)
        
        match_id = re.search(r'ID:\s*(\d+)', _full_disc)
        match_oid = re.search(r'ID2:\s*(\d+)', _full_disc)
        match_quan = re.search(r'#Quan:\s*(\d+)', _full_disc)
        
        if match_id:
            id_value = match_id.group(1)
            if match_quan:
                quan_value = int(match_quan.group(1))
            else:
                quan_value = 1
            order_handler(c, e, id_value, quan_value, _buyer_uz)
        elif match_oid:
            id_value = match_oid.group(1)
            if match_quan:
                quan_value = int(match_quan.group(1))
            else:
                quan_value = 1
            order_handler(c, e, id_value, quan_value, _buyer_uz, 'API_2')
        else:
            logger.info("Заказ не найден для автонакрутки")
    except Exception as ex:
        logger.error(ex)


def order_handler(c: Cardinal, e: NewOrderEvent, id_value, quan_value, buyer_uz, type_api='API_1') -> None:
    try:
        orders_data = load_payorders()

        order_ = e.order
        orderID = order_.id
        orderAmount = order_.amount * quan_value  # умножаем количество на значение из #Quan:
        orderPrice = order_.price
        orderCurrency = order_.currency
        url = "" 
        
        current_datetime = datetime.now().strftime("%Y-%m-%d %H:%M:%S") #час бачив? пісяти і спати

        current_order_data = {
            'OrderID': str(orderID),  
            'Amount': int(orderAmount),  
            'OrderPrice': orderPrice,  
            'OrderCurrency': f"{orderCurrency}",
            'Order': f"{str(order_)}",  
            'service_id': int(id_value),  
            'buyer': str(buyer_uz),  
            'url': str(url),  
            'NewUser': True,  
            'chat_id': "",  
            'OrderDateTime': current_datetime,  
            'api_type': type_api  # Тип API, default api 1
        }

        orders_data.append(current_order_data)
        save_payorders(orders_data)

        handle_order(c, current_order_data, [])

    except Exception as ex:
        logger.error(f"Ошибка при обработке нового заказа: {ex}")


logger.info(f"$MAGENTA{LOGGER_PREFIX} успешно запущен.$RESET")
DESCRIPTION = "Плагин добавляет возможность автонакрутки с api-V2!"


def msg_hook(c: Cardinal, e: NewMessageEvent) -> None:
    """
    Обрабатывает новое сообщение
    """
    orders_data = load_payorders()
    
    msg = e.message
    msgname = msg.chat_name
    message_text = msg.text.strip()
    
    order = find_order_by_buyer(orders_data, msgname)
    try:
        api_key = get_api_key() if order.get('api_type') == 'API_1' else get_api_key(order.get('api_type'))
        api_url = get_api_url() if order.get('api_type') == 'API_1' else get_api_url(order.get('api_type'))
    except Exception as e:
        api_key = get_api_key()
        api_url = get_api_url()
    
    
    # Удаление пользователя из списка при отмене заказа
    if "вернул деньги покупателю" in message_text:
        order = find_order_by_buyer(orders_data, msgname)
        if order:
            try:
                orders_data.remove(order)
                save_payorders(orders_data)
            except Exception as e:
                logger.error(f"Ошибка при возврате: {e}")
        return
        
    # Проверка на системные сообщения
    if msg.type != MessageTypes.NON_SYSTEM:
        logger.info("Ignoring system message.")
        return

    # Проверка на свои сообщения
    if msg.author_id == c.account.id:
        return

    # Проверяем, есть ли ожидаемое подтверждение
    if msg.chat_id in pending_confirmations:
        if message_text in ["+", "-"]:
            confirm_order(c, msg.chat_id, message_text, api_url, api_key)
        elif "http" in message_text:  # Проверяем, содержит ли сообщение ссылку
            order = pending_confirmations.get(msg.chat_id)
            if order:
                order['chat_id'] = msg.chat_id
                links = extract_links(message_text)
                handle_order(c, order, links)
        else:
            # Сообщение с ЗАЩИТОЙ от новых правил FunPay
            protection_msg = (
                "⚪️ Подтвердите заказ:\n\n"
                "✅ Для подтверждения отправьте: +\n"
                "❌ Для отмены отправьте: -\n\n"
                "⚠️ ВАЖНО:\n"
                "🕸️ Сроки выполнения заказов от 1 минуты до 96 часов. (В редких случаях)\n"
                "🔴 Отмена заказа доступна только сейчас\n"
                "🕸️ Продолжая покупку вы соглашаетесь с написаным выше, а также с информацией в описании."
            )
            c.send_message(msg.chat_id, protection_msg)
        return

    # Ищем ордер по имени покупателя
    order = find_order_by_buyer(orders_data, msgname)
    if order:
        logger.info(f"Пользователь {msgname} есть в списке заказов")
        order['chat_id'] = msg.chat_id
        links = extract_links(message_text)
        handle_order(c, order, links)
    else:
        logger.info(f"Пользователя {msgname} нету в списке заказов автонакрутки")

    
    command_parts = msg.text.split()
    if len(command_parts) >= 2 and command_parts[0] == "#статус":
        smm_order_id = command_parts[1]

        status = SocTypeAPI.get_order_status(int(smm_order_id), api_url, api_key)
        if status:
            start_count = status['start_count']
            if start_count == 0:
                display_start_count = "*"
            else:
                display_start_count = str(start_count)
            
            status_text = f"📈 Статус заказа: {smm_order_id}\n"
            status_text += f"⠀∟📊 Статус: {status['status']}\n"
            status_text += f"⠀∟🔢 Было: {display_start_count}\n"
            status_text += f"⠀∟👀 Остаток выполнения: {status['remains']}"
            c.send_message(msg.chat_id, status_text)
        else:
            c.send_message(msg.chat_id, "🔴 Не удалось получить статус заказа.")
            
    if len(command_parts) >= 2 and command_parts[0] == "#инфо":
        smm_order_id = command_parts[1]

        status = SocTypeAPI.get_order_status(int(smm_order_id), get_api_url('API_2'), get_api_key('API_2'))
        if status:
            start_count = status['start_count']
            if start_count == 0:
                display_start_count = "*"
            else:
                display_start_count = str(start_count)
            
            status_text = f"📈 Статус заказа: {smm_order_id}\n"
            status_text += f"⠀∟📊 Статус: {status['status']}\n"
            status_text += f"⠀∟🔢 Было: {display_start_count}\n"
            status_text += f"⠀∟👀 Остаток выполнения: {status['remains']}"
            c.send_message(msg.chat_id, status_text)
        else:
            c.send_message(msg.chat_id, "🔴 Не удалось получить статус заказа.")

    elif len(command_parts) >= 2 and command_parts[0] == "#рефилл":
        smm_order_id = command_parts[1]
        refill_result = SocTypeAPI.refill_order(int(smm_order_id), api_url, api_key)
        if refill_result is not None:
            c.send_message(msg.chat_id, f"✅ Запрос на рефилл отправлен!")
        else:
            c.send_message(msg.chat_id, f"🔴 Ошибка при выполнении рефилла.\n⚠️ Возможно, рефилл еще недоступен!")
    
    





def handle_order(c: Cardinal, order: Dict, links: List[str]) -> None:
    """
    Логика обработки сообщений от покупателя
    """
    if links:
        link = links[0]
        orders_data = load_payorders()
        settings = load_settings()
        if not settings.get("set_tg_private", False):
            if "t.me" in link and ("/c/" in link or "+" in link):
                c.send_message(order['chat_id'], "❌ Данный тип ссылки не поддерживается. Канал/группа должны быть публичными!")
                return
        order['url'] = link
        link = link.replace("https://", "")
        confirmation_text = f"""━━━━━━━━━━━━━━━━━━━━
📋 ДЕТАЛИ ЗАКАЗА
━━━━━━━━━━━━━━━━━━━━

🔗 Ссылка: {link}
📊 Количество: {order['Amount']} шт
💰 Цена: {order['OrderPrice']} {order['OrderCurrency']}

━━━━━━━━━━━━━━━━━━━━

✅ Подтвердить → +
❌ Отменить → -
🔄 Новая ссылка → отправьте её

━━━━━━━━━━━━━━━━━━━━
⚠️ УСЛОВИЯ ЗАКАЗА
━━━━━━━━━━━━━━━━━━━━

🕸️ Сроки: от 1 минуты до 96 часов
🔴 Отмена доступна только СЕЙЧАС  
🕸️ Подтверждая (+), вы соглашаетесь

━━━━━━━━━━━━━━━━━━━━"""
        c.send_message(order['chat_id'], confirmation_text)
        
        pending_confirmations[order['chat_id']] = order
        # сохранение в json (нужно!)
        orders_data = load_payorders()
        existing_order = next((o for o in orders_data if o['OrderID'] == order['OrderID']), None)
        
        if existing_order:
            existing_order.update(order)
        else:
            orders_data.append(order)
        
        save_payorders(orders_data)


def confirm_order(c: Cardinal, chat_id: int, text: str, api_url, api_key) -> None:
    """
    Подтверждает заказ, если пользователь отправил "+"
    """
    orders_data = load_payorders()
    settings = load_settings()
    
    if chat_id in pending_confirmations:
        order = pending_confirmations.pop(chat_id)
        
        if text.strip() == "+":
            smm_order_id = None
            try:
                smm_order_id = SocTypeAPI.create_order(order['service_id'], order['url'], order['Amount'], api_url, api_key)
            except Exception as e:
                logger.error(f"Exception при создании заказа: {e}")
                smm_order_id = None
            
            # ПРОВЕРКА НА ОШИБКУ ЛИМИТА И РАЗДЕЛЕНИЕ НА ЧАСТИ
            if smm_order_id and isinstance(smm_order_id, str) and "quantity must be between" in smm_order_id.lower():
                logger.info(f"Превышен лимит, разделяем заказ на части")
                
                # Извлекаем лимиты из ошибки
                match = re.search(r'between\s+(\d+)\s+and\s+(\d+)', smm_order_id, re.IGNORECASE)
                if match:
                    min_limit = int(match.group(1))
                    max_limit = int(match.group(2))
                    total_amount = order['Amount']
                    
                    # Уведомляем
                    c.send_message(order['chat_id'], 
                                 f"⚠️ Количество ({total_amount}) превышает лимит ({max_limit}).\n"
                                 f"🔄 Разделяем заказ на части...\n"
                                 f"⏳ Подождите...")
                    
                    # Разделяем
                    parts = []
                    remaining = total_amount
                    while remaining > 0:
                        part_size = min(max_limit, remaining)
                        parts.append(part_size)
                        remaining -= part_size
                    
                    # Создаём подзаказы
                    created_orders = []
                    failed_parts = 0
                    
                    for i, part_amount in enumerate(parts, 1):
                        try:
                            part_order_id = SocTypeAPI.create_order(
                                order['service_id'], order['url'], part_amount, api_url, api_key
                            )
                            
                            if isinstance(part_order_id, (int, str)) and str(part_order_id).isdigit():
                                created_orders.append({
                                    "smm_id": int(part_order_id),
                                    "amount": part_amount,
                                    "part_num": i
                                })
                                
                                orders = load_orders()
                                orders[part_order_id] = {
                                    "service_id": order['service_id'],
                                    "chat_id": order['chat_id'],
                                    "order_id": order['OrderID'],
                                    "order_url": order['url'],
                                    "order_amount": part_amount,
                                    "partial_amount": 0,
                                    "orderdatetime": order['OrderDateTime'],
                                    "status": "pending",
                                    "is_part": True,
                                    "part_number": i,
                                    "total_parts": len(parts)
                                }
                                save_orders(orders)
                                logger.info(f"Создана часть {i}/{len(parts)}: ID {part_order_id}")
                            else:
                                failed_parts += 1
                        except Exception as e:
                            failed_parts += 1
                            logger.error(f"Ошибка части {i}: {e}")
                    
                    # Результат
                    if created_orders:
                        status = 'статус' if order.get('api_type') == 'API_1' else 'инфо'
                        
                        msg = f"""━━━━━━━━━━━━━━━━━━━━
✅ ЗАКАЗ РАЗДЕЛЁН
━━━━━━━━━━━━━━━━━━━━

📊 Общее: {total_amount} шт
✅ Создано: {len(created_orders)}/{len(parts)} частей

━━━━━━━━━━━━━━━━━━━━
🆔 ID ЧАСТЕЙ
━━━━━━━━━━━━━━━━━━━━

"""
                        for o in created_orders:
                            msg += f"Часть {o['part_num']}: #{o['smm_id']} ({o['amount']} шт)\n"
                        
                        msg += f"""
━━━━━━━━━━━━━━━━━━━━
📋 КОМАНДЫ
━━━━━━━━━━━━━━━━━━━━

#{status} [ID] - статус части
#рефилл [ID] - дозаполнение

━━━━━━━━━━━━━━━━━━━━
⏱ Срок: 1 мин - 96 часов
🔴 Отмена была доступна только ДО подтверждения
━━━━━━━━━━━━━━━━━━━━"""
                        
                        if failed_parts > 0:
                            msg += f"\n\n⚠️ Не удалось создать {failed_parts} частей"
                        
                        c.send_message(order['chat_id'], msg)
                        
                        if settings.get("set_alert_neworder", False):
                            try:
                                c.telegram.send_notification(
                                    f"📊 РАЗДЕЛЁННЫЙ ЗАКАЗ\n"
                                    f"🆔 {order['OrderID']}\n"
                                    f"📦 {total_amount} → {len(created_orders)} частей"
                                )
                            except:
                                pass
                    else:
                        c.send_message(order['chat_id'], "❌ Не удалось создать части.\n💰 Деньги возвращены.")
                        try:
                            c.account.refund(order['OrderID'])
                        except:
                            pass
                else:
                    # Не удалось извлечь лимит - обычная ошибка
                    c.send_message(order['chat_id'], f"❌ Ошибка: {smm_order_id}")
                    if settings.get("set_refund_smm", False):
                        try:
                            c.account.refund(order['OrderID'])
                        except:
                            pass
            
            elif isinstance(smm_order_id, (int, str)) and str(smm_order_id).isdigit():
                # Обычный успешный заказ
                try:
                    orders = load_orders()
                    orders[smm_order_id] = {
                        "service_id": order['service_id'],
                        "chat_id": order['chat_id'],
                        "order_id": order['OrderID'],
                        "order_url": order['url'],
                        "order_amount": order['Amount'],
                        "partial_amount": 0,
                        "orderdatetime": order['OrderDateTime'],
                        "status": "pending"
                    }
                    save_orders(orders)
                    if settings.get("set_alert_neworder", False):
                        send_order_info(c, order, int(smm_order_id), api_url, api_key)
                except Exception as e:
                    logger.error(f"{e}")
                
                status = 'статус' if order.get('api_type') == 'API_1' else 'инфо'
                c.send_message(order['chat_id'], 
                            f"""━━━━━━━━━━━━━━━━━━━━
✅ ЗАКАЗ СОЗДАН
━━━━━━━━━━━━━━━━━━━━

🆔 ID: {smm_order_id}
📊 Статус: В обработке
⏱ Срок: 1 мин - 96 часов

━━━━━━━━━━━━━━━━━━━━
📋 КОМАНДЫ
━━━━━━━━━━━━━━━━━━━━

#{status} {smm_order_id} - статус
#рефилл {smm_order_id} - дозаполнение  

━━━━━━━━━━━━━━━━━━━━
🔴 Отмена была доступна только ДО подтверждения
━━━━━━━━━━━━━━━━━━━━""")
            else:
                # Другая ошибка
                c.send_message(order['chat_id'], f"❌ Ошибка при создании заказа: {smm_order_id}")
                if settings.get("set_alert_errororder", False):
                    send_order_error_info(c, smm_order_id, order)
                if settings.get("set_alert_smmbalance", False):
                    send_smm_balance_info(c)
                if settings.get("set_refund_smm", False):
                    try:
                        c.account.refund(order['OrderID'])
                    except Exception as e:
                        logger.error(f"Ошибка при возврате средств: {e}")
        else:
            if text.strip() == "-":
                c.send_message(chat_id, "❌ Заказ отменен.\n")
                try:
                    c.account.refund(order['OrderID'])
                except Exception as e:
                    pass
        orders_data.remove(order) 
        save_payorders(orders_data)

# телеграм
def send_order_info(c: Cardinal, order: Dict, smm_order_id: int, api_url, api_key) -> None:
    """
    Инфа в телеграм о новом заказе
    """
    def getcoingaterate(fromcurrency='USD', tocurrency='RUB'):
        try:
            url = f'https://api.coingate.com/v2/rates/merchant/{fromcurrency}/{tocurrency}'
            response = requests.get(url)
            response.raise_for_status()
            return float(response.text)
        except Exception as e:
            logger.error(f"Ошибка при получении курса: {e}")
            return None
    
    fp_balance = c.get_balance()
    price_smm_order = SocTypeAPI.get_order_status(smm_order_id, api_url, api_key)['charge']
    currency = SocTypeAPI.get_order_status(smm_order_id, api_url, api_key)['currency']
    
    smm_balance_info = SocTypeAPI.get_balance(api_url, api_key)
    balance, currency = smm_balance_info
    
    fp_currency = order['OrderCurrency']
    if fp_currency == '₽':
        if currency == 'RUB':
            pass
        elif currency == 'USD':
            price_smm_order = float(price_smm_order) * float(getcoingaterate('USD', 'RUB'))
    elif fp_currency == '$':
        if currency == 'RUB':
            price_smm_order = float(price_smm_order) * float(getcoingaterate('RUB', 'USD'))
        elif currency == 'USD':
            pass
    sum_order = float(order['OrderPrice']) - float(price_smm_order)
    sum_order_6com = sum_order * 0.94  # 6% комиссию за вывод
    sum_order_3com = sum_order * 0.97  # 3% комиссию за вывод
    
    order_info = (
        f"<b>✅ Создан заказ <code>{NAME}</code>:</b> <code>{order['Order']}</code>\n\n"
        f"<b><i>🙍‍♂️ Покупатель:</i></b> <code>{order['buyer']}</code>\n\n"
        f"<b><i>💵 Сумма заказа:</i></b> <code>{order['OrderPrice']} {fp_currency}</code>\n"
        f"<b><i>💵 Потрачено:</i></b> <code>{price_smm_order} {currency}</code>\n"
        f"<b><i>💵 Прибыль:</i></b> {order['OrderPrice']}-{price_smm_order} = <code>{sum_order:.2f}</code>\n"
        f"<b><i>💵 Прибыль с комисией:</i></b> <code>{sum_order_6com:.2f} 6% {sum_order_3com:.2f} 3%</code>\n"
        f"<b><i>💰 Остаток на балансе:</i></b> <code>{balance:.2f} {currency}</code>\n"
        f"<b><i>💰 Баланс на FunPay:</i></b> <code>{fp_balance.total_rub}₽, {fp_balance.available_usd}$, {fp_balance.total_eur}€</code>\n\n"
        f"<b><i>📇 ID заказа на FunPay:</i></b> <code>{order['OrderID']}</code>\n"
        f"<b><i>🆔 ID заказа на сайте:</i></b> <code>{smm_order_id}</code>\n"
        f"<b><i>🔍 Сервис ID:</i></b> <code>{order['service_id']}</code>\n"
        f"<b><i>🔢 Кол-во:</i></b> <code>{order['Amount']}</code>\n"
        f"<b><i>🔗 Ссылка:</i></b> {order['url'].replace('https://', '')}\n\n"
    )

    button = InlineKeyboardButton(text="🌐 Открыть страницу заказа", url=f"https://funpay.com/orders/{order['OrderID']}/")
    keyboard = InlineKeyboardMarkup().add(button)

    try:
        users = load_authorized_users()
        if not users:
            return
        
        for user_id in users:
            c.telegram.bot.send_message(
                user_id,
                order_info,
                parse_mode='HTML',
                reply_markup=keyboard,
                disable_web_page_preview=True
            )
    except Exception as e:
        logging.error(e)

def send_order_error_info(c: Cardinal, text: Dict, order: Dict) -> None:
    """
    Инфа в телеграм о ошибке при создании заказа
    """
    text_error = (
        f"<b>❌ Ошибка при создании заказа <code>{NAME}</code> #{order['OrderID']}:</b> <code>{text}</code>\n\n"
    )
    
    button = InlineKeyboardButton(text="🌐 Открыть страницу заказа", url=f"https://funpay.com/orders/{order['OrderID']}/")
    keyboard = InlineKeyboardMarkup().add(button)
    
    try:
        users = load_authorized_users()
        if not users:
            return
        
        for user_id in users:
            c.telegram.bot.send_message(
                user_id,
                text_error,
                parse_mode='HTML',
                reply_markup=keyboard,
                disable_web_page_preview=True
            )
    except Exception as e:
        logging.error(e)
    
def send_smm_balance_info(c: Cardinal) -> None:
    """
    Инфа в телеграм о балансе смм
    """
    try:
        fp_balance = c.get_balance()
        api_url = get_api_url()
        api_key = get_api_key()
        smm_balance_info = SocTypeAPI.get_balance(api_url, api_key)
        balance, currency = smm_balance_info
        api_url_2 = get_api_url('2')
        api_key_2 = get_api_key('2')
        smm_balance_info_2 = SocTypeAPI.get_balance(api_url_2, api_key_2)
        balance_2, currency_2 = smm_balance_info_2
        
        text_balance = (
            f"<b>💰 Баланс {api_url.replace('https://', '').replace('/api/v2/', '').replace('/api/v2', '')}:</b> <code>{balance:.2f} {currency}</code>\n"
            f"<b>💰 Баланс {api_url_2.replace('https://', '').replace('/api/v2/', '').replace('/api/v2', '')}:</b> <code>{balance_2:.2f} {currency_2}</code>\n"
            f"<b>💰 Баланс на FunPay:</b> <code>{fp_balance.total_rub}₽, {fp_balance.available_usd}$, {fp_balance.total_eur}€</code>"
        )
    except Exception as e:
        api_url = get_api_url()
        api_key = get_api_key()
        smm_balance_info = SocTypeAPI.get_balance(api_url, api_key)
        balance, currency = smm_balance_info
        
        text_balance = (
            f"<b>💰 Баланс сайта:</b> <code>{balance:.2f} {currency}</code>\n"
            f"<b>💰 Балансе на FunPay:</b> <code>{fp_balance.total_rub}₽, {fp_balance.available_usd}$, {fp_balance.total_eur}€</code>"
        )
    
    try:
        users = load_authorized_users()
        if not users:
            return
        
        for user_id in users:
            c.telegram.bot.send_message(
                user_id,
                text_balance,
                parse_mode='HTML'
            )
    except Exception as e:
        logging.error(e)
    
    
def get_autosmm_promo_keyboard() -> InlineKeyboardMarkup:
    keyboard = InlineKeyboardMarkup(row_width=1)
    keyboard.add(InlineKeyboardButton("🟢 Дешевый SMM сервис | Скидка на заказы до 50% по данной ссылке", url="https://twiboost.com/ref3937743"))
    return keyboard


def send_smm_start_info(c: Cardinal) -> None:
    """
    Инфа в телеграм при старте бота
    """
    text_start = (
        f"<b><u>✅ Авто-накрутка инициализирована!</u></b>\n\n"
        f"<b><i>ℹ️ Версия:</i></b> <code>{VERSION}</code>\n"
        f"<b><i>⚙️ Настройки /autosmm</i></b>\n\n"
        f"<i>ℹ️ Авто-накрутка</i>"
    )
    
    try:
        users = load_authorized_users()
        if not users:
            return
        
        for user_id in users:
            c.telegram.bot.send_message(
                user_id,
                text_start,
                parse_mode='HTML',
                reply_markup=get_autosmm_promo_keyboard()
            )
    except Exception as e:
        logging.error(e)
###
    
    
class SocTypeAPI:
    @staticmethod
    def create_order(service_id: int, link: str, quantity: int, api_url: str, api_key: str) -> Optional[int]:
        url = f"{api_url}?action=add&service={service_id}&link={link}&quantity={quantity}&key={api_key}"
        try:
            response = requests.get(url)
            response.raise_for_status() 

            response_json = response.json()

            if "order" in response_json:
                return response_json["order"]
            elif "error" in response_json:
                return response_json["error"]
            else:
                logger.error(f"Ошибка при создании заказа: {response_json}")
                return "Неизвестная ошибка при создании заказа."
        
        except requests.exceptions.HTTPError as http_err:
            try:
                error_response = response.json()
                if "error" in error_response:
                    return error_response["error"]
            except ValueError:
                logger.error(f"HTTP ошибка: {http_err} - Ответ от сервера: {response.text}")
                return "Неизвестная ошибка при создании заказа."

    @staticmethod
    def get_order_status(order_id: int, api_url: str, api_key: str) -> Optional[dict]:
        url = f"{api_url}?action=status&order={order_id}&key={api_key}"
        try:
            response = requests.get(url)
            response.raise_for_status()
            order_status = response.json()
            return order_status
        except requests.exceptions.RequestException as ex:
            logger.error(f"Error getting SocType order status: {ex}")
            return None

    @staticmethod
    def refill_order(order_id: int, api_url: str, api_key: str) -> Optional[str]:
        url = f"{api_url}?action=refill&order={order_id}&key={api_key}"
        try:
            response = requests.get(url)
            response.raise_for_status()
            refill = response.json().get("refill")
            return refill
        except requests.exceptions.RequestException as ex:
            logger.error(f"Error refilling SocType order: {ex}")
            return None

    @staticmethod
    def get_balance(api_url: str, api_key: str):
        url = f"{api_url}?action=balance&key={api_key}"
        try:
            response = requests.get(url)
            response.raise_for_status()
            balance_data: Dict[str, str] = response.json()
            
            pattern = r'\d+\.\d+'
            match = re.search(pattern, balance_data['balance'])
            
            if match:
                balance = float(match.group())
                currency = balance_data['currency']
                return balance, currency
            else:
                raise ValueError("Invalid balance format")
        except Exception as e:
            logger.error(f"Ошибка при получения баланса сайта, возможно неверный API: {e}")
            return None

    @staticmethod
    def cancel_order(order_id: int, api_url: str, api_key: str) -> Optional[str]:
        url = f"{api_url}?action=cancel&order={order_id}&key={api_key}"
        try:
            response = requests.get(url)
            response.raise_for_status()
            cancel_status = response.json().get("cancel")
            return cancel_status
        except requests.exceptions.RequestException as ex:
            logger.error(f"Error cancelling SocType order: {ex}")
            return None

# --------------------------------------------------------

def checkbox(cardinal: Cardinal):
    threading.Thread(target=process_orders, args=[cardinal]).start()

# Проверка статусов заказов
def process_orders(c: Cardinal):
    while True:
        logger.info("Проверка статусов заказов...")
        
        
        
        api_url = get_api_url()
        api_key = get_api_key()
        
        def check_order_status(order_id: str) -> dict:
            url = f"{api_url}?action=status&order={order_id}&key={api_key}"
            try:
                response = requests.get(url)
                response.raise_for_status() 
                return response.json()
            except requests.exceptions.RequestException as e:
                print(f"Ошибка при получении статуса заказа {order_id}: {e}")
                return {}

        def check_and_send_message(orders: dict, status: str, send_message_func):
            orders_to_delete = []
            for order_id, order_info in orders.items():
                if order_info["status"] == status:
                    logger.info(f"Заказ {order_id} {status.lower()}!")
                    send_message_func(c, order_id)
                    orders_to_delete.append(order_id)
            return orders_to_delete

        def send_completion_message(c: Cardinal, order_id: str):
            orders = load_orders()
            chat_id = orders[order_id]["chat_id"]
            
            # Проверяем является ли это частью разделённого заказа
            if orders[order_id].get('is_part'):
                # Это часть разделённого заказа
                part_num = orders[order_id].get('part_number')
                total_parts = orders[order_id].get('total_parts')
                order_amount = orders[order_id].get('order_amount')
                funpay_order_id = orders[order_id]['order_id']
                
                # Проверяем выполнены ли все части
                completed_parts = 0
                for oid, odata in orders.items():
                    if odata.get('order_id') == funpay_order_id and odata.get('is_part'):
                        if odata.get('status') == 'Completed':
                            completed_parts += 1
                
                if completed_parts == total_parts:
                    # ВСЕ части выполнены!
                    message_text = f"""━━━━━━━━━━━━━━━━━━━━
🎉 ВСЕ ЧАСТИ ВЫПОЛНЕНЫ
━━━━━━━━━━━━━━━━━━━━

Заказ #{funpay_order_id} готов!

━━━━━━━━━━━━━━━━━━━━
✅ ПОДТВЕРДИТЕ ЗАКАЗ
━━━━━━━━━━━━━━━━━━━━

1. Перейдите по ссылке:
funpay.com/orders/{funpay_order_id}/

2. Нажмите кнопку
"Подтвердить выполнение"

━━━━━━━━━━━━━━━━━━━━
💬 Остались вопросы? Пишите в чат!
━━━━━━━━━━━━━━━━━━━━"""
                else:
                    # Часть выполнена, но не все
                    message_text = f"""━━━━━━━━━━━━━━━━━━━━
✅ ЧАСТЬ {part_num}/{total_parts} ВЫПОЛНЕНА
━━━━━━━━━━━━━━━━━━━━

📊 Выполнено: {order_amount} шт
🆔 ID части: {order_id}

━━━━━━━━━━━━━━━━━━━━
⏳ ПРОГРЕСС
━━━━━━━━━━━━━━━━━━━━

Готово: {completed_parts}/{total_parts}
Осталось: {total_parts - completed_parts}

━━━━━━━━━━━━━━━━━━━━
💡 Проверить: #статус [ID]
━━━━━━━━━━━━━━━━━━━━"""
            else:
                # Обычный заказ
                message_text = f"""━━━━━━━━━━━━━━━━━━━━
✅ ЗАКАЗ ВЫПОЛНЕН
━━━━━━━━━━━━━━━━━━━━

Заказ #{orders[order_id]['order_id']} готов!

━━━━━━━━━━━━━━━━━━━━
✅ ПОДТВЕРДИТЕ ЗАКАЗ
━━━━━━━━━━━━━━━━━━━━

1. Перейдите по ссылке:
funpay.com/orders/{orders[order_id]['order_id']}/

2. Нажмите кнопку
"Подтвердить выполнение"

━━━━━━━━━━━━━━━━━━━━
💬 Остались вопросы? Пишите в чат!
━━━━━━━━━━━━━━━━━━━━"""
            
            try:
                c.send_message(chat_id, message_text)
            except Exception as e:
                logger.error(f"{e}")

        def send_canceled_message(c: Cardinal, order_id: str):
            orders = load_orders()
            chat_id = orders[order_id]["chat_id"]
            
            # Проверяем является ли это частью разделённого заказа
            if orders[order_id].get('is_part'):
                part_num = orders[order_id].get('part_number')
                total_parts = orders[order_id].get('total_parts')
                
                message_text = f"❌ Часть {part_num}/{total_parts} заказа #{orders[order_id]['order_id']} отменена!\n"
                message_text += f"🆔 ID части: {order_id}\n\n"
                message_text += f"💬 Напишите в поддержку для решения проблемы."
            else:
                message_text = f"❌ Заказ #{orders[order_id]['order_id']} отменён!"
            
            try:
                c.send_message(chat_id, message_text)
                # Возврат делаем только если это НЕ часть
                if not orders[order_id].get('is_part'):
                    try:
                        c.account.refund(orders[order_id]['order_id'])
                    except Exception as e:
                        pass
            except Exception as e:
                logger.error(f"{e}")
                
        def send_partial_message(c: Cardinal, order_id: str):
            settings = load_settings()
            orders = load_orders()
            cashlist = load_cashlist()
            chat_id = orders[order_id]["chat_id"]
            partial_amount = int(orders[order_id].get('partial_amount', 0))
            new_service_id = orders[order_id]['service_id']
            new_link = orders[order_id]['order_url']
            order_fid = orders[order_id]['order_id']
            orderdatatime = orders[order_id]['orderdatetime']
            try:
                smm_order_id = SocTypeAPI.create_order(new_service_id, new_link, partial_amount, api_url, api_key)

                if smm_order_id is not None:
                    if settings.get("set_recreated_order", False):
                        cashlist[smm_order_id] = {
                            "service_id": new_service_id,
                            "chat_id": chat_id,
                            "order_id": order_fid,
                            "order_url": new_link,
                            "order_amount": partial_amount,
                            "partial_amount": 0,
                            "orderdatetime": orderdatatime,
                            "status": "new"
                        }
                        save_cashlist(cashlist)
                        c.send_message(chat_id, f"""
                                    📈 Ваш заказ #{order_fid} был пересоздан!
                                    🆔 Новый ID заказа: {smm_order_id}
                                    ⏳ Остаток выполнения: {partial_amount}
                                    """)
                else:
                    c.send_message(chat_id, f"""
                                🔴 Заказ #{order_fid} был приостановлен!
                                ⏳ Остаток выполнения: {partial_amount}
                                """)             
            except Exception as e:
                logger.error(f"{e}")

        orders = load_orders()
        updated_orders = {}

        for order_id, order_info in orders.items():
            order_status = check_order_status(order_id)
            if order_status:
                updated_orders[order_id] = {
                    "service_id": order_info['service_id'],
                    "chat_id": order_info["chat_id"],
                    "order_id": order_info["order_id"],
                    "order_url": order_info['order_url'],
                    "order_amount": order_info['order_amount'],
                    "partial_amount": int(order_status.get("remains", 0)),
                    "orderdatetime": order_info['orderdatetime'],
                    "status": order_status.get("status", "unknown"),
                    # СОХРАНЯЕМ МЕТАДАННЫЕ ЧАСТЕЙ!
                    "is_part": order_info.get('is_part', False),
                    "part_number": order_info.get('part_number'),
                    "total_parts": order_info.get('total_parts')
                }
        save_orders(updated_orders)

        completed_orders = check_and_send_message(updated_orders, "Completed", send_completion_message)
        canceled_orders = check_and_send_message(updated_orders, "Canceled", send_canceled_message)
        partial_orders = check_and_send_message(updated_orders, "Partial", send_partial_message)

        # удаление заказов после отправки сообщений
        for order_id in completed_orders + canceled_orders + partial_orders:
            if order_id in updated_orders:
                del updated_orders[order_id]
        save_orders(updated_orders)
        
        # обновление списка orders с данными из cashlist
        cashlist = load_cashlist()
        for order_id, order_info in cashlist.items():
            if order_id not in updated_orders:
                updated_orders[order_id] = order_info
        save_orders(updated_orders)
        cashlist.clear()
        save_cashlist(cashlist)
        
        logger.info("Проверка статусов заказов завершена. Сплю минуту..")
        time.sleep(60)

    
def init_commands(cardinal: Cardinal, *args):
    # старт сообщение
    settings = load_settings()
    if settings.get("set_start_mess", False):
        send_smm_start_info(cardinal)
        
    if not cardinal.telegram:
        return
    tg = cardinal.telegram
    bot = tg.bot

    def send_smm_balance_command(m: types.Message):
        text_balance = send_smm_balance_info(cardinal)
        try:
            bot.reply_to(m, text_balance, parse_mode='HTML')
        except:
            pass

    settings_smm_keyboard = InlineKeyboardMarkup(row_width=1)
    set_api = InlineKeyboardButton("🔗 API URL", callback_data='set_api')
    set_api_key = InlineKeyboardButton("🔐 API KEY", callback_data='set_api_key')
    set_api_2 = InlineKeyboardButton("🔗 API URL 2", callback_data='set_api_2')
    set_api_key_2 = InlineKeyboardButton("🔐 API KEY 2", callback_data='set_api_key_2')
    set_usersm_settings = InlineKeyboardButton("🛠 Настройки", callback_data='set_usersm_settings')
    pay_orders = InlineKeyboardButton("📝 Оплаченные заказы", callback_data='pay_orders')
    active_orders = InlineKeyboardButton("📋 Активные заказы", callback_data='active_orders')
    promo_button = InlineKeyboardButton("🟢 Дешевый SMM сервис | Скидка на заказы до 50% по данной ссылке", url="https://twiboost.com/ref3937743")
    settings_smm_keyboard.row(set_api, set_api_key)
    settings_smm_keyboard.row(set_api_2, set_api_key_2)
    settings_smm_keyboard.add(set_usersm_settings, pay_orders, active_orders)
    settings_smm_keyboard.add(promo_button)
        
    def update_alerts_keyboard():
        alerts_smm_keyboard = InlineKeyboardMarkup(row_width=1)
        if settings.get("set_alert_neworder", False):
            set_alert_neworder = InlineKeyboardButton("🔔 Увед. о созданом заказе", callback_data='set_alert_neworder')
        else:
            set_alert_neworder = InlineKeyboardButton("🔕 Увед. о созданом заказе", callback_data='set_alert_neworder')
        if settings.get("set_alert_errororder", False):
            set_alert_errororder = InlineKeyboardButton("🔔 Увед. при ошибке создания", callback_data='set_alert_errororder')
        else:
            set_alert_errororder = InlineKeyboardButton("🔕 Увед. при ошибке создания", callback_data='set_alert_errororder')
        if settings.get("set_alert_smmbalance_new", False):
            set_alert_smmbalance_new = InlineKeyboardButton("🔔 Увед. о балансе смм до создания", callback_data='set_alert_smmbalance_new')
        else:
            set_alert_smmbalance_new = InlineKeyboardButton("🔕 Увед. о балансе смм до создания", callback_data='set_alert_smmbalance_new')
        if settings.get("set_alert_smmbalance", False):
            set_alert_smmbalance = InlineKeyboardButton("🔔 Увед. о балансе смм после создания", callback_data='set_alert_smmbalance')
        else:
            set_alert_smmbalance = InlineKeyboardButton("🔕 Увед. о балансе смм после создания", callback_data='set_alert_smmbalance')
        if settings.get("set_refund_smm", False):
            set_refund_smm = InlineKeyboardButton("🟢 Автовозврат", callback_data='set_refund_smm')
        else:
            set_refund_smm = InlineKeyboardButton("🔴 Автовозврат", callback_data='set_refund_smm')
        if settings.get("set_start_mess", False):
            set_start_mess = InlineKeyboardButton("🟢 Сообщение при запуске FPC", callback_data='set_start_mess')
        else:
            set_start_mess = InlineKeyboardButton("🔴 Сообщение при запуске FPC", callback_data='set_start_mess')
        if settings.get("set_tg_private", False):
            set_tg_private = InlineKeyboardButton("🟢 Закрытые ТГ каналы/группы", callback_data='set_tg_private')
        else:
            set_tg_private = InlineKeyboardButton("🔴 Закрытые ТГ каналы/группы", callback_data='set_tg_private')
        if settings.get("set_recreated_order", False):
            set_recreated_order = InlineKeyboardButton("🟢 Пересоздание заказа", callback_data='set_recreated_order')
        else:
            set_recreated_order = InlineKeyboardButton("🔴 Пересоздание заказа", callback_data='set_recreated_order')
        set_back_butt = InlineKeyboardButton("⬅️ Назад", callback_data='set_back_butt')
        alerts_smm_keyboard.add(set_alert_neworder, set_alert_errororder, set_alert_smmbalance_new, set_alert_smmbalance, set_refund_smm, set_start_mess, set_tg_private, set_recreated_order, set_back_butt)
        
        return alerts_smm_keyboard

    def send_settings(m: types.Message):
        bot.reply_to(m, "API 1: <code>ID:</code>\nAPI 2: <code>ID2:</code>\n\n⚙️ AutoSmm:", reply_markup=settings_smm_keyboard)

    def edit(call: telebot.types.CallbackQuery):
        text_sett_uss = f"🛠 Настройки:"
        if call.data == 'set_usersm_settings':
            bot.edit_message_text(
                chat_id=call.message.chat.id,
                message_id=call.message.message_id,
                text=text_sett_uss,
                reply_markup=update_alerts_keyboard()
            )
        
        # кнопки настроек
        elif call.data == 'set_alert_neworder':
            settings['set_alert_neworder'] = not settings['set_alert_neworder']
            save_settings(settings)

            new_markup = update_alerts_keyboard()
            bot.edit_message_reply_markup(
                chat_id=call.message.chat.id,
                message_id=call.message.message_id,
                reply_markup=new_markup
            )
        elif call.data == 'set_alert_errororder':
            settings['set_alert_errororder'] = not settings['set_alert_errororder']
            save_settings(settings)

            new_markup = update_alerts_keyboard()
            bot.edit_message_reply_markup(
                chat_id=call.message.chat.id,
                message_id=call.message.message_id,
                reply_markup=new_markup
            )
        elif call.data == 'set_alert_smmbalance_new':
            settings['set_alert_smmbalance_new'] = not settings['set_alert_smmbalance_new']
            save_settings(settings)

            new_markup = update_alerts_keyboard()
            bot.edit_message_reply_markup(
                chat_id=call.message.chat.id,
                message_id=call.message.message_id,
                reply_markup=new_markup
            )
        elif call.data == 'set_alert_smmbalance':
            settings['set_alert_smmbalance'] = not settings['set_alert_smmbalance']
            save_settings(settings)

            new_markup = update_alerts_keyboard()
            bot.edit_message_reply_markup(
                chat_id=call.message.chat.id,
                message_id=call.message.message_id,
                reply_markup=new_markup
            )
        elif call.data == 'set_refund_smm':
            settings['set_refund_smm'] = not settings['set_refund_smm']
            save_settings(settings)

            new_markup = update_alerts_keyboard()
            bot.edit_message_reply_markup(
                chat_id=call.message.chat.id,
                message_id=call.message.message_id,
                reply_markup=new_markup
            )
        elif call.data == 'set_start_mess':
            settings['set_start_mess'] = not settings['set_start_mess']
            save_settings(settings)

            new_markup = update_alerts_keyboard()
            bot.edit_message_reply_markup(
                chat_id=call.message.chat.id,
                message_id=call.message.message_id,
                reply_markup=new_markup
            )
        elif call.data == 'set_auto_refill':
            settings['set_auto_refill'] = not settings['set_auto_refill']
            save_settings(settings)

            new_markup = update_alerts_keyboard()
            bot.edit_message_reply_markup(
                chat_id=call.message.chat.id,
                message_id=call.message.message_id,
                reply_markup=new_markup
            )
        elif call.data == 'set_tg_private':
            if 'set_tg_private' not in settings:
                settings['set_tg_private'] = True
            else:
                settings['set_tg_private'] = not settings['set_tg_private']
            save_settings(settings)
            
            new_markup = update_alerts_keyboard()
            bot.edit_message_reply_markup(
                chat_id=call.message.chat.id,
                message_id=call.message.message_id,
                reply_markup=new_markup
            )
        elif call.data == 'set_recreated_order':
            if 'set_recreated_order' not in settings:
                settings['set_recreated_order'] = True
            else:
                settings['set_recreated_order'] = not settings['set_recreated_order']
            save_settings(settings)
            
            new_markup = update_alerts_keyboard()
            bot.edit_message_reply_markup(
                chat_id=call.message.chat.id,
                message_id=call.message.message_id,
                reply_markup=new_markup
            )

            
        elif call.data == 'set_back_butt':
            bot.edit_message_text(
                chat_id=call.message.chat.id,
                message_id=call.message.message_id,
                text="API 1: <code>ID:</code>\nAPI 2: <code>ID2:</code>\n\n⚙️ AutoSmm:",
                reply_markup=settings_smm_keyboard
            )
    
        elif call.data == 'set_api':
            back_button = InlineKeyboardButton("❌ Отмена", callback_data='delete_back_butt')
            kb = InlineKeyboardMarkup().add(back_button)
            current_value = settings.get('api_url', 'Не установлено')
            result = bot.send_message(call.message.chat.id,
                                    f'<b>Текущее значение URL: </b><span class="tg-spoiler">{current_value}</span>\n\n'
                                    '<i>⬇️ Введите новое значение ⬇️</i>',
                                    reply_markup=kb)

            result = tg.set_state(
                chat_id=call.message.chat.id,
                message_id=result.id,
                user_id=call.from_user.id,
                state="setting_url"
            )
        elif call.data == 'set_api_key':
            back_button = InlineKeyboardButton("❌ Отмена", callback_data='delete_back_butt')
            kb = InlineKeyboardMarkup().add(back_button)
            current_value = settings.get('api_key', 'Не установлен')
            result = bot.send_message(call.message.chat.id,
                                    f'<b>Текущее значение API KEY: </b><span class="tg-spoiler">{current_value}</span>\n\n'
                                    '<i>⬇️ Введите новое значение ⬇️</i>',
                                    reply_markup=kb)
            result = tg.set_state(
                chat_id=call.message.chat.id,
                message_id=result.id,
                user_id=call.from_user.id,
                state="setting_api_key"
            )
    
        elif call.data == 'set_api_2':
            back_button = InlineKeyboardButton("❌ Отмена", callback_data='delete_back_butt')
            kb = InlineKeyboardMarkup().add(back_button)
            current_value = settings.get('api_url_2', 'Не установлено')
            result = bot.send_message(call.message.chat.id,
                                    f'<b>Текущее значение URL 2: </b><span class="tg-spoiler">{current_value}</span>\n\n'
                                    '<i>⬇️ Введите новое значение ⬇️</i>',
                                    reply_markup=kb)

            result = tg.set_state(
                chat_id=call.message.chat.id,
                message_id=result.id,
                user_id=call.from_user.id,
                state="setting_url_2"
            )
        elif call.data == 'set_api_key_2':
            back_button = InlineKeyboardButton("❌ Отмена", callback_data='delete_back_butt')
            kb = InlineKeyboardMarkup().add(back_button)
            current_value = settings.get('api_key_2', 'Не установлен')
            result = bot.send_message(call.message.chat.id,
                                    f'<b>Текущее значение API KEY 2: </b><span class="tg-spoiler">{current_value}</span>\n\n'
                                    '<i>⬇️ Введите новое значение ⬇️</i>',
                                    reply_markup=kb)
            result = tg.set_state(
                chat_id=call.message.chat.id,
                message_id=result.id,
                user_id=call.from_user.id,
                state="setting_api_key_2"
            )
        elif call.data == 'delete_back_butt':
            bot.delete_message(call.message.chat.id, call.message.message_id)
            tg.clear_state(call.message.chat.id, call.from_user.id)
        
        elif call.data == 'pay_orders':
            back_button = InlineKeyboardButton("⬅️ Назад", callback_data='delete_back_butt')
            kb = InlineKeyboardMarkup().add(back_button)
            orders_data = load_payorders()
            if not orders_data:
                orders_text = "📝 Оплаченные заказы отсутствуют."
                bot.send_message(call.message.chat.id, orders_text, reply_markup=kb)
                bot.answer_callback_query(call.id)
                return                
            orders_text = "📝 Оплаченные заказы:\n\n"
            for order in orders_data:
                orders_text += f"🆔 ID заказа: {order['OrderID']}\n"
                orders_text += f"⠀∟📋 Название: {order['Order']}\n"
                orders_text += f"⠀∟🔢 Кол-во: {order['Amount']}\n"
                orders_text += f"⠀∟👤 Покупатель: {order['buyer']}\n"
                orders_text += f"⠀∟📅 Дата: {order['OrderDateTime']}\n"
                orders_text += f"⠀∟🔗 Ссылка: {order['url']}\n\n"
            bot.send_message(call.message.chat.id, orders_text, reply_markup=kb)
        elif call.data == 'active_orders':
            back_button = InlineKeyboardButton("⬅️ Назад", callback_data='delete_back_butt')
            kb = InlineKeyboardMarkup().add(back_button)
            orders_data = load_orders()
            if not orders_data:
                orders_text = "📋 Активные заказы отсутствуют."
                bot.send_message(call.message.chat.id, orders_text, reply_markup=kb)
                bot.answer_callback_query(call.id)
                return
            orders_text = "📋 Активные заказы:\n\n"
            for order_id, order in orders_data.items():
                orders_text += f"🆔 ID заказа: {order_id}\n"
                orders_text += f"⠀∟🔢 Кол-во: {order['order_amount']}\n"
                orders_text += f"⠀∟📅 Дата: {order['orderdatetime']}\n"
                orders_text += f"⠀∟📋 Статус: {order['status']}\n\n"
            bot.send_message(call.message.chat.id, orders_text, reply_markup=kb)
        bot.answer_callback_query(call.id)

    def handle_text_input(message: telebot.types.Message):
        state_data = tg.get_state(message.chat.id, message.from_user.id)
        if state_data and 'state' in state_data:
            state = state_data['state']
            if state == 'setting_url':
                settings['api_url'] = message.text
                save_settings(settings)
                bot.send_message(message.from_user.id, f'✅ Успех: URL обновлён на: <span class="tg-spoiler">{message.text}</span>\n‼️ Напишите /restart')
                bot.delete_message(message.chat.id, message.message_id)
                bot.delete_message(message.chat.id, message.message_id - 1)
                logger.info(f'URL обновлён на: {message.text}')
            
            elif state == 'setting_api_key':
                settings['api_key'] = message.text
                save_settings(settings)
                bot.send_message(message.from_user.id, f'✅ Успех: API KEY обновлён на: <span class="tg-spoiler">{message.text}</span>\n‼️ Напишите /restart')
                bot.delete_message(message.chat.id, message.message_id)
                bot.delete_message(message.chat.id, message.message_id - 1)
                logger.info(f'API KEY обновлён на: {message.text}')
            
            elif state == 'setting_url_2':
                settings['api_url_2'] = message.text
                save_settings(settings)
                bot.send_message(message.from_user.id, f'✅ Успех: URL 2 обновлён на: <span class="tg-spoiler">{message.text}</span>\n‼️ Напишите /restart')
                bot.delete_message(message.chat.id, message.message_id)
                bot.delete_message(message.chat.id, message.message_id - 1)
                logger.info(f'URL 2 обновлён на: {message.text}')
            
            elif state == 'setting_api_key_2':
                settings['api_key_2'] = message.text
                save_settings(settings)
                bot.send_message(message.from_user.id, f'✅ Успех: API KEY 2 обновлён на: <span class="tg-spoiler">{message.text}</span>\n‼️ Напишите /restart')
                bot.delete_message(message.chat.id, message.message_id)
                bot.delete_message(message.chat.id, message.message_id - 1)
                logger.info(f'API KEY 2 обновлён на: {message.text}')
                
            tg.clear_state(message.chat.id, message.from_user.id)

    tg.cbq_handler(edit, lambda c: c.data in [
        'set_api',
        'set_api_key',
        'set_api_2',
        'set_api_key_2',
        'set_usersm_settings',
        'set_back_butt',
        'set_alert_neworder',
        'set_alert_errororder',
        'set_alert_smmbalance_new',
        'set_alert_smmbalance',
        'set_refund_smm',
        'set_auto_refill',
        'set_start_mess',
        'set_tg_private',
        'pay_orders',
        'active_orders',
        'set_recreated_order',
        'delete_back_butt'
        ])
    tg.msg_handler(
        handle_text_input, 
        func=lambda m: tg.check_state(m.chat.id, m.from_user.id, "setting_url") or 
                    tg.check_state(m.chat.id, m.from_user.id, "setting_api_key") or
                    tg.check_state(m.chat.id, m.from_user.id, "setting_url_2") or
                    tg.check_state(m.chat.id, m.from_user.id, "setting_api_key_2")
    )
    tg.msg_handler(send_settings, commands=["autosmm"])
    tg.msg_handler(send_smm_balance_command, commands=["check_balance"])
    cardinal.add_telegram_commands(UUID, [
        ("autosmm", f"настройки {NAME}", True),
        ("check_balance", f"баланс {NAME}", True)
    ])


BIND_TO_PRE_INIT = [init_commands]
BIND_TO_POST_INIT = [checkbox]
BIND_TO_NEW_ORDER = [bind_to_new_order]
BIND_TO_NEW_MESSAGE = [msg_hook]
BIND_TO_DELETE = None
