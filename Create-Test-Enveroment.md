# Create test django enveroment 


```bash
source .venv/bin/activate
source .env
docker run -id --name postgres -e POSTGRES_PASSWORD=$POSTGRES_PASSWORD -e POSTGRES_USER=$POSTGRES_USER -p 5432:5432  postgres
docker run -i -d --name redis -p 6379:6379 redis:5-alpine 
```
