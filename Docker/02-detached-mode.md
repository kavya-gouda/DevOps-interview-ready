# detached mode

Ever started a database container only to have it die the moment you close your terminal? Been there, done that!

The magic flag you need is -d (detached mode):

```
docker run -d postgres
```
This runs your container in the background, completely independent of your terminal session. Think of it as hiring a reliable employee who works even when you're not watching! 

## Use Cases:

Beginner: Running a local PostgreSQL database for your side project
docker run -d --name my-db postgres

Seasoned Pro #1: Multi-service application with detached database and Redis
docker run -d --name prod-db -e POSTGRES_PASSWORD=secret postgres
docker run -d --name cache redis

Seasoned Pro #2: Microservices deployment with network isolation 
docker run -d --network app-network --name user-db postgres

Pro Tip: Remember -d as "Don't wait" - you don't want to wait around watching your database start up, you want it running in the background while you focus on building your application!

Common gotcha: Without -d, your container runs in foreground mode and stops when you close the terminal or hit Ctrl+C. With -d, it keeps running until you explicitly stop it.

Perfect for background services, web servers, databases, and anything that needs to stay alive 24/7

