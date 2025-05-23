# Documentación de la Aplicación Products

## Introducción

La aplicación `products` gestiona los productos disponibles para la venta. Permite a los administradores crear, editar, eliminar y listar productos.

## Modelos

### Product

```python
from django.db import models

class Product(models.Model):
    """
    Modelo que representa un producto disponible para la venta.

    Atributos
    ----------
    name : CharField
        Nombre del producto.
    description : TextField
        #### Descripción opcional del producto.
    price : DecimalField
        Precio del producto, con hasta 8 dígitos y 2 decimales.
    stock : PositiveIntegerField
        Cantidad disponible en inventario.
    image : ImageField
        Imagen del producto. Puede ser personalizada o usar una imagen por defecto.
    """
    name = models.CharField(max_length=100)
    description = models.TextField(blank=True, null=True)
    price = models.DecimalField(max_digits=8, decimal_places=2)
    stock = models.PositiveIntegerField()
    image = models.ImageField(
        upload_to='media/product_images/',
        default='products_images/no_product.png',
        blank=True,
        null=True,
    )

    def __str__(self):
        return self.name

```

#### Descripción

El modelo Product representa un producto disponible para la venta, incluyendo atributos como nombre, descripción, precio, stock y una imagen.

## Serializadores

### ProductSerializer

```python
from shared.serializers import BaseSerializer

class ProductSerializer(BaseSerializer):
    """
    Serializador para el modelo Product.

    Métodos
    -------
    serialize_instance(instance) -> dict
        Serializa una instancia de Product en un diccionario con sus atributos principales.
    """
    def serialize_instance(self, instance) -> dict:
        return {
            'id': instance.id,
            'name': instance.name,
            'description': instance.description,
            'price': instance.price,
            'stock': instance.stock,
            'image': f'http://localhost:8000{self.build_url(instance.image.url)}',
        }
```

...

#### Descripción

El ProductSerializer se utiliza para serializar instancias del modelo Product, proporcionando una representación en formato JSON de los atributos relevantes.

## Decoradores

### verify_product

```python
from django.http import JsonResponse
from .models import Product

def verify_product(func):
    """
    Decorador que verifica la existencia de un producto por su ID.

    Este decorador intenta recuperar un producto desde la base de datos utilizando
    el parámetro 'product_pk' proporcionado en la URL. Si el producto existe, se
    adjunta al objeto `request` como `request.product`. Si no existe, retorna un error 404.
    """
    def wrapper(request, *args, **kwargs):
        try:
            product = Product.objects.get(id=kwargs['product_pk'])
            request.product = product
        except Product.DoesNotExist:
            return JsonResponse({'error': 'Product not found'}, status=404)
        return func(request, *args, **kwargs)

    return wrapper
```

...

#### Descripción

Este decorador verifica la existencia de un producto en la base de datos utilizando el product_pk proporcionado en la URL. Si no existe, devuelve un error 404.

## Vistas

### product_list

```python
@required_method('GET')
def product_list(request):
    """
    Devuelve una lista de todos los productos en formato JSON.

    Este endpoint permite obtener todos los productos registrados en la base de datos.
    """
    products = ProductSerializer(Product.objects.all())
    return products.json_response()
```

...

#### Descripción

Este endpoint devuelve una lista de todos los productos registrados en la base de datos en formato JSON.

### product_detail

```python
@csrf_exempt
@required_method('GET')
@verify_product
def product_detail(request, product_pk):
    """
    Devuelve los detalles de un producto específico.

    Este endpoint permite obtener los detalles de un producto utilizando su ID.
    """
    serializer = ProductSerializer(request.product, request=request)
    return serializer.json_response()
```

...

#### Descripción

Este endpoint devuelve los detalles de un producto específico utilizando su ID.

### add_product

```python
@csrf_exempt
@required_method('POST')
@load_json_body
@required_fields('name', 'description', 'price', 'stock', model=Product)
@verify_token
@verify_admin
def add_product(request):
    """
    Agrega un nuevo producto.

    Este endpoint permite a un administrador autenticado crear un nuevo producto
    proporcionando el nombre, descripción, precio y stock del producto.
    """
    name = request.json_body['name']
    description = request.json_body['description']
    price = request.json_body['price']
    stock = request.json_body['stock']
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

    product = Product.objects.create(
        name=name, description=description, price=price, stock=stock, image=image_file
    )
    return JsonResponse({'id': product.pk})
```

...

#### Descripción

Este endpoint permite a un administrador autenticado crear un nuevo producto proporcionando los datos necesarios.

### edit_product

```python
@csrf_exempt
@required_method('POST')
@load_json_body
@required_fields('name', 'description', 'price', 'stock', model=Product)
@verify_token
@verify_admin
@verify_product
def edit_product(request, product_pk: int):
    """
    Edita un producto existente.

    Este endpoint permite a un administrador autenticado editar un producto
    existente proporcionando los nuevos datos del producto.
    """
    product = request.product
    product.name = request.json_body['name']
    product.description = request.json_body['description']
    product.price = request.json_body['price']
    product.stock = request.json_body['stock']
    image_base64 = request.json_body.get('image')
    if image_base64:
        try:
            format_part, data_part = image_base64.split(',')
            file_format = format_part.split('/')[1].split(';')[0]

            image_data = base64.b64decode(data_part)

            filename = f'service_{uuid.uuid4().hex[:8]}.{file_format}'
            image_file = ContentFile(image_data, name=filename)
            product.image = image_file
        except Exception as e:
            return JsonResponse({'error': f'Error procesando la imagen: {str(e)}'}, status=400)

    product.save()
    return JsonResponse({'msg': 'El producto ha sido editado'})
```

...

#### Descripción

Este endpoint permite a un administrador autenticado editar un producto existente.

### delete_product

```python
@csrf_exempt
@required_method('POST')
@verify_token
@verify_admin
@verify_product
def delete_product(request, product_pk: int):
    """
    Elimina un producto existente.

    Este endpoint permite a un administrador autenticado eliminar un producto
    específico utilizando su ID.
    """
    product = request.product
    product.delete()
    return JsonResponse({'msg': 'El producto ha sido eliminado'})
```

...

#### Descripción

Este endpoint permite a un administrador autenticado eliminar un producto específico.

## URLs

```python

name = 'products'
urlpatterns = [
    path('', views.product_list, name='product-list'),
    path('add/', views.add_product, name='add_product'),
    path('<int:product_pk>/', views.product_detail, name='product-detail'),
    path('<int:product_pk>/edit/', views.edit_product, name='edit-product'),
    path('<int:product_pk>/delete/', views.delete_product, name='delete-product'),
]
```

...
