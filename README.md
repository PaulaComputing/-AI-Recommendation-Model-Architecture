# Sistema de Recomendación E-commerce en Tiempo Real (PyTorch NeuMF + AWS EC2)

Arquitectura distribuida de microservicios para inferencia de recomendaciones personalizadas en tiempo real mediante **Neural Collaborative Filtering (NeuMF)** entrenado en **PyTorch**, optimizada con caché en memoria (**Redis**), mensajería asíncrona orientada a eventos (**RabbitMQ**) e interfaz interactiva (**Streamlit**), desplegada sobre una instancia **AWS EC2** con **Docker Compose**.

---

## Arquitectura del Sistema

El ecosistema está orquestado en 5 contenedores independientes:

| Servicio | Contenedor | Puerto | Función Principal |
| :--- | :--- | :--- | :--- |
| **API REST** | `ep3-api` | `8000` | Endpoints FastAPI para inferencia del modelo NeuMF, healthcheck y publicación de eventos. |
| **Dashboard** | `ep3-dashboard` | `8501` | Interfaz visual en Streamlit para consultar recomendaciones e interpretar afinidades. |
| **Caché** | `ep3-redis` | `6379` | Almacenamiento en memoria de predicciones recientes para reducir la latencia de respuesta. |
| **Message Broker** | `ep3-rabbitmq` | `5672` / `15672` | Cola de mensajes para desacoplar eventos de interacción de usuario (`view`, `add_to_cart`, `purchase`). |
| **Worker** | `ep3-worker` | Interno | Consumidor asíncrono que procesa eventos desde RabbitMQ e invalida/actualiza el estado de caché. |

### Flujo de Datos
1. **Consulta de Recomendaciones (`POST /api/v1/recommendations`):** La API verifica si existe una respuesta vigente en **Redis**. Si no existe (*cache miss*), ejecuta la inferencia en el modelo **PyTorch NeuMF** y almacena el resultado.
2. **Eventos de Usuario (`POST /api/v1/events`):** Las interacciones del usuario se publican en **RabbitMQ**, son consumidas por el **Worker** en segundo plano e invalidan la caché del usuario para refrescar sus recomendaciones.

---

## Guía Paso a Paso del Despliegue en AWS EC2

### Paso 1: Aprovisionamiento y Configuración de Red (Security Groups)
1. Se creó una instancia EC2 (`servidor-ep3-recomendaciones`) de tipo **`t3.medium`** con sistema operativo **Ubuntu 22.04 LTS**.
2. En la pestaña **Seguridad > Grupos de seguridad > Editar reglas de entrada**, se habilitaron los siguientes puertos con origen `0.0.0.0/0` (*Cualquier lugar-IPv4*):
   * **Puerto `22` (SSH):** Para acceso remoto mediante *EC2 Instance Connect*.
   * **Puerto `8000` (TCP personalizado):** Para exponer la API REST de FastAPI y Swagger UI.
   * **Puerto `8501` (TCP personalizado):** Para exponer el Dashboard visual de Streamlit.
3. Una vez en ejecución, AWS asignó la dirección IPv4 pública **`18.212.91.247`**.

### Paso 2: Conexión e Instalación de Dependencias en Ubuntu
1. Desde la consola de AWS, se accedió mediante **Conectar > Conexión de la instancia EC2** con el usuario predeterminado `ubuntu`.
2. En la terminal del servidor se instaló Docker, Docker Compose y Git, y se otorgaron permisos al usuario `ubuntu`:

```bash
sudo apt-get update
sudo apt-get install -y docker.io docker-compose-v2 git
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ubuntu
newgrp docker
```

### Paso 3: Clonación del Repositorio y Orquestación de Contenedores
1. Se descargó el proyecto limpio desde GitHub directamente al servidor EC2:

```bash
git clone [https://github.com/PaulaComputing/AI-Model-Architecture---Recommendations.git](https://github.com/PaulaComputing/AI-Model-Architecture---Recommendations.git)
cd AI-Model-Architecture---Recommendations
```

2. Se construyeron las imágenes y se levantaron los 5 microservicios en segundo plano:

```bash
docker compose up -d --build
```

3. Se verificó que todos los contenedores estuvieran activos en estado `Up`:

```bash
docker compose ps
```

### Paso 4: Acceso Web a los Servicios Desplegados
> **Importante:** El acceso desde el navegador se realiza obligatoriamente mediante protocolo **`http://`** (sin `https://`), apuntando directamente a la dirección IPv4 pública de la instancia EC2 y al puerto correspondiente.

* **Documentación Interactiva de la API (FastAPI / Swagger UI):** `http://18.212.91.247:8000/docs`
* **Endpoint de Health Check:** `http://18.212.91.247:8000/health`
* **Dashboard Interactivo (Streamlit):** `http://18.212.91.247:8501`

*(Nota: En entornos de laboratorio como AWS Academy, al detener y volver a iniciar la instancia EC2 se asignará una nueva dirección IPv4 pública; para reactivar el sistema basta con conectarse a la terminal, entrar a `AI-Model-Architecture---Recommendations` y ejecutar `docker compose up -d`).*

---

## Ejecución de Pruebas Funcionales y de Rendimiento (`test/`)

El proyecto incluye pruebas automatizadas de integración y carga concurrente dentro de la carpeta `test/`:

### 1. Pruebas de Integración End-to-End (`test/test_api.py`)
Valida el funcionamiento completo de la arquitectura: disponibilidad (`GET /health`), primera inferencia con PyTorch, lectura acelerada desde Redis e invalidación de caché mediante eventos en RabbitMQ.

```bash
pip install requests
python test/test_api.py
```

*(Si `test_api.py` apunta por defecto a `localhost:8000`, reemplaza la URL base por `http://18.212.91.247:8000` para probar directamente contra la nube).*

### 2. Pruebas de Estrés y Concurrencia (`test/locustfile.py`)
Simula múltiples usuarios concurrentes realizando consultas de recomendación y enviando interacciones de compra/vista hacia la API desplegada en AWS.

* **Ejecución con interfaz web de Locust (abrir `http://localhost:8089` en el navegador y configurar 50 usuarios con tasa 5/s):**

```bash
pip install locust
locust -f test/locustfile.py --host=[http://18.212.91.247:8000](http://18.212.91.247:8000)
```

* **Ejecución directa por consola (modo headless):**

```bash
locust -f test/locustfile.py --host=[http://18.212.91.247:8000](http://18.212.91.247:8000) --headless -u 50 -r 5 -t 30s
```

---

## Resultados y Evidencia de Despliegue

### Métricas Observadas en Producción (EC2 `t3.medium`)
* **Inferencia directa en modelo PyTorch (`PyTorch-Inference-EC2`):** `14.27 ms`
* **Respuesta desde memoria (`Redis-Cache`):** `0.23 ms` *(reducción de ~98 % en tiempo de respuesta)*
* **Tasa de error bajo concurrencia:** `0 %`

### Capturas del Sistema Operativo - Carpeta Capturas_Relacionadas
1. **API REST y Endpoints (Swagger UI):**
   ![API Swagger](evidencias/01_api_swagger_endpoints.png)

2. **Validación de Estado (`GET /health` - HTTP 200):**
   ![Healthcheck 200](evidencias/02_api_healthcheck_200.png)

3. **Dashboard Interactivo (Streamlit):**
   ![Dashboard Inicio](evidencias/03_dashboard_inicio.png)

4. **Inferencia en Tiempo Real con PyTorch NeuMF:**
   ![Inferencia PyTorch](evidencias/04_inferencia_pytorch_ec2.png)

5. **Aceleración con Caché Distribuida Redis (`0.23 ms`):**
   ![Respuesta Redis](evidencias/05_respuesta_redis_cache.png)

6. **Publicación de Eventos en RabbitMQ e Invalidación de Caché:**
   ![Evento RabbitMQ](evidencias/06_evento_rabbitmq_cola.png)

7. **Orquestación de Contenedores en AWS EC2 (`docker compose ps`):**
   ![Contenedores EC2](evidencias/09_docker_compose_ps.png)