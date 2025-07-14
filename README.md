!pip install  pyTelegramBotAPI
!pip install telepot
!pip install --upgrade telepot
!pip install pillow
!apt-get install -y fonts-dejavu
!pip install qrcode[pil]
import telebot
from datetime import datetime, timedelta
import random
import os
from PIL import Image, ImageDraw, ImageFont
import requests
import qrcode

# Token del bot proporcionado por BotFather
TOKEN = '7020556961:AAE6fhjoXSHG3DhvylnQB8UsKYHOubEE0uA'

# Crear instancia del bot
bot = telebot.TeleBot(TOKEN)

# Opciones de trámites disponibles
trámites = {
    'Licenciatura': {
        '1': 'Solicitud de historial academico',
        '2': 'Solicitud de constancia de estudios',
        '3': 'Solicitud de constancia de creditos',
        '4': 'Solicitud de credencial nuevo ingreso',
        '5': 'Solicitud extemporanea de credencial',
        '6': 'Solicitud de baja temporal',
        '7': 'Solicitud de baja definitiva',
        '8': 'Solicitud de seguro facultativo',
        '9': 'Solicitud de corrección de datos',
    },
    'Posgrado': {
        '1': 'Solicitud de historial academico',
        '2': 'Solicitud de constancia de estudios',
        '3': 'Solicitud de credencial',
        '4': 'Solicitud de Seguro Facultativo',
        '5': 'Constancia de créditos y promedio',
        '6': 'Constancia de termino de estudios',
        '7': 'Solicitud de Corrección de datos',
    },
    'Egresado': {
        '1': 'Revisión de estudios de licenciatura',
        '2': 'Revisión de estudios de posgrado',
        '3': 'Solicitud de constancia de términos de estudio',
    }
}

# Variables temporales para almacenar la selección del usuario
grupo_temporal = None
trámite_temporal = None
unidad_académica = None
licenciatura = None
matrícula = None
nombre = None
fecha_hora_cita = None
ultimo_paso = None

# Manejar comando /start
@bot.message_handler(commands=['start'])
def handle_start(message):
    try:
        with open('/content/welcome_image.jpg', 'rb') as photo:
            bot.send_photo(message.chat.id, photo)
    except FileNotFoundError:
        bot.send_message(message.chat.id, "No se pudo encontrar la imagen de bienvenida.")

    respuesta = ("Bienvenido, soy OAFbot🤖 de la Universidad Rosario Castellanos. Te ayudaré con tus trámites administrativos.\n\n"
                 "Por favor, selecciona a qué estatus academico perteneces:\n"
                 "1. Licenciatura 📚🙇🏻\n"
                 "2. Posgrado 🧑🏻‍🏫👨🏻‍🏫\n"
                 "3. Egresado 👩🏻‍⚕️👨🏻‍⚕")
    bot.send_message(message.chat.id, respuesta)
    bot.register_next_step_handler(message, handle_grupo)

def handle_grupo(message):
    global grupo_temporal, ultimo_paso
    grupo_opciones = {'1': 'Licenciatura', '2': 'Posgrado', '3': 'Egresado'}
    grupo_seleccionado = grupo_opciones.get(message.text)

    if grupo_seleccionado:
        grupo_temporal = grupo_seleccionado
        ultimo_paso = handle_grupo
        opciones = "\n".join([f"{clave}. {valor}" for clave, valor in trámites[grupo_temporal].items()])
        respuesta = (f"Has seleccionado el estatus academico: {grupo_temporal}\n\n"
                     "Por favor, selecciona el trámite que deseas realizar:\n"
                     f"{opciones}\n\n"
                     "Para regresar al paso anterior, escribe 'Regresar'.")
        bot.send_message(message.chat.id, respuesta)
        bot.register_next_step_handler(message, handle_trámite)
    else:
        bot.send_message(message.chat.id, "Selección inválida. Por favor, selecciona un grupo válido:")
        bot.register_next_step_handler(message, handle_grupo)

def handle_trámite(message):
    global trámite_temporal, ultimo_paso
    if message.text.lower() == 'regresar':
        handle_start(message)
        return

    if message.text in trámites[grupo_temporal]:
        trámite_temporal = trámites[grupo_temporal][message.text]
        ultimo_paso = handle_trámite
        bot.send_message(message.chat.id, "Por favor, proporciona la siguiente información para generar el ticket:")
        bot.send_message(message.chat.id, "1. Unidad Académica:")
        bot.register_next_step_handler(message, handle_unidad_académica)
    else:
        bot.send_message(message.chat.id, "Selección de trámite inválida. Por favor, selecciona un trámite válido:")
        bot.register_next_step_handler(message, handle_trámite)

def handle_unidad_académica(message):
    global unidad_académica, ultimo_paso
    if message.text.lower() == 'regresar':
        handle_grupo(message)
        return

    unidad_académica = message.text
    ultimo_paso = handle_unidad_académica
    bot.send_message(message.chat.id, "Has seleccionado la Unidad Académica: " + unidad_académica)
    bot.send_message(message.chat.id, "2. Licenciatura:")
    bot.register_next_step_handler(message, handle_licenciatura)

def handle_licenciatura(message):
    global licenciatura, ultimo_paso
    if message.text.lower() == 'regresar':
        handle_trámite(message)
        return

    licenciatura = message.text
    ultimo_paso = handle_licenciatura
    bot.send_message(message.chat.id, "Has seleccionado la Licenciatura: " + licenciatura)
    bot.send_message(message.chat.id, "3. Matrícula:")
    bot.register_next_step_handler(message, handle_matrícula)

def handle_matrícula(message):
    global matrícula, ultimo_paso
    if message.text.lower() == 'regresar':
        handle_unidad_académica(message)
        return

    matrícula = message.text
    ultimo_paso = handle_matrícula
    bot.send_message(message.chat.id, "Has seleccionado la Matrícula: " + matrícula)
    bot.send_message(message.chat.id, "4. Nombre:")
    bot.register_next_step_handler(message, handle_nombre)

def handle_nombre(message):
    global nombre, ultimo_paso
    if message.text.lower() == 'regresar':
        handle_licenciatura(message)
        return

    nombre = message.text
    ultimo_paso = handle_nombre
    bot.send_message(message.chat.id, "Has seleccionado el Nombre: " + nombre)

    # Generar una fecha y hora aleatoria para la cita
    start_date = datetime(2024, 6, 17)
    end_date = datetime(2024, 12, 31)
    random_date = start_date + timedelta(days=random.randint(0, (end_date - start_date).days))
    random_time = datetime(random_date.year, random_date.month, random_date.day, random.randint(9, 17), random.randint(0, 59))
    global fecha_hora_cita
    fecha_hora_cita = random_time

    # Confirmar los datos
    confirmación = (f"Por favor, confirma que los datos proporcionados son correctos:\n\n"
                    f"Nombre: {nombre}\n"
                    f"Matrícula: {matrícula}\n"
                    f"Unidad Académica: {unidad_académica}\n"
                    f"Licenciatura: {licenciatura}\n"
                    f"Trámite Seleccionado: {trámite_temporal}\n"
                    f"Estatus: {grupo_temporal}\n\n"
                    "Escribe 'Confirmar' para proceder o 'Regresar' para corregir.")
    bot.send_message(message.chat.id, confirmación)
    bot.register_next_step_handler(message, handle_confirmación)

def handle_confirmación(message):
    if message.text.lower() == 'regresar':
        ultimo_paso(message)
        return

    if message.text.lower() == 'confirmar':
        # Generar el contenido del ticket
        contenido_ticket = (
            "****************************\n"
            "*       TICKET DE CITA       *\n"
            "****************************\n\n"
            "Información del usuario:\n"
            "------------------------\n"
            f"Nombre: {nombre}\n"
            f"Matrícula: {matrícula}\n"
            f"Unidad Académica: {unidad_académica}\n"
            f"Licenciatura: {licenciatura}\n"
            f"Trámite Seleccionado: {trámite_temporal}\n"
            f"Estatus: {grupo_temporal}\n\n"
            "------------------------\n"
            "Fecha y hora de la cita:\n"
            "------------------------\n"
            f"{fecha_hora_cita.strftime('%Y-%m-%d %H:%M')}\n\n"
            "****************************\n"
            "* Universidad Rosario Castellanos *\n"
            "****************************\n"
        )

        # Generar el ticket en formato de imagen
        def generar_imagen_ticket(contenido, archivo_imagen, logo):
            # Crear una imagen en blanco
            ancho_imagen = 400
            alto_imagen = 600
            imagen = Image.new("RGB", (ancho_imagen, alto_imagen), (255, 255, 255))

            # Cargar el logo
            logo_img = Image.open(logo).convert("RGBA")
            logo_img.thumbnail((100, 100))

            # Crear una imagen temporal para el logo con el mismo tamaño que la imagen de fondo
            logo_fondo = Image.new("RGBA", imagen.size, (255, 255, 255, 0))
            logo_x = (ancho_imagen - logo_img.width) // 2
            logo_y = 10
            logo_fondo.paste(logo_img, (logo_x, logo_y), logo_img)

            # Convertir la imagen del logo a modo 'RGB'
            logo_fondo = logo_fondo.convert("RGB")

            # Pegar el logo en el fondo
            imagen.paste(logo_fondo, (0, 0))

            # Crear el objeto para dibujar en la imagen
            draw = ImageDraw.Draw(imagen)

            # Especificar la fuente y el tamaño
            try:
                font_path = "/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf"
                font = ImageFont.truetype(font_path, 16)
            except IOError:
                font = ImageFont.load_default()

            # Calcular la posición del texto para centrarlo
            x = 20
            y = logo_y + logo_img.height + 20

            # Dibujar el contenido del ticket en la imagen
            for linea in contenido.split("\n"):
                draw.text((x, y), linea, font=font, fill=(0, 0, 0))
                y += 20

            # Guardar la imagen como archivo
            imagen.save(archivo_imagen)

        # Llamar a la función para generar la imagen del ticket
        logo_path = '/content/logo.png'  # Ruta del archivo de imagen del logo
        archivo_imagen = '/content/ticket.png'  # Ruta del archivo de imagen generado
        generar_imagen_ticket(contenido_ticket, archivo_imagen, logo_path)

        # Enviar la imagen del ticket al usuario
        with open(archivo_imagen, 'rb') as photo:
            bot.send_photo(message.chat.id, photo, caption="Aquí está tu ticket de cita:")

        # Generar un código QR con la información de la cita
        qr_info = (
            f"Nombre: {nombre}\n"
            f"Matrícula: {matrícula}\n"
            f"Unidad Académica: {unidad_académica}\n"
            f"Licenciatura: {licenciatura}\n"
            f"Trámite Seleccionado: {trámite_temporal}\n"
            f"Estatus: {grupo_temporal}\n"
            f"Fecha y hora de la cita: {fecha_hora_cita.strftime('%Y-%m-%d %H:%M')}"
        )
        qr = qrcode.QRCode(
            version=1,
            error_correction=qrcode.constants.ERROR_CORRECT_L,
            box_size=10,
            border=4,
        )
        qr.add_data(qr_info)
        qr.make(fit=True)

        qr_img = qr.make_image(fill_color="black", back_color="white")
        qr_img.save('/content/qr_code.png')

        # Enviar el código QR al usuario
        with open('/content/qr_code.png', 'rb') as qr_photo:
            bot.send_photo(message.chat.id, qr_photo, caption="Aquí está tu código QR con la información de la cita:")

    else:
        bot.send_message(message.chat.id, "Respuesta inválida. Por favor, escribe 'Confirmar' o 'Regresar'.")
        bot.register_next_step_handler(message, handle_confirmación)

# Iniciar el bot
bot.polling()
