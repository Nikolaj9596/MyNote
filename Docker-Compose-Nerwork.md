
## Настройка позволяющая добавить сеть в docker-compose.yaml

```yaml

  web-stage:
    container_name: web-stage
    image: web-stage
    build: .
    volumes:
      - .:/app
      - static_volume:/app/static/
    expose:
      - 8000
    env_file:
      - ./.env
    networks:
      - web-dev-network 
      - default
    restart: always

networks:
   web-dev-network:
    name: web_default
    external: true
```
