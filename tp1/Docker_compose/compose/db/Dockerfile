FROM postgres:14.1-alpine
COPY CreateScheme.sql /docker-entrypoint-initdb.d
COPY InsertData.sql /docker-entrypoint-initdb.d
CMD ["postgres"]