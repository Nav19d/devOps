FROM postgres:14.1-alpine

ENV POSTGRES_DB=db \
   POSTGRES_USER=Navid \
   POSTGRES_PASSWORD=1234

COPY 01-CreateScheme.sql /docker-entrypoint-initdb.d
 
COPY 02-InsertData.sql /docker-entrypoint-initdb.d