# vidasalud-infra

Archivos de Docker Compose para levantar la infraestructura de VidaSalud: un **clúster RabbitMQ de 2 nodos** y los microservicios que lo usan (`appointments` y `notify`).

## Contenido

| Archivo | Descripción |
|---|---|
| `docker-compose.yml` | Nodos `rabbitmq1` y `rabbitmq2` en clúster, con panel de administración, más los servicios `appointments` y `notify` (perfil `apps`) |
| `rabbitmq/rabbitmq.conf` | Formación automática del clúster (`classic_config`) y manejo de particiones (`autoheal`) |
| `.env.example` | Usuario, contraseña y Erlang cookie compartida por los nodos |

## Cómo funciona el clúster

- Ambos nodos usan la misma **Erlang cookie** (`RABBITMQ_ERLANG_COOKIE`), requisito para que puedan comunicarse.
- `rabbitmq.conf` declara los dos miembros (`rabbit@rabbitmq1` y `rabbit@rabbitmq2`). `rabbitmq2` arranca cuando `rabbitmq1` está sano y se une automáticamente a su clúster.
- Los microservicios se conectan con `RABBITMQ_ADDRESSES=rabbitmq1:5672,rabbitmq2:5672`: si un nodo no responde, el cliente se conecta al otro.
- Las colas, los exchanges y los bindings no se definen aquí. Los declaran los microservicios al arrancar (ver `vidasalud-ms-notify`).

| Nodo | AMQP | Panel web |
|---|---|---|
| rabbitmq1 | `5672` | http://localhost:15672 |
| rabbitmq2 | `5673` | http://localhost:15673 |

## Uso

Los repos deben estar clonados uno al lado del otro:

```
VidaSalud/
├── vidasalud-infra/            ← este repo
├── vidasalud-ms-appointments/
└── vidasalud-ms-notify/
```

```bash
cp .env.example .env          # y cambia las claves

# Solo el clúster RabbitMQ
docker compose up -d

# Clúster + microservicios appointments y notify
docker compose --profile apps up -d --build

# Ver los nodos del clúster (deben aparecer los dos en "Running Nodes")
docker exec rabbitmq1 rabbitmqctl cluster_status

# Ver colas, exchanges y bindings
docker exec rabbitmq1 rabbitmqctl list_queues name messages consumers
docker exec rabbitmq1 rabbitmqctl list_exchanges name type
docker exec rabbitmq1 rabbitmqctl list_bindings source_name destination_name routing_key

# Detener
docker compose --profile apps down
```

## Despliegue en la nube (EC2)

1. Instala Docker y el plugin de Compose en la instancia.
2. Clona `vidasalud-infra`, `vidasalud-ms-appointments` y `vidasalud-ms-notify` en la misma carpeta.
3. Crea el `.env` con claves seguras.
4. Ejecuta `docker compose --profile apps up -d --build`.
5. En el Security Group abre solo lo necesario: `15672` (panel, limitado a tu IP) y `8081`/`8083` si el BFF o el API Gateway deben llegar a los servicios. **No expongas `5672` a internet.**

Si `appointments` ya corre en la instancia fuera de Compose, detén ese contenedor antes, para que no choquen los puertos. Revisa también que la variable `APPOINTMENTS_URL` del BFF siga apuntando a él.

> **Nota:** con 2 nodos, las colas clásicas viven en el nodo donde se crearon. El clúster comparte usuarios, exchanges y la definición de las colas, pero si cae el nodo dueño de una cola, esa cola queda no disponible hasta que vuelva.
