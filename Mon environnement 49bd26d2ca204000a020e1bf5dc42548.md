# Mon environnement

Pour l’environnement, j’ai réutilisé le docker compose existant en lui rajoutant des conditions de healthcheck notamment sur le orion-ld qui indique être started avant ready, en effet, je conteneur orion-ld se marque comme lancé avant d’avoir finis son initialisation ce qui provoque des erreur dans les autre conteneurs lors de leur lancement. 

Les modification que j’ai effectué sont les suivantes:

D’abord, j’ai ajouté la condition de healthchecking au conteneur timescale en lui ajoutant le champs suivant:

```yaml
healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 10s
      timeout: 5s
      retries: 5
```

ce qui donne:

```yaml
timescale:
    image: timescale/timescaledb-postgis:1.7.5-pg12
    ports:
      - "5455:5432"
    environment:
      - 'POSTGRES_HOST_AUTH_METHOD=trust'
      - 'POSTGRES_PASSWORD=orion'
      - 'POSTGRES_USER=orion'
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 10s
      timeout: 5s
      retries: 5
    volumes:
      - ./timescale:/var/lib/postgresql/data
```

Puis, je précisé dans les dépendances de Orion-LD que le timescale devait être healthy pour pouvoir continuer:

```yaml
orion-ld:
    image: fiware/orion-ld:1.0.0-PRE-588
    command: -dbhost mongo-ld -corsOrigin __ALL -writeConcern 0 -notificationMode threadpool:10000:1000 -reqMutexPolicy none -httpTimeout 60000 -reqPoolSize 16 -logLevel FATAL -maxConnections 20000 -lmtmp
    environment:
      - 'ORIONLD_DISABLE_FILE_LOG=TRUE'
      - 'ORIONLD_LOG_LEVEL=DEBUG'
      - 'ORIONLD_TROE_USER=orion'
      - 'ORIONLD_TROE_PWD=orion'
      - 'ORIONLD_TROE=TRUE'
      - 'ORIONLD_TROE_HOST=timescale'    
    ports:
      - "1026:1026"
    depends_on:
      mongo-ld:
        condition: service_started
      timescale: 
        condition: service_healthy
```

Pour nodered, j’ai utilisé un conteneur docker pour faciliter l’installation ce qui peut expliquer des hostname de la forme `host.docker.interal`