# Indice del Backend

Aqui puedes ver todas las applicaciones de django que tienen una importancia clave en el desarrollo de la aplicacio Web

### Indice

- [Aplicacion compartida](shared.md)
- [Aplicacion de Autenticacion](accounts.md)
- [Aplicacion de reservas](bookings.md)
- [Aplicacion de eventos](events.md)
- [Aplicacion de pedidos](orders.md)
- [Aplicacion de productos](products.md)
- [Aplicacion de servicios](services.md)
- [Aplicacion de usuarios](users.md)
- [Aplicacion de Bot de telegram](tele_bot.md)

### Urls principales

```python

urlpatterns = [
    path('admin/', admin.site.urls),
    path('login/', accounts.views.user_login, name='login'),
    path('logout/', accounts.views.user_logout, name='logout'),
    path('signup/', accounts.views.user_signup, name='signup'),
    path('api/user/', users.views.get_user_profile, name='user'),
    path('api/users-per-mounth/', users.views.users_per_mounth, name='users-per-mounth'),
    path('api/barbers/', users.views.get_barbers, name='barber'),
    path('api/bookings/', include('bookings.urls')),
    path('api/products/', include('products.urls')),
    path('api/services/', include('services.urls')),
    path('api/events/', include('events.urls')),
    path('api/orders/', include('orders.urls')),
]
if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)

```
