# Documentación de la Aplicación Events

## Introducción

La aplicación `events` gestiona eventos programados, permitiendo a los administradores crear, editar, eliminar y listar eventos.

## Modelos

### Event

```python
from django.db import models

class Event(models.Model):
    """
    Modelo que representa un evento programado.

    Atributos
    ----------
    name : CharField
        Nombre del evento.
    description : TextField
        #### Descripción opcional del evento.
    date : DateField
        Fecha en la que se llevará a cabo el evento.
    time : TimeField
        Hora en la que comenzará el evento.
    image : ImageField
        Imagen representativa del evento. Puede estar vacía o usar una imagen por defecto.
    location : CharField
        Dirección o lugar donde se realiza el evento.
    created_at : DateTimeField
        Fecha y hora en la que se creó el evento (automáticamente asignada).
    """
    name = models.CharField(max_length=100)
    description = models.TextField(blank=True, null=True)
    date = models.DateField()
    time = models.TimeField()
    image = models.ImageField(
        upload_to='media/events_images/',
        default='events_images/no_event.png',
        blank=True,
        null=True,
    )
    location = models.CharField(max_length=255)
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        """
        Retorna una representación legible del evento.

        Returns
        -------
        str
            Cadena con el formato 'Nombre en Fecha en Ubicación'.
        """
        return f'{self.name} on {self.date} at {self.location}'
```

#### Descripción

El modelo Event representa un evento programado, incluyendo atributos como nombre, Descripción, fecha, hora, imagen y ubicación.

## Serializadores

### EventSerializer

```python

class EventSerializer(BaseSerializer):
    """
    Serializador para instancias del modelo Event.

    Este serializador transforma una instancia de Event en un diccionario
    con los campos relevantes para su representación en una API.
    """
    def serialize_instance(self, instance) -> dict:
        """
        Serializa una instancia del modelo Event a un diccionario.

        Parameters
        ----------
        instance : Event
            Instancia del modelo Event que se desea serializar.

        Returns
        -------
        dict
            Diccionario con los datos serializados del evento.
        """
        return {
            'id': instance.id,
            'name': instance.name,
            'description': instance.description,
            'date': instance.date,
            'time': instance.time,
            'location': instance.location,
            'image': f'{self.build_url(instance.image.url)}',
        }
```

...

#### Descripción

El EventSerializer se utiliza para serializar instancias del modelo Event, proporcionando una representación en formato JSON de los atributos relevantes.

## Decoradores

### verify_event

```python

def verify_event(func):
    """
    Decorador que intenta recuperar un evento usando 'event_pk' desde los parámetros de la URL.
    Si el evento existe, se adjunta al objeto request como 'request.event'.
    Si no existe, devuelve una respuesta JSON con error 404 (No encontrado).
    Parameters
    ----------
    func : callable
        Vista a decorar.
    Returns
    -------
    callable
        Vista decorada que incluye la verificación de existencia del evento.

    """

    def wrapper(request, *args, **kwargs):
        try:
            event = Event.objects.get(id=kwargs['event_pk'])
            request.event = event
        except Event.DoesNotExist:
            return JsonResponse({'error': 'Evento no encontrado'}, status=404)

        return func(request, *args, **kwargs)

    return wrapper
```

#### Descripción

El decorador verify_event se utiliza para verificar la existencia de un evento en la base de datos utilizando el event_pk proporcionado en los parámetros de la URL.

## Vistas

### event_list

```python
@csrf_exempt
@required_method('GET')
def event_list(request):
    """
    Devuelve una lista de todos los eventos en formato JSON.

    Este endpoint permite obtener todos los eventos registrados en la base de datos.

    Parameters
    ----------
    request : HttpRequest
        Objeto de solicitud HTTP.

    Returns
    -------
    JsonResponse
        Respuesta JSON con la lista de eventos serializados.
    """
    events = Event.objects.all()
    serializer = EventSerializer(events, request=request)
    return serializer.json_response()
```

...

#### Descripción

Este endpoint devuelve una lista de todos los eventos registrados en la base de datos en formato JSON.

### event_detail

```python
@required_method('GET')
@verify_event
def event_detail(request, event_pk):
    """
    Devuelve los detalles de un evento específico.

    Parameters
    ----------
    request : HttpRequest
        Objeto de solicitud HTTP.
    event_pk : int
        ID del evento a consultar.

    Returns
    -------
    JsonResponse
        Respuesta JSON con los detalles del evento.
    """
    event = request.event
    serializer = EventSerializer(event, request=request)
    return serializer.json_response()
```

...

#### Descripción

Este endpoint devuelve los detalles de un evento específico utilizando su ID.

### add_event

```python
@csrf_exempt
@required_method('POST')
@load_json_body
@required_fields('name', 'description', 'date', 'time', 'location', model=Event)
@verify_token
@verify_admin
def add_event(request):
    """
    Agrega un nuevo evento.

    Solo un administrador autenticado puede crear eventos nuevos.

    Parameters
    ----------
    request : HttpRequest
        Objeto de solicitud HTTP que contiene los datos del evento.

    Returns
    -------
    JsonResponse
        Respuesta JSON con el ID del evento creado.
    """
    name = request.json_body['name']
    description = request.json_body['description']
    date = request.json_body['date']
    time = request.json_body['time']
    location = request.json_body['location']
    image_base64 = request.json_body.get('image')

    image_file = None
    if image_base64:
        try:
            format_part, data_part = image_base64.split(',')
            file_format = format_part.split('/')[1].split(';')[0]

            image_data = base64.b64decode(data_part)

            filename = f'service_{uuid.uuid4().hex[:8]}.{file_format}'
            image_file = ContentFile(image_data, name=filename)

        except Exception as e:
            return JsonResponse({'error': f'Error procesando la imagen: {str(e)}'}, status=400)

    event = Event.objects.create(
        name=name,
        description=description,
        location=location,
        date=date,
        image=image_file,
        time=time,
    )

    return JsonResponse({'id': event.pk, 'msg': 'Servicio creado exitosamente'})
```

...

#### Descripción

Este endpoint permite a un administrador autenticado crear un nuevo evento.

### edit_event

```python
@csrf_exempt
@required_method('POST')
@load_json_body
@required_fields('name', 'description', 'date', 'time', 'location', model=Event)
@verify_token
@verify_admin
@verify_event
def edit_event(request, event_pk: int):
    """
    Edita un evento existente.

    Un administrador autenticado puede modificar los datos de un evento.

    Parameters
    ----------
    request : HttpRequest
        Objeto de solicitud HTTP con los datos del evento.
    event_pk : int
        ID del evento a editar.

    Returns
    -------
    JsonResponse
        Respuesta JSON con un mensaje de confirmación.
    """
    event = request.event
    event.name = request.json_body['name']
    event.description = request.json_body['description']
    event.date = request.json_body['date']
    event.location = request.json_body['location']
    event.time = request.json_body['time']
    image_base64 = request.json_body.get('image')
    if image_base64:
        try:
            format_part, data_part = image_base64.split(',')
            file_format = format_part.split('/')[1].split(';')[0]

            image_data = base64.b64decode(data_part)

            filename = f'service_{uuid.uuid4().hex[:8]}.{file_format}'
            image_file = ContentFile(image_data, name=filename)
            event.image = image_file
        except Exception as e:
            return JsonResponse({'error': f'Error procesando la imagen: {str(e)}'}, status=400)

    event.save()
    return JsonResponse({'msg': 'Event has been edited'})
```

...

#### Descripción

Este endpoint permite a un administrador autenticado editar un evento existente.

### delete_event

```python
@csrf_exempt
@required_method('POST')
@verify_token
@verify_admin
@verify_event
def delete_event(request, event_pk: int):
    """
    Elimina un evento existente.

    Solo un administrador autenticado puede eliminar eventos.

    Parameters
    ----------
    request : HttpRequest
        Objeto de solicitud HTTP.
    event_pk : int
        ID del evento a eliminar.

    Returns
    -------
    JsonResponse
        Respuesta JSON con un mensaje de éxito.
    """
    event = request.event
    event.delete()
    return JsonResponse({'msg': 'Event has been deleted'})
```

...

#### Descripción

Este endpoint permite a un administrador autenticado eliminar un evento existente.

## URLs

```python

name = 'events'
urlpatterns = [
    path('', views.event_list, name='event_list'),
    path('<int:event_pk>/', views.event_detail, name='event-detail'),
    path('add/', views.add_event, name='add-event'),
    path('<int:event_pk>/edit/', views.edit_event, name='edit-event'),
    path('<int:event_pk>/delete/', views.delete_event, name='delete-event'),
]
```

...
