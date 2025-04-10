import os
import telebot
import requests
import hashlib
import hmac
import base64
import time
import json

# Bot va ACRCloud ma'lumotlari
BOT_TOKEN = '8018904607:AAEyjexQXLon-TSSEHPXcBDNP20e5TZ9V3A'
ACR_HOST = 'identify-eu-west-1.acrcloud.com'
ACR_ACCESS_KEY = 'YOUR_ACCESS_KEY'
ACR_ACCESS_SECRET = 'YOUR_ACCESS_SECRET'

bot = telebot.TeleBot(BOT_TOKEN)

# Audio aniqlash funktsiyasi
def recognize_music(file_path):
    http_method = "POST"
    http_uri = "/v1/identify"
    data_type = "audio"
    signature_version = "1"
    timestamp = str(int(time.time()))

    string_to_sign = f"{http_method}\n{http_uri}\n{ACR_ACCESS_KEY}\n{data_type}\n{signature_version}\n{timestamp}"
    sign = base64.b64encode(hmac.new(ACR_ACCESS_SECRET.encode('ascii'), string_to_sign.encode('ascii'), digestmod=hashlib.sha1).digest()).decode('ascii')

    files = {
        'sample': open(file_path, 'rb'),
        'access_key': (None, ACR_ACCESS_KEY),
        'data_type': (None, data_type),
        'signature': (None, sign),
        'signature_version': (None, signature_version),
        'timestamp': (None, timestamp)
    }

    response = requests.post(f"http://{ACR_HOST}/v1/identify", files=files)
    return response.json()

# Voice yoki Audio qabul qilganda
@bot.message_handler(content_types=['voice', 'audio'])
def handle_audio(message):
    file_info = bot.get_file(message.voice.file_id if message.voice else message.audio.file_id)
    downloaded_file = bot.download_file(file_info.file_path)

    file_name = 'sample.ogg'
    with open(file_name, 'wb') as new_file:
        new_file.write(downloaded_file)

    bot.reply_to(message, "Musiqa aniqlanmoqda, iltimos kuting...")

    result = recognize_music(file_name)

    if 'metadata' in result and 'music' in result['metadata']:
        music = result['metadata']['music'][0]
        title = music.get('title', 'Noma\'lum')
        artist = music['artists'][0]['name'] if music.get('artists') else 'Noma\'lum ijrochi'
        bot.reply_to(message, f"Topildi:\n🎵 {title}\n👤 {artist}")
    else:
        bot.reply_to(message, "Kechirasiz, musiqa aniqlanmadi.")

    os.remove(file_name)

# Start komanda
@bot.message_handler(commands=['start'])
def start(message):
    bot.send_message(message.chat.id, "Salom! Menga ovozli yoki audio fayl yuboring, men sizga qo‘shiq nomini topib beraman.")

bot.polling()
