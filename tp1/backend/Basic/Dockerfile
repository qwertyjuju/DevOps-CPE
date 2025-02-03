FROM   amazoncorretto:8-alpine3.21
COPY ./backend/Basic/Main.java /srv
WORKDIR /srv
RUN javac Main.java
CMD ["java", "Main"]