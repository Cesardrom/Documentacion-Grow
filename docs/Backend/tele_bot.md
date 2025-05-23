# Documentación de la Aplicación Tele Bot

## Introducción

La aplicación `tele_bot` gestiona la interacción con un bot de Telegram, permitiendo a los usuarios interactuar con el bot a través de comandos y recibir respuestas automatizadas.

## Configuración del Bot

### setup_bot

```python
from django.conf import settings
from telegram.ext import Application

def setup_bot():
    """
    Configura y devuelve la instancia de la aplicación del bot de Telegram.

    Este método crea una instancia de la aplicación del bot usando el token
    configurado en los ajustes, y añade los handlers para los comandos
    disponibles.

    Returns
    -------
    Application
        Instancia configurada de la aplicación del bot de Telegram.
    """
    application = Application.builder().token(settings.TELEGRAM_BOT_TOKEN).build()

    application.add_handler(start_handler)
    application.add_handler(hoy_handler)
    application.add_handler(semana_handler)
    application.add_handler(servicios_handler)

    return application
```

### Descripción

La función setup_bot configura la instancia del bot de Telegram utilizando el token de autenticación almacenado en la configuración del proyecto. También añade los manejadores (handlers) para los comandos disponibles, como /start, /hoy, /semana, y /servicios.

## Utilidades del Bot

### send_message

```python
import requests

def send_message(chat_id, text):
    """
    Envía un mensaje al chat de Telegram.

    Este método utiliza la API de Telegram para enviar un mensaje al
    chat especificado por el ID.

    Parameters
    ----------
    chat_id : int
        El ID del chat al que se enviará el mensaje.
    text : str
        El texto del mensaje que se enviará.

    Returns
    -------
    dict
        Respuesta JSON de la API de Telegram con los detalles del mensaje enviado.
    """
    url = f'https://api.telegram.org/bot{TELEGRAM_TOKEN}/sendMessage'
    params = {'chat_id': chat_id, 'text': text}

    print(f'Enviando mensaje a {chat_id}: {text}')  # Esto ayudará a ver los datos que estás enviando

    response = requests.get(url, params=params)

    print(f'Respuesta de Telegram: {response.json()}')  # Ver la respuesta completa para detectar errores

    return response.json()
```

...

### Descripción

La función send_message envía un mensaje al chat de Telegram especificado por el chat_id. Utiliza la API de Telegram para enviar el mensaje y devuelve la respuesta JSON de la API.

### get_updates

```python
def get_updates():
    """
    Obtiene los mensajes recientes que el bot ha recibido.

    Este método utiliza la API de Telegram para recuperar las actualizaciones
    (mensajes) que han sido enviados al bot.

    Returns
    -------
    dict
        Respuesta JSON de la API de Telegram con las actualizaciones recibidas.
    """
    url = BASE_URL + 'getUpdates'
    response = requests.get(url)
    return response.json()
```

...

### Descripción

La función get_updates obtiene los mensajes recientes que el bot ha recibido utilizando la API de Telegram. Devuelve la respuesta JSON con las actualizaciones recibidas.
