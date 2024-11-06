# FWD Repaso General (Ejercicio de Reserva de Vuelos)

### Paso 1: Crear el Proyecto en Django y Configurar MySQL

1. **Crear un proyecto Django**:
   ```bash
   django-admin startproject flight_reservation
   cd flight_reservation
   ```

2. **Crear una aplicación**:
   ```bash
   python manage.py startapp api
   ```

3. **Instalar las dependencias**:
   - Asegúrate de tener instalados Django y MySQL.
   ```bash
   pip install django mysqlclient djangorestframework djangorestframework-jwt
   ```

4. **Configurar la base de datos MySQL** en `settings.py`:
   ```python
   DATABASES = {
       'default': {
           'ENGINE': 'django.db.backends.mysql',
           'NAME': 'nombre_bd',
           'USER': 'tu_usuario',
           'PASSWORD': 'tu_contraseña',
           'HOST': 'localhost',
           'PORT': '3306',
       }
   }
   ```

5. **Añadir las aplicaciones y configuraciones necesarias en `settings.py`**:
   ```python
   INSTALLED_APPS = [
       'django.contrib.admin',
       'django.contrib.auth',
       'django.contrib.contenttypes',
       'django.contrib.sessions',
       'django.contrib.messages',
       'django.contrib.staticfiles',
       'rest_framework',
       'api',  # Nuestra app
   ]

   REST_FRAMEWORK = {
       'DEFAULT_AUTHENTICATION_CLASSES': (
           'rest_framework_jwt.authentication.JSONWebTokenAuthentication',
       ),
   }
   ```

### Paso 2: Crear el Modelo de Usuario y Configurar JWT

1. **Configurar JWT** en `settings.py`:
   ```python
   import datetime

   JWT_AUTH = {
       'JWT_SECRET_KEY': 'tu_clave_secreta',
       'JWT_EXPIRATION_DELTA': datetime.timedelta(days=1),
   }
   ```

2. **Configurar el modelo `User`** en `models.py`:
   Django ya cuenta con un modelo `User`, así que podemos extenderlo para personalizarlo si es necesario.

3. **Realizar la migración inicial** para crear las tablas:
   ```bash
   python manage.py makemigrations
   python manage.py migrate
   ```

### Paso 3: Crear Modelos de Vuelo y Reservas

En `models.py` dentro de la aplicación `api`, define los modelos de `Flight` y `Reservation`:

```python
from django.db import models
from django.contrib.auth.models import User

class Flight(models.Model):
    origin = models.CharField(max_length=100)
    destination = models.CharField(max_length=100)
    departure_time = models.DateTimeField()
    arrival_time = models.DateTimeField()
    price = models.DecimalField(max_digits=10, decimal_places=2)

class Reservation(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    flight = models.ForeignKey(Flight, on_delete=models.CASCADE)
    status = models.CharField(max_length=20, default='confirmado')
```

Ejecuta migraciones para estos modelos:

```bash
python manage.py makemigrations api
python manage.py migrate
```

### Paso 4: Serializadores y Vistas

1. **Crear serializadores** en `api/serializers.py`:

   ```python
   from rest_framework import serializers
   from .models import Flight, Reservation
   from django.contrib.auth.models import User

   class FlightSerializer(serializers.ModelSerializer):
       class Meta:
           model = Flight
           fields = '__all__'

   class ReservationSerializer(serializers.ModelSerializer):
       class Meta:
           model = Reservation
           fields = '__all__'

   class UserSerializer(serializers.ModelSerializer):
       class Meta:
           model = User
           fields = ('username', 'email', 'password')
       extra_kwargs = {'password': {'write_only': True}}

       def create(self, validated_data):
           user = User(
               email=validated_data['email'],
               username=validated_data['username']
           )
           user.set_password(validated_data['password'])
           user.save()
           return user
   ```

2. **Crear vistas** en `api/views.py` para el registro y autenticación:

   ```python
   from rest_framework import viewsets, status
   from rest_framework.response import Response
   from rest_framework.permissions import IsAuthenticated
   from .models import Flight, Reservation
   from .serializers import FlightSerializer, ReservationSerializer, UserSerializer
   from django.contrib.auth.models import User
   from rest_framework_jwt.settings import api_settings
   from rest_framework.decorators import action

   class UserViewSet(viewsets.ModelViewSet):
       queryset = User.objects.all()
       serializer_class = UserSerializer

       @action(detail=False, methods=['post'])
       def register(self, request):
           serializer = UserSerializer(data=request.data)
           if serializer.is_valid():
               user = serializer.save()
               return Response({'user': UserSerializer(user).data}, status=status.HTTP_201_CREATED)
           return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

   class FlightViewSet(viewsets.ReadOnlyModelViewSet):
       queryset = Flight.objects.all()
       serializer_class = FlightSerializer

   class ReservationViewSet(viewsets.ModelViewSet):
       queryset = Reservation.objects.all()
       serializer_class = ReservationSerializer
       permission_classes = [IsAuthenticated]

       def perform_create(self, serializer):
           serializer.save(user=self.request.user)
   ```

### Paso 5: Configurar URLs

1. **Configurar las rutas** en `api/urls.py`:

   ```python
   from django.urls import path, include
   from rest_framework.routers import DefaultRouter
   from .views import UserViewSet, FlightViewSet, ReservationViewSet
   from rest_framework_jwt.views import obtain_jwt_token

   router = DefaultRouter()
   router.register(r'users', UserViewSet)
   router.register(r'flights', FlightViewSet)
   router.register(r'reservations', ReservationViewSet)

   urlpatterns = [
       path('api/', include(router.urls)),
       path('api/auth/login/', obtain_jwt_token),
   ]
   ```

2. **Incluir las rutas de la aplicación** en `flight_reservation/urls.py`:

   ```python
   from django.contrib import admin
   from django.urls import path, include

   urlpatterns = [
       path('admin/', admin.site.urls),
       path('', include('api.urls')),
   ]
   ```

### Paso 6: Pruebas de la API

Ejecuta el servidor y prueba los endpoints:

1. **Registro de usuario**:
   ```plaintext
   POST /api/users/register/
   Body: { "username": "user1", "email": "user1@example.com", "password": "password123" }
   ```

2. **Inicio de sesión**:
   ```plaintext
   POST /api/auth/login/
   Body: { "username": "user1", "password": "password123" }
   ```

3. **Rutas protegidas (Reservas)**:
   - Crear una reserva:
     ```plaintext
     POST /api/reservations/
     Header: Authorization: JWT <token_obtenido_en_login>
     Body: { "flight": 1, "status": "confirmado" }
     ```
