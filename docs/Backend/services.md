# Documentación de la Aplicación Services

## Introducción

La aplicación `services` gestiona los servicios que pueden ser ofrecidos por profesionales, como barberos. Permite a los administradores crear, editar, eliminar y listar servicios.

## Modelos

### Service

```python

class Service(models.Model):
    """
    Modelo que representa un servicio que puede ser ofrecido por un profesional.

    Atributos
    ----------
    name : CharField
        Nombre del servicio.
    description : TextField
        #### Descripción opcional del servicio.
    price : DecimalField
        Precio del servicio, con hasta 8 dígitos y 2 decimales.
    image : ImageField
        Imagen representativa del servicio. Puede estar vacía o usar una por defecto.
    duration : DurationField
        Duración del servicio como objeto timedelta.
    created_at : DateTimeField
        Fecha y hora en que el servicio fue creado.
    """
    name = models.CharField(max_length=100)
    description = models.TextField(blank=True, null=True)
    price = models.DecimalField(max_digits=8, decimal_places=2)
    image = models.ImageField(
        upload_to='services_images/',
        default='product_images/no_product.png',
        blank=True,
        null=True,
    )
    duration = models.DurationField()
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.name

    @staticmethod
    def convert_duration_string(duration_str):
        """
        Convierte una cadena en formato ISO 8601 a un objeto timedelta.
        """
        pattern = re.compile(r'P(?:\d+D)?T(?:(\d+)H)?(?:(\d+)M)?(?:(\d+)S)?')
        match = pattern.match(duration_str)
        if not match:
            raise ValueError('Formato de duración no válido')
        hours = int(match.group(1)) if match.group(1) else 0
        minutes = int(match.group(2)) if match.group(2) else 0
        seconds = int(match.group(3)) if match.group(3) else 0
        return timedelta(hours=hours, minutes=minutes, seconds=seconds)
```

#### Descripción

El modelo Service representa un servicio que puede ser ofrecido por un profesional, incluyendo atributos como nombre, descripción, precio, duración y una imagen.

## Serializadores

### ServiceSerializer

```python
from shared.serializers import BaseSerializer

class ServiceSerializer(BaseSerializer):
    """
    Serializador para el modelo Service.

    Métodos
    -------
    serialize_instance(instance) -> dict
        Serializa una instancia de Service en un diccionario con sus atributos principales.
    """
    def serialize_instance(self, instance) -> dict:
        return {
            'id': instance.id,
            'name': instance.name,
            'description': instance.description,
            'duration': instance.duration,
            'price': str(instance.price),
            'created_at': instance.created_at,
            'image': f'{self.build_url(instance.image.url)}',
        }
```

...

#### Descripción

El ServiceSerializer se utiliza para serializar instancias del modelo Service, proporcionando una representación en formato JSON de los atributos relevantes.

## Decoradores

### verify_service

```python

def verify_service(func):
    """
    Verifica que el servicio especificado exista.

    Este decorador intenta obtener un servicio de la base de datos utilizando
    el ID proporcionado en los argumentos de la función. Si el servicio existe,
    se asigna al objeto de solicitud. Si no se encuentra el servicio, se devuelve
    un error 404.
    """
    def wrapper(request, *args, **kwargs):
        try:
            service = Service.objects.get(id=kwargs['service_pk'])
            request.service = service
            return func(request, *args, **kwargs)
        except Service.DoesNotExist:
            return JsonResponse({'error': 'Servicio no encontrado'}, status=404)

    return wrapper
```

...

#### Descripción

Este decorador verifica la existencia de un servicio en la base de datos utilizando el service_pk proporcionado en la URL. Si no existe, devuelve un error 404.
Vistas

### service_list

```python
@csrf_exempt
@required_method('GET')
def service_list(request):
    """
    Devuelve una lista de todos los servicios en formato JSON.

    Este endpoint permite obtener todos los servicios registrados en la base de datos.
    """
    services = Service.objects.all()
    serializer = ServiceSerializer(services, request=request)
    return serializer.json_response()
```

...

#### Descripción

Este endpoint devuelve una lista de todos los servicios registrados en la base de datos en formato JSON.

### service_detail

```python
@csrf_exempt
@required_method('GET')
@verify_service
def service_detail(request, service_pk):
    """
    Devuelve los detalles de un servicio específico.

    Este endpoint permite obtener los detalles de un servicio utilizando su ID.
    """
    serializer = ServiceSerializer(request.service, request=request)
    return serializer.json_response()
```

...

#### Descripción

Este endpoint devuelve los detalles de un servicio específico utilizando su ID.

### add_service

```python
@csrf_exempt
@required_method('POST')
@load_json_body
@verify_token
@verify_admin
def add_service(request):
    """
    Agrega un nuevo servicio.

    Este endpoint permite a un administrador autenticado crear un nuevo servicio
    proporcionando el nombre, descripción, precio y duración del servicio.
    """
    try:
        name = request.json_body.get('name')
        description = request.json_body.get('description')
        price = request.json_body.get('price')
        duration = request.json_body.get('duration')
        image_base64 = request.json_body.get('image')

        if not all([name, description, price, duration]):
            return JsonResponse(
                {'error': 'Todos los campos son requeridos: name, description, price, duration'},
                status=400,
            )

        duration = Service.convert_duration_string(duration)

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

        service = Service.objects.create(
            name=name, description=description, price=price, duration=duration, image=image_file
        )

        return JsonResponse({'id': service.pk, 'msg': 'Servicio creado exitosamente'})

    except Exception as e:
        return JsonResponse({'error': f'Error al crear el servicio: {str(e)}'}, status=500)
```

...

#### Descripción

Este endpoint permite a un administrador autenticado crear un nuevo servicio proporcionando los datos necesarios.

### edit_service

```python
@csrf_exempt
@required_method('POST')
@load_json_body
@required_fields('name', 'description', 'price', 'duration', model=Service)
@verify_token
@verify_admin
@verify_service
def edit_service(request, service_pk: int):
    """
    Edita un servicio existente.

    Este endpoint permite a un administrador autenticado editar un servicio
    existente proporcionando los nuevos datos del servicio.
    """
    service = request.service
    service.name = request.json_body['name']
    service.description = request.json_body['description']
    service.price = request.json_body['price']
    duration = request.json_body['duration']
    service.duration = Service.convert_duration_string(duration)
    image_base64 = request.json_body.get('image')
    if image_base64:
        try:
            format_part, data_part = image_base64.split(',')
            file_format = format_part.split('/')[1].split(';')[0]

            image_data = base64.b64decode(data_part)

            filename = f'service_{uuid.uuid4().hex[:8]}.{file_format}'
            image_file = ContentFile(image_data, name=filename)
            service.image = image_file
        except Exception as e:
            return JsonResponse({'error': f'Error procesando la imagen: {str(e)}'}, status=400)

    service.save()
    return JsonResponse({'msg': 'El servicio ha sido editado'})
```

...

#### Descripción

Este endpoint permite a un administrador autenticado editar un servicio existente.

### delete_service

```python
@csrf_exempt
@required_method('POST')
@verify_token
@verify_admin
@verify_service
def delete_service(request, service_pk: int):
    """
    Elimina un servicio existente.

    Este endpoint permite a un administrador autenticado eliminar un servicio
    específico utilizando su ID.
    """
    service = request.service
    service.delete()
    return JsonResponse({'msg': 'El servicio ha sido eliminado'})
```

...

#### Descripción

Este endpoint permite a un administrador autenticado eliminar un servicio específico.

## URLs

```python
from django.urls import path

from . import views

name = 'services'
urlpatterns = [
    path('', views.service_list, name='service-list'),
    path('add/', views.add_service, name='add-service'),
    path('<int:service_pk>/', views.service_detail, name='service-detail'),
    path('<int:service_pk>/edit/', views.edit_service, name='edit-service'),
    path('<int:service_pk>/delete/', views.delete_service, name='delete-service'),
]
```

...
