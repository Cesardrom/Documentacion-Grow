# Documentación de la Aplicación Bookings

## Introducción

La aplicación `bookings` gestiona las reservas de servicios de los usuarios. Permite a los usuarios crear, editar, cancelar y listar sus reservas, así como obtener información sobre las fechas y horarios disponibles.

## Modelos

### TimeSlot

```python
class TimeSlot(models.Model):
    """
    Modelo que representa un bloque horario para una reserva.

    Attributes
    ----------
    start_time : TimeField
        Hora de inicio del bloque.
    end_time : TimeField
        Hora de fin del bloque.
    """
    start_time = models.TimeField()
    end_time = models.TimeField()

    def __str__(self):
        """
        Devuelve una representación legible del bloque horario.

        Returns
        -------
        str
            Cadena con el formato 'HH:MM:SS-HH:MM:SS'.
        """
        return f'{self.start_time}-{self.end_time}'
```

...

#### Descripción

El modelo TimeSlot representa un bloque horario para una reserva, con atributos para la hora de inicio y fin.

## Booking

```python
class Booking(models.Model):
    """
    Modelo que representa una reserva de servicio entre un cliente y un barbero.

    Attributes
    ----------
    user : ForeignKey
        Usuario que realiza la reserva.
    barber : ForeignKey
        Usuario con rol de barbero que atiende la reserva.
    service : ForeignKey
        Servicio seleccionado para la reserva.
    date : DateField
        Fecha de la reserva.
    time_slot : ForeignKey
        Bloque horario en el que se agenda la reserva.
    status : IntegerField
        Estado de la reserva (confirmada o cancelada).
    created_at : DateTimeField
        Fecha de creación de la reserva.
    """
    class Meta:
        unique_together = ['barber', 'date', 'time_slot']

    class Status(models.IntegerChoices):
        """
        Enumeración de los posibles estados de una reserva.
        """
        CONFIRMED = 2, 'Confirmed'
        CANCELLED = -1, 'Cancelled'

    user = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name='bookings'
    )
    barber = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name='barber_bookings',
        limit_choices_to={'profile__role': Profile.Role.WORKER},
        verbose_name='Barbero',
    )
    service = models.ForeignKey(Service, on_delete=models.CASCADE, related_name='bookings')
    date = models.DateField()
    time_slot = models.ForeignKey(TimeSlot, on_delete=models.CASCADE, related_name='bookings')
    status = models.IntegerField(choices=Status.choices, default=Status.CONFIRMED)
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        """
        Devuelve una representación legible de la reserva.

        Returns
        -------
        str
            Cadena con el formato 'usuario servicio fecha horario'.
        """
        return f'{self.user} {self.service} {self.date} {self.time_slot}'
```

...

#### Descripción

El modelo Booking representa una reserva de servicio entre un cliente y un barbero, incluyendo información sobre el usuario, el barbero, el servicio, la fecha, el horario y el estado de la reserva.

## Serializadores

### BookingSerializer

```python
class BookingSerializer(BaseSerializer):
    """
    Serializador para una reserva.
    """
    def serialize_instance(self, instance) -> dict:
        """
        Serializa una instancia de reserva.

        Parameters
        ----------
        instance : Booking
            La instancia de reserva a serializar.

        Returns
        -------
        dict
            Un diccionario que representa la instancia de reserva.
        """
        return {
            'id': instance.id,
            'user': instance.user.id,
            'service': ServiceSerializer(instance.service).serialize_instance(instance.service),
            'date': instance.date,
            'time_slot': TimeSlotSerializer(instance.time_slot).serialize_instance(
                instance.time_slot
            ),
            'barber': instance.barber.get_full_name(),
            'barber_id': instance.barber.id,
            'status': instance.get_status_display(),
            'created_at': instance.created_at,
        }
```

...

#### Descripción

El BookingSerializer se utiliza para serializar instancias de reservas, proporcionando una representación en formato JSON de los atributos relevantes.

### TimeSlotSerializer

```python
class TimeSlotSerializer(BaseSerializer):
    """
    Serializador para un bloque horario.
    """
    def serialize_instance(self, instance) -> dict:
        """
        Serializa una instancia de bloque horario.

        Parameters
        ----------
        instance : TimeSlot
            La instancia de bloque horario a serializar.

        Returns
        -------
        dict
            Un diccionario que representa la instancia de bloque horario.
        """
        return {
            'id': instance.id,
            'start_time': instance.start_time,
            'end_time': instance.end_time,
        }
```

...

#### Descripción

El TimeSlotSerializer se utiliza para serializar instancias de bloques horarios, proporcionando una representación en formato JSON de los atributos relevantes.

### BookingEarningsSerializer

```python
class BookingEarningsSerializer(BaseSerializer):
    """
    Serializador para el resumen de ganancias de reservas.
    """
    def serialize_instance(self, instance=None) -> dict:
        """
        Serializa el resumen de ganancias de las reservas.

        Parameters
        ----------
        instance : Booking, opcional
            La instancia de reserva (no se utiliza en este método).

        Returns
        -------
        dict
            Un diccionario que contiene el resumen de ganancias.
        """
        summary = Booking.earnings_summary()
        return {
            'daily_earnings': summary['daily'],
            'weekly_earnings': summary['weekly'],
            'monthly_earnings': summary['monthly'],
        }
```

...

#### Descripción

El BookingEarningsSerializer se utiliza para serializar el resumen de ganancias de las reservas, proporcionando una representación en formato JSON de los ingresos diarios, semanales y mensuales.

## Vistas

### user_booking_list

```python

@csrf_exempt
@required_method('GET')
@verify_token
def user_booking_list(request):
    """
    Devuelve una lista de todas las reservas de usuario en formato JSON.
    Este endpoint requiere que el usuario esté autenticado.
    Realiza una consulta a la base de datos para obtener todas las reservas de un usuario
    y las serializa antes de devolverlas en una respuesta JSON.

    Parameters
    ----------
    request : HttpRequest
        Objeto de solicitud HTTP.
    Returns
    -------
    JsonResponse
        Respuesta JSON con la lista de reservas.
    """
    bookings = Booking.objects.filter(user=request.user)
    bookings_serializer = [BookingSerializer(booking).serialize() for booking in bookings]
    return JsonResponse(bookings_serializer, safe=False, status=200)
```

#### Descripción

Este endpoint devuelve una lista de todas las reservas del usuario autenticado en formato JSON.

### create_booking

```python
@csrf_exempt
@required_method('POST')
@load_json_body
@required_fields('service', 'time_slot', 'date', 'barber', model=Booking)
@verify_token
@validate_barber_and_timeslot_existence
@validate_barber_availability
def create_booking(request):
    """
    Crea una nueva reserva.

    Este endpoint permite a un usuario autenticado crear una nueva reserva
    proporcionando el servicio, el horario, la fecha y el barbero.
    Si el servicio no existe, devuelve un error.

    Parameters
    ----------
    request : HttpRequest
        Objeto de solicitud HTTP que contiene los datos de la reserva.

    Returns
    -------
    JsonResponse
        Respuesta JSON con el ID de la nueva reserva o un error si el servicio no se encuentra.
    """
    service_pk = request.json_body['service']
    barber_id = request.json_body['barber']
    date = request.json_body['date']

    try:
        service = Service.objects.get(pk=service_pk)
    except Service.DoesNotExist:
        return JsonResponse({'error': 'Servicio no encontrado.'}, status=400)

    try:
        barber_profile = Profile.objects.get(user_id=barber_id, role=Profile.Role.WORKER)
    except Profile.DoesNotExist:
        return JsonResponse({'error': 'Barbero no encontrado o no válido.'}, status=404)

    booking = Booking.objects.create(
        user=request.user,
        barber=barber_profile.user,
        service=service,
        date=date,
        time_slot=request.time_slot,
    )

    return JsonResponse({'id': booking.pk})
```

...

#### Descripción

Este endpoint permite a un usuario autenticado crear una nueva reserva proporcionando el servicio, el horario, la fecha y el barbero.

### edit_booking

```python
@login_required
@csrf_exempt
@required_method('POST')
@load_json_body
@required_fields('service', 'time_slot', 'date', 'barber', model=Booking)
@verify_token
@verify_booking
@validate_barber_and_timeslot_existence
@validate_barber_availability
def edit_booking(request, booking_pk):
    """
    Edita una reserva existente.

    Este endpoint permite a un usuario autenticado editar una reserva
    existente proporcionando el nuevo servicio, horario, fecha y barbero.

    Parameters
    ----------
    request : HttpRequest
        Objeto de solicitud HTTP que contiene los datos de la reserva.
    booking_pk : int
        ID de la reserva a editar.

    Returns
    -------
    JsonResponse
        Respuesta JSON con un mensaje de éxito.
    """
    service_pk = request.json_body['service']
    date = request.json_body['date']

    service = Service.objects.get(pk=service_pk)
    booking = request.booking

    booking.service = service
    booking.barber = request.barber_profile.user
    booking.date = date
    booking.time_slot = request.time_slot
    booking.save()

    return JsonResponse({'msg': 'La reserva ha sido editada'})
```

...

#### Descripción

Este endpoint permite a un usuario autenticado editar una reserva existente.

### booking_detail

```python
@login_required
@csrf_exempt
@required_method('GET')
@verify_booking
def booking_detail(request, booking_pk):
    """
    Devuelve los detalles de una reserva específica.

    Este endpoint permite a un usuario autenticado obtener los detalles
    de una reserva específica utilizando su ID.

    Parameters
    ----------
    request : HttpRequest
        Objeto de solicitud HTTP.
    booking_pk : int
        ID de la reserva.

    Returns
    -------
    JsonResponse
        Respuesta JSON con los detalles de la reserva.
    """
    booking = request.booking
    serializer = BookingSerializer(booking, request=request)
    return serializer.json_response()
```

...

#### Descripción

Este endpoint devuelve los detalles de una reserva específica utilizando su ID.

### cancel_booking

```python
@csrf_exempt
@required_method('POST')
@verify_token
@verify_booking
def cancel_booking(request, booking_pk):
    """
    Cancela una reserva existente.

    Este endpoint permite a un usuario autenticado cancelar una reserva
    específica utilizando su ID.

    Parameters
    ----------
    request : HttpRequest
        Objeto de solicitud HTTP.
    booking_pk : int
        ID de la reserva a cancelar.

    Returns
    -------
    JsonResponse
        Respuesta JSON con un mensaje de éxito.
    """
    booking = request.booking
    booking.delete()
    return JsonResponse({'msg': 'La reserva ha sido cancelada'})
```

...

#### Descripción

Este endpoint permite a un usuario autenticado cancelar una reserva específica.

### get_available_dates

```python
@csrf_exempt
@required_method('GET')
@verify_token
def get_available_dates(request):
    """
    Devuelve las fechas disponibles para reservas de un barbero específico.

    Parámetros GET:
    - barber_id: ID del barbero para filtrar (requerido)

    Parameters
    ----------
    request : HttpRequest
        Objeto de solicitud HTTP.

    Returns
    -------
    JsonResponse
        Respuesta JSON con las fechas y horarios disponibles para el barbero especificado.
    """
    barber_id = request.GET.get('barber_id')

    if not barber_id:
        return JsonResponse({'error': 'El parámetro barber_id es requerido'}, status=400)

    try:
        barber = User.objects.get(id=barber_id, profile__role=Profile.Role.WORKER)
    except User.DoesNotExist:
        return JsonResponse({'error': 'Barbero no encontrado'}, status=404)

    now = timezone.now()
    today = now.date()
    start_date = today
    end_date = today + timedelta(days=13)  # 14 días desde hoy, incluyendo hoy.

    available_slots = {}
    current_date = start_date

    while current_date <= end_date:
        if is_working_day(current_date):
            current_time = now.time() if current_date == today else None
            available_slots[current_date.isoformat()] = get_available_time_slots(
                barber, current_date, current_time
            )
        current_date += timedelta(days=1)

    return JsonResponse(
        {
            'barber_id': barber.id,
            'barber_name': barber.get_full_name() or barber.username,
            'available_dates': available_slots,
        }
    )
```

...

#### Descripción

Este endpoint devuelve las fechas y horarios disponibles para reservas de un barbero específico.

### get_earnings

```python
@csrf_exempt
@required_method('GET')
@verify_token
@verify_admin
def get_earnings(request):
    """
    Obtiene las ganancias diarias del mes actual basadas en reservas confirmadas.

    Esta vista recorre cada día del mes actual y calcula el total de ingresos generados
    por las reservas confirmadas (status = CONFIRMED). Las ganancias se determinan
    a partir del precio del servicio asociado a cada reserva.

    Parameters
    ----------
    request : HttpRequest
        La solicitud HTTP entrante. Debe ser de tipo GET y debe incluir un token de autenticación válido.
        Solo accesible por usuarios con permisos de administrador.

    Returns
    -------
    JsonResponse
        Un objeto JSON con:
            - 'labels': Lista de fechas (str) en formato 'YYYY-MM-DD'.
            - 'values': Lista de floats representando las ganancias totales por día.
    """
    now = timezone.now()
    first_day_of_month = now.replace(day=1)
    last_day_of_month = (first_day_of_month + timezone.timedelta(days=31)).replace(
        day=1
    ) - timezone.timedelta(days=1)

    total_earnings = []
    labels = []

    for day in range(1, last_day_of_month.day + 1):
        date = first_day_of_month.replace(day=day)
        bookings = Booking.objects.filter(created_at__date=date, status=Booking.Status.CONFIRMED)
        earnings = sum(booking.service.price for booking in bookings)
        total_earnings.append(earnings)
        labels.append(date.strftime('%Y-%m-%d'))

    return JsonResponse(
        {
            'labels': labels,
            'values': total_earnings,
        }
    )
```

...

#### Descripción

Este endpoint obtiene las ganancias diarias del mes actual basadas en reservas confirmadas.

## URLs

```python
from django.urls import path
from . import views

name = 'bookings'
urlpatterns = [
    path('', views.user_booking_list, name='user_booking_list'),
    path('get-earnings/', views.get_earnings, name='get-earnings'),
    path('<int:booking_pk>/', views.booking_detail, name='booking_detail'),
    path('add/', views.create_booking, name='add-booking'),
    path('dates/', views.get_available_dates, name='add-available-dates'),
    path('<int:booking_pk>/edit/', views.edit_booking, name='edit-booking'),
    path('<int:booking_pk>/cancel/', views.cancel_booking, name='cancel-booking'),
]
```

...

## Utilitarios

### get_available_time_slots

```python
def get_available_time_slots(barber, date, current_time=None):
    """
    Obtiene los horarios disponibles para un barbero en una fecha específica.

    Parameters
    ----------
    barber : Barber
        Instancia del barbero.
    date : datetime.date
        Fecha para la que se buscan los horarios.
    current_time : datetime.time, opcional
        Hora actual, utilizada para excluir horarios pasados.

    Returns
    -------
    list of dict
        Lista de horarios disponibles, donde cada horario es un diccionario
        que contiene las siguientes claves:
            - id : int
            - start_time : str
            - end_time : str
    """
    time_slots = TimeSlot.objects.all()
    booked_slots = Booking.objects.filter(barber=barber, date=date).values_list(
        'time_slot', flat=True
    )
    available_time_slots = time_slots.exclude(id__in=booked_slots)

    if current_time:
        available_time_slots = available_time_slots.filter(start_time__gt=current_time)

    return [
        {
            'id': slot.id,
            'start_time': slot.start_time.strftime('%H:%M'),
            'end_time': slot.end_time.strftime('%H:%M'),
        }
        for slot in available_time_slots
    ]
```

#### Descripción

Esta función obtiene los horarios disponibles para un barbero en una fecha específica, excluyendo los horarios ya reservados y, opcionalmente, los horarios pasados.

### is_working_day

```python
def is_working_day(date):
    """
    Determina si una fecha es día laboral (excluye domingos).

    Parameters
    ----------
    date : datetime.date
        Fecha a evaluar.

    Returns
    -------
    bool
        True si es día laboral, False en caso contrario.
    """
    return date.weekday() != 6  # Excluye domingos.
```

...

#### Descripción

Esta función determina si una fecha es un día laboral, excluyendo los domingos.

## Decoradores

### verify_booking

```python

def verify_booking(func):
    """
    Decorador que intenta recuperar una reserva (booking) a partir de 'booking_pk' en los parámetros de la URL.

    Si la reserva existe, se adjunta al objeto request como 'request.booking'.
    Si no existe, devuelve una respuesta JSON con error 404 (Not Found).

    Parameters
    ----------
    func : callable
        Vista a decorar.

    Returns
    -------
    callable
        Vista decorada con la verificación de existencia de la reserva.
    """
    def wrapper(request, *args, **kwargs):
        try:
            booking = Booking.objects.get(id=kwargs['booking_pk'])
            request.booking = booking
        except Booking.DoesNotExist:
            return JsonResponse({'error': 'Booking not found'}, status=404)

        return func(request, *args, **kwargs)

    return wrapper
```

...

#### Descripción

Este decorador verifica la existencia de una reserva y la adjunta al objeto request. Si la reserva no existe, devuelve un error 404.

### validate_barber_and_timeslot_existence

```python
def validate_barber_and_timeslot_existence(view_func):
    """
    Decorador que valida la existencia del barbero y del intervalo de tiempo (time slot)
    en los datos enviados con la solicitud.

    Agrega el perfil del barbero y el time slot al objeto request si existen.
    Si alguno no existe o no es válido, devuelve una respuesta JSON con el error correspondiente.

    Parameters
    ----------
    view_func : callable
        Vista a decorar.

    Returns
    -------
    callable
        Vista decorada con validación de barbero e intervalo de tiempo.
    """
    def wrapper(request, *args, **kwargs):
        data = request.json_body

        try:
            request.barber_profile = Profile.objects.get(user_id=data['barber'])
            if request.barber_profile.role != Profile.Role.WORKER:
                return JsonResponse({'error': 'The user is not a Barber'}, status=400)
        except Profile.DoesNotExist:
            return JsonResponse({'error': 'Barber not found'}, status=404)

        try:
            request.time_slot = TimeSlot.objects.get(pk=data['time_slot'])
        except TimeSlot.DoesNotExist:
            return JsonResponse({'error': 'Invalid time slot'}, status=404)

        return view_func(request, *args, **kwargs)

    return wrapper
```

...

#### Descripción

Este decorador valida la existencia del barbero y del intervalo de tiempo en los datos de la solicitud. Si alguno no existe, devuelve un error correspondiente.

### validate_barber_availability

```python
def validate_barber_availability(view_func):
    """
    Decorador que valida si el barbero está disponible en la fecha y hora solicitadas.

    Utiliza los datos 'date', 'barber_profile' y 'time_slot' previamente cargados en el objeto request.
    Si ya existe una reserva para ese barbero en la fecha y horario indicados,
    devuelve una respuesta JSON con error 400 (Bad Request).

    Parameters
    ----------
    view_func : callable
        Vista a decorar.

    Returns
    -------
    callable
        Vista decorada que valida la disponibilidad del barbero.
    """
    def wrapper(request, *args, **kwargs):
        date = request.json_body['date']
        barber_user = request.barber_profile.user
        time_slot = request.time_slot

        if Booking.objects.filter(barber=barber_user, date=date, time_slot=time_slot).exists():
            return JsonResponse(
                {'error': 'El barbero no está disponible en ese horario.'}, status=400
            )

        return view_func(request, *args, **kwargs)

    return wrapper
```

...

#### Descripción

Este decorador valida si el barbero está disponible en la fecha y hora solicitadas. Si ya existe una reserva, devuelve un error 400.
