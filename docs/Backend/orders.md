# Documentación de la Aplicación Orders

## Introducción

La aplicación `orders` gestiona las órdenes de compra realizadas por los usuarios. Permite a los usuarios crear, ver, pagar y cancelar órdenes, así como a los administradores obtener un resumen de las ganancias.

## Modelos

###

```python

class Order(models.Model):
    """
    Modelo que representa una orden de compra realizada por un usuario.

    Attributes
    ----------
    status : CharField
        Estado de la orden: Pendiente, Completada o Cancelada.
    products : ManyToManyField
        Lista de productos asociados a la orden.
    price : DecimalField
        Precio total de la orden (opcional).
    created_at : DateTimeField
        Fecha y hora de creación de la orden.
    updated_at : DateTimeField
        Fecha y hora de la última actualización.
    user : ForeignKey
        Usuario que realizó la orden.
    """
    class Status(models.TextChoices):
        PENDING = 'P', 'Pending'
        COMPLETED = 'C', 'Completed'
        CANCELLED = 'X', 'Cancelled'

    status = models.CharField(max_length=1, choices=Status.choices, default=Status.PENDING)
    products = models.ManyToManyField('products.Product', blank=True)
    price = models.DecimalField(max_digits=10, decimal_places=2, blank=True, null=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    user = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name='orders'
    )

    def __str__(self):
        return f'{self.user} Status:{self.status}'

    def add(self, product):
        self.products.add(product)
        self.save()

    @classmethod
    def earnings_summary(cls):
        today = now().date()
        start_week = today - timedelta(days=today.weekday())
        start_month = today.replace(day=1)

        daily = (
            cls.objects.filter(created_at=today, status=cls.Status.COMPLETED).aggregate(
                total=Sum('price')
            )['total'] or 0
        )

        weekly = (
            cls.objects.filter(
                created_at__gte=start_week, created_at__lte=today, status=cls.Status.COMPLETED
            ).aggregate(total=Sum('price'))['total'] or 0
        )

        monthly = (
            cls.objects.filter(
                created_at__gte=start_month, created_at__lte=today, status=cls.Status.COMPLETED
            ).aggregate(total=Sum('price'))['total'] or 0
        )

        return {'daily': daily, 'weekly': weekly, 'monthly': monthly}
```

#### Descripción

El modelo Order representa una orden de compra, incluyendo atributos como estado, productos, precio, fechas de creación y actualización, y el usuario que realizó la orden.

## Serializadores

### OrderSerializer

```python

class OrderSerializer(BaseSerializer):
    """
    Serializador para el modelo Order.

    Este serializador convierte instancias de la clase Order en diccionarios
    que pueden ser fácilmente convertidos a JSON.
    """
    def serialize_instance(self, instance) -> dict:
        return {
            'id': instance.id,
            'products': ProductSerializer(
                instance.products.all(), request=self.request
            ).serialize(),
            'price': instance.price,
            'created_at': instance.created_at,
            'status': instance.get_status_display(),
        }
```

...

#### Descripción

El OrderSerializer se utiliza para serializar instancias del modelo Order, proporcionando una representación en formato JSON de los atributos relevantes.
Decoradores

### verify_user

```python
def verify_user(func):
    """
    Verifica que el usuario autenticado sea el propietario de la orden.

    Si el usuario que realiza la solicitud no es el propietario de la orden
    especificada en `order_pk`, se devuelve una respuesta con error 403 (Forbidden).
    """
    def wrapper(request, *args, **kwargs):
        order = Order.objects.get(pk=kwargs['order_pk'])
        if order.user != request.user:
            return JsonResponse({'error': 'User  is not the owner of requested order'}, status=403)
        return func(request, *args, **kwargs)

    return wrapper
```

...

#### Descripción

Este decorador verifica que el usuario autenticado sea el propietario de la orden especificada. Si no lo es, devuelve un error 403.

### verify_order

```python
def verify_order(func):
    """
    Verifica si la orden especificada por `order_pk` existe.

    Si existe, se asigna a `request.order`; de lo contrario, se retorna un error 404.
    """
    def wrapper(request, *args, **kwargs):
        try:
            order = Order.objects.get(pk=kwargs['order_pk'])
            request.order = order
        except Order.DoesNotExist:
            return JsonResponse({'error': 'Order not found'}, status=404)
        return func(request, *args, **kwargs)

    return wrapper
```

...

#### Descripción

Este decorador verifica la existencia de la orden especificada y la asigna a request.order. Si no existe, devuelve un error 404.

### validate_credit_card

```python
def validate_credit_card(func):
    """
    Valida los datos de la tarjeta de crédito enviados en la solicitud.

    Comprueba el formato del número de tarjeta, la fecha de expiración y el CVC.
    También verifica que la tarjeta no esté expirada.
    """
    def wrapper(request, *args, **kwargs):
        CARD_NUMBER_PATTERN = re.compile(r'^\d{4}-\d{4}-\d{4}-\d{4}$')
        EXP_DATE_PATTERN = re.compile(r'^(0[1-9]|1[0-2])\/\d{4}$')
        CVC_PATTERN = re.compile(r'^\d{3}$')
        card_number = request.json_body['card-number']
        exp_date = request.json_body['exp-date']
        cvc = request.json_body['cvc']
        if not CARD_NUMBER_PATTERN.match(card_number):
            return JsonResponse({'error': 'Invalid card number'}, status=400)
        if not EXP_DATE_PATTERN.match(exp_date):
            return JsonResponse({'error': 'Invalid expiration date'}, status=400)
        if not CVC_PATTERN.match(cvc):
            return JsonResponse({'error': 'Invalid CVC'}, status=400)
        card_exp_date = datetime.strptime(exp_date, '%m/%Y')
        current_date = datetime.now()
        if card_exp_date < current_date:
            return JsonResponse({'error': 'Card expired'}, status=400)
        return func(request, *args, **kwargs)

    return wrapper
```

...

#### Descripción

Este decorador valida los datos de la tarjeta de crédito enviados en la solicitud, asegurando que el formato sea correcto y que la tarjeta no esté expirada.

### validate_status

```python
def validate_status(func):
    """
    Valida el estado actual de una orden antes de permitir cambios.

    Impide modificar órdenes con estado `CANCELLED` o `COMPLETED`.
    """
    def wrapper(request, *args, **kwargs):
        if request.order.status == Order.Status.CANCELLED:
            return JsonResponse({'error': 'You cannot modify a canceled order.'}, status=400)
        if request.order.status == Order.Status.COMPLETED:
            return JsonResponse({'error': 'You cannot modify a completed order.'}, status=400)
        return func(request, *args, **kwargs)

    return wrapper
```

...

#### Descripción

Este decorador valida el estado de la orden antes de permitir cambios, impidiendo modificaciones en órdenes que ya han sido completadas o canceladas.

## Vistas

### user_order_list

```python
@csrf_exempt
@required_method('GET')
@verify_token
def user_order_list(request):
    """
    Recupera los detalles de todos los pedidos del usuario autenticado.

    Este endpoint permite a un usuario autenticado obtener la información de todas sus órdenes.
    """
    orders = Order.objects.filter(user=request.user)
    orders_serializer = [OrderSerializer(order).serialize() for order in orders]
    return JsonResponse(orders_serializer, safe=False, status=200)
```

...

#### Descripción

Este endpoint devuelve una lista de todas las órdenes del usuario autenticado.

### order_detail

```python
@login_required
@csrf_exempt
@required_method('GET')
@verify_order
@verify_user
def order_detail(request, order_pk: int):
    """
    Recupera los detalles de un pedido específico.

    Este endpoint permite a un usuario autenticado obtener la información de una orden,
    validando que el pedido exista y que pertenezca al usuario que realiza la solicitud.
    """
    serializer = OrderSerializer(request.order, request=request)
    return serializer.json_response()
```

...

#### Descripción

Este endpoint devuelve los detalles de una orden específica, validando que pertenezca al usuario autenticado.

### add_order

```python
@csrf_exempt
@required_method('POST')
@load_json_body
@verify_token
def add_order(request):
    """
    Crea una nueva orden de pedido.

    Este endpoint permite a un usuario autenticado generar una orden,
    siempre que exista suficiente stock para los productos solicitados.
    """
    order_data = request.json_body
    order = Order(user=request.user)
    order.save()
    total_price = Decimal('0.00')
    for item in order_data['products']:
        product_pk = item['id']
        quantity = item['quantity']
        try:
            product = Product.objects.get(pk=product_pk)
        except Product.DoesNotExist:
            return JsonResponse({'error': f'Product with id {product_pk} not found'}, status=404)
        if product.stock < quantity:
            return JsonResponse({'error': f'Insufficient stock for {product.name}'}, status=400)
        product.stock -= quantity
        product.save()
        order.products.add(product)
        total_price += product.price * Decimal(quantity)
    order.price = total_price
    order.save()

    return JsonResponse({'id': order.pk})
```

...

#### Descripción

Este endpoint permite a un usuario autenticado crear una nueva orden de pedido, validando la disponibilidad de stock para los productos solicitados.

### pay_order

```python
@csrf_exempt
@required_method('POST')
@load_json_body
@required_fields('card-number', 'exp-date', 'cvc', model=Order)
@verify_token
@verify_order
@validate_credit_card
@verify_user
@validate_status
def pay_order(request, order_pk: int):
    """
    Procesa el pago de una orden.

    Cambia el estado de la orden a 'COMPLETED' si se valida correctamente la tarjeta.
    """
    request.order.status = Order.Status.COMPLETED
    request.order.save()
    return JsonResponse({'msg': 'Your order has been paid and complete successfully'})
```

...

#### Descripción

Este endpoint permite procesar el pago de una orden, cambiando su estado a 'COMPLETED' si la tarjeta de crédito es válida.

### cancel_order

```python
@csrf_exempt
@required_method('POST')
@load_json_body
@verify_token
@verify_order
@verify_user
@validate_status
def cancel_order(request, order_pk: int):
    """
    Cancela una orden de pedido.

    Cambia el estado de la orden a 'CANCELLED' y repone el stock de los productos involucrados.
    """
    order_data = request.json_body
    for item in order_data['products']:
        product_pk = item['id']
        quantity = item['quantity']
        try:
            product = Product.objects.get(pk=product_pk)
        except Product.DoesNotExist:
            return JsonResponse({'error': f'Product with id {product_pk} not found'}, status=404)
        product.stock += quantity
        product.save()
    request.order.status = Order.Status.CANCELLED
    request.order.save()
    return JsonResponse({'msg': f'Order {order_pk} has been cancelled'})
```

...

#### Descripción

Este endpoint permite cancelar una orden de pedido, cambiando su estado a 'CANCELLED' y reponiendo el stock de los productos.

### get_earnings

```python
@csrf_exempt
@required_method('GET')
@verify_token
@verify_admin
def get_earnings(request):
    """
    Obtiene las ganancias diarias del mes actual para todas las órdenes completadas.
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
        orders = Order.objects.filter(created_at__date=date, status=Order.Status.COMPLETED)
        earnings = sum(order.price for order in orders)

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

Este endpoint permite a los administradores obtener un resumen de las ganancias diarias del mes actual para todas las órdenes completadas.

### URLs

```python

urlpatterns = [
    path('', views.user_order_list, name='user_order_list'),
    path('add/', views.add_order, name='add-order'),
    path('get-earnings/', views.get_earnings, name='get-earnings'),
    path('<order_pk>/', views.order_detail, name='order-detail'),
    path('<order_pk>/pay-order/', views.pay_order, name='pay-order'),
    path('<order_pk>/cancel-order/', views.cancel_order, name='cancel-order'),
]
```

...
