# Étape 1 : compilation avec le JDK
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /usr/src
COPY Main.java .
RUN javac Main.java

# Étape 2 : exécution avec le JRE seulement
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=build /usr/src/Main.class .
CMD ["java", "Main"]