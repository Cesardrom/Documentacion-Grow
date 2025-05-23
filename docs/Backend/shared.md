# Documentación de la Aplicación Shared

## Introducción

La aplicación `shared` contiene componentes reutilizables que son utilizados por otras aplicaciones en el proyecto, incluyendo serializadores y decoradores.

## Serializadores

### BaseSerializer

```python

class BaseSerializer(ABC):
    def __init__(
        self,
        to_serialize: object | Iterable[object],
        *,
        fields: Iterable[str] = [],
        request: HttpRequest = None,
    ):
        self.to_serialize = to_serialize
        self.fields = fields
        self.request = request

    def build_url(self, path: str) -> str:
        return self.request.build_absolute_uri(path) if self.request else path

    def serialize_instance(self, instance: object) -> dict:
        raise NotImplementedError

    def __serialize_instance(self, instance: object) -> dict:
        serialized = self.serialize_instance(instance)
        return {f: v for f, v in serialized.items() if not self.fields or f in self.fields}

    def serialize(self) -> dict | list[dict]:
        if not isinstance(self.to_serialize, Iterable):
            return self.__serialize_instance(self.to_serialize)
        return [self.__serialize_instance(instance) for instance in self.to_serialize]

    def to_json(self) -> str:
        return json.dumps(self.serialize())

    def json_response(self) -> str:
        return JsonResponse(self.serialize(), safe=False)
```

#### Descripción

El BaseSerializer es una clase base para serializadores que permite convertir instancias de modelos en diccionarios que pueden ser fácilmente convertidos a JSON. Proporciona métodos para construir URLs, serializar instancias y generar respuestas JSON.

## Decoradores

### verify_token

```python

def verify_token(func):
    """
    Verifica el token de autenticación del usuario.

    Este decorador comprueba si el token de autenticación proporcionado en
    la cabecera 'Authorization' es válido. Si el token es válido, se
    asigna el usuario correspondiente a la solicitud. Si el token no es
    válido o no está registrado, se devuelve un error.
    """
    def wrapper(request, *args, **kwargs):
        UUID_PATTERN = re.compile(
            r'Bearer (?P<token>[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12})'
        )
        bearer_auth = request.headers.get('Authorization')
        if m := re.fullmatch(UUID_PATTERN, bearer_auth):
            token_reg = m['token']
            try:
                token = Token.objects.get(key=token_reg)
                request.user = token.user
            except Token.DoesNotExist:
                return JsonResponse({'error': 'Token de autenticación no registrado'}, status=401)
        else:
            return JsonResponse({'error': 'Token de autenticación inválido'}, status=400)
        return func(request, *args, **kwargs)

    return wrapper
```

...

#### Descripción

El decorador verify_token verifica el token de autenticación del usuario. Si el token es válido, asigna el usuario correspondiente a la solicitud; de lo contrario, devuelve un error.

### required_method

```python
def required_method(method_type):
    """
    Verifica que el método de la solicitud sea el esperado.

    Este decorador comprueba si el método HTTP de la solicitud coincide
    con el tipo de método requerido. Si no coincide, se devuelve un error
    de método no permitido.
    """
    def decorator(func):
        def wrapper(request, *args, **kwargs):
            if request.method != method_type:
                return JsonResponse({'error': 'Método no permitido'}, status=405)
            return func(request, *args, **kwargs)

        return wrapper

    return decorator
```

...

#### Descripción

El decorador required_method verifica que el método HTTP de la solicitud coincida con el tipo de método requerido. Si no coincide, devuelve un error 405.

### load_json_body

```python
def required_fields(*fields, model):
    """
    Verifica que los campos requeridos estén presentes en el cuerpo de la solicitud.

    Este decorador comprueba si los campos requeridos están presentes en
    el cuerpo JSON de la solicitud. Si falta algún campo, se devuelve un
    error.
    """
    def decorator(func):
        def wrapper(request, *args, **kwargs):
            json_body = json.loads(request.body)
            for field in fields:
                if field not in json_body:
                    return JsonResponse({'error': 'Faltan campos requeridos'}, status=400)
            return func(request, *args, **kwargs)

        return wrapper

    return decorator
```

...

#### Descripción

El decorador load_json_body carga el cuerpo de la solicitud como un objeto JSON. Si el cuerpo está vacío o no es un JSON válido, devuelve un error.

### required_fields

```python
def required_fields(*fields, model):
    """
    Verifica que los campos requeridos estén presentes en el cuerpo de la solicitud.

    Este decorador comprueba si los campos requeridos están presentes en
    el cuerpo JSON de la solicitud. Si falta algún campo, se devuelve un
    error.
    """
    def decorator(func):
        def wrapper(request, *args, **kwargs):
            json_body = json.loads(request.body)
            for field in fields:
                if field not in json_body:
                    return JsonResponse({'error': 'Faltan campos requeridos'}, status=400)
            return func(request, *args, **kwargs)

        return wrapper

    return decorator
```

...

#### Descripción

El decorador required_fields verifica que los campos requeridos estén presentes en el cuerpo de la solicitud. Si falta algún campo, devuelve un error.

### verify_admin

```python
def verify_admin(func):
    """
    Verifica que el usuario autenticado sea un administrador.

    Este decorador comprueba si el usuario autenticado tiene el rol de
    administrador. Si no es un administrador, se devuelve un error de
    acceso denegado.
    """
    def wrapper(request, *args, **kwargs):
        if request.user.is_authenticated:
            if request.user.profile.role == 'A':
                return func(request, *args, **kwargs)
        return JsonResponse({'error': 'El usuario debe ser un administrador'}, status=403)

    return wrapper
```

...

#### Descripción

El decorador verify_admin verifica que el usuario autenticado sea un administrador. Si no lo es, devuelve un error 403.
