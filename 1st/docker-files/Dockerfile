FROM ubuntu:24.04

RUN apt-get update && apt-get install -y openjdk-17-jdk

WORKDIR /app
COPY ./demo.jar ./app.jar

CMD ["java", "-jar", "app.jar"]

# 컨테이너 포트를 의미함
EXPOSE 8080