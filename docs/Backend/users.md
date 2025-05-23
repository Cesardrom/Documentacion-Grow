# Documentación de la Aplicación Users

## Introducción

La aplicación `users` gestiona la información de los usuarios del sistema, incluyendo sus perfiles y tokens de autenticación. Permite la creación de perfiles extendidos y la gestión de roles.

## Modelos

### Profile

```python

class Profile(models.Model):
    """
    Modelo que representa el perfil extendido de un usuario, incluyendo su rol en el sistema.

    Atributos
    ----------
    user : OneToOneField
        Relación uno a uno con el usuario del sistema.
    role : CharField
        Rol del usuario, que puede ser ADMIN, WORKER o CLIENT.
    created_at : DateTimeField
        Fecha y hora de creación del perfil.
    updated_at : DateTimeField
        Fecha y hora de la última actualización del perfil.
    """
    class Role(models.TextChoices):
        ADMIN = 'A', 'ADMIN'
        WORKER = 'W', 'WORKER'
        CLIENT = 'C', 'CLIENT'

    user = models.OneToOneField(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name='profile'
    )
    role = models.CharField(choices=Role.choices, max_length=1, default=Role.CLIENT)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    def __str__(self):
        return f'{self.user}'
```

#### Descripción

El modelo Profile representa el perfil extendido de un usuario, incluyendo su rol en el sistema (ADMIN, WORKER o CLIENT) y las fechas de creación y actualización.

### Token

```python
class Token(models.Model):
    """
    Modelo que representa un token de autenticación único asociado a un usuario.

    Atributos
    ----------
    user : OneToOneField
        Relación uno a uno con el usuario del sistema.
    key : UUIDField
        Token único generado automáticamente para el usuario.
    created_at : DateTimeField
        Fecha y hora de creación del token.
    """
    user = models.OneToOneField(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name='token'
    )
    key = models.UUIDField(default=uuid.uuid4, unique=True)
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return f'{self.user}'
```

...

#### Descripción

El modelo Token representa un token de autenticación único asociado a un usuario, que se genera automáticamente al crear un nuevo usuario.

## Serializadores

### TokenSerializer

```python
from shared.serializers import BaseSerializer

class TokenSerializer(BaseSerializer):
    """
    Serializador para el modelo Token.

    Métodos
    -------
    serialize_instance(instance) -> dict
        Serializa una instancia de Token en un diccionario con su clave única.
    """
    def serialize_instance(self, instance) -> dict:
        return {'key': instance.key}
```

...

#### Descripción

El TokenSerializer se utiliza para serializar instancias del modelo Token, proporcionando una representación en formato JSON de la clave del token.

### ProfileSerializer

```python
class ProfileSerializer(BaseSerializer):
    """
    Serializador para el modelo Profile.

    Métodos
    -------
    serialize_instance(instance) -> dict
        Serializa una instancia de Profile en un diccionario con sus atributos principales.
    """
    def serialize_instance(self, instance) -> dict:
        return {
            'id': instance.user.id,
            'user': instance.user.username,
            'role': instance.role,
            'token': TokenSerializer(instance.user.token).serialize_instance(instance.user.token),
        }
```

...

#### Descripción

El ProfileSerializer se utiliza para serializar instancias del modelo Profile, proporcionando una representación en formato JSON de los atributos del perfil, incluyendo el usuario y su token.
Vistas

### get_user_profile

```python

@login_required
@csrf_exempt
@required_method('GET')
def get_user_profile(request):
    """
    Devuelve el perfil del usuario autenticado.

    Este endpoint permite a un usuario autenticado obtener su perfil.
    Si el perfil no se encuentra, se devuelve un error 404.
    """
    try:
        profile = Profile.objects.get(user=request.user)
    except Profile.DoesNotExist:
        return JsonResponse({'error': 'Perfil no encontrado'}, status=404)

    serializer = ProfileSerializer(profile, request=request)
    return serializer.json_response()
```

...

#### Descripción

Este endpoint permite a un usuario autenticado obtener su perfil. Si el perfil no se encuentra, devuelve un error 404.

### users_per_month

```python
@csrf_exempt
@required_method('GET')
@verify_token
@verify_admin
def users_per_mounth(request):
    """
    Obtiene la cantidad de usuarios registrados por día durante el mes actual.

    Esta vista recorre cada día del mes actual y cuenta cuántos perfiles de usuario
    fueron creados en cada fecha.
    """
    now = timezone.now()
    first_day_of_month = now.replace(day=1)
    last_day_of_month = (first_day_of_month + timezone.timedelta(days=31)).replace(
        day=1
    ) - timezone.timedelta(days=1)

    user_counts = []
    labels = []

    for day in range(1, last_day_of_month.day + 1):
        date = first_day_of_month.replace(day=day)
        count = Profile.objects.filter(created_at__date=date).count()
        user_counts.append(count)
        labels.append(date.strftime('%Y-%m-%d'))

    return JsonResponse(
        {
            'labels': labels,
            'values': user_counts,
        }
    )
```

...

#### Descripción

Este endpoint permite a un administrador obtener la cantidad de usuarios registrados por día durante el mes actual.

### get_barbers

```python
@csrf_exempt
@required_method('GET')
@verify_token
def get_barbers(request):
    """
    Devuelve una lista de barberos.

    Este endpoint permite a un usuario autenticado obtener una lista de
    todos los barberos registrados en el sistema.
    """
    barbers = Profile.objects.filter(role=Profile.Role.WORKER)
    serializer = ProfileSerializer(barbers, request=request)
    return serializer.json_response()
```

...

#### Descripción

Este endpoint permite a un usuario autenticado obtener una lista de todos los barberos registrados en el sistema.

## Señales

### create_user_related_models

```python
from django.conf import settings
from django.db.models.signals import post_save
from django.dispatch import receiver
from .models import Profile, Token

@receiver(post_save, sender=settings.AUTH_USER_MODEL)
def create_user_related_models(sender, instance, created, **kwargs):
    """
    Crea modelos relacionados al usuario después de que se guarda una instancia de usuario.

    Este receptor se activa después de que se guarda un nuevo usuario. Si el usuario
    es creado, se generan automáticamente un perfil y un token de autenticación para él.
    """
    if created:
        Profile.objects.create(user=instance)
        Token.objects.create(user=instance)
```

...

#### Descripción

Esta señal se activa después de que se guarda un nuevo usuario. Si el usuario es creado, se generan automáticamente un perfil y un token de autenticación para él.
