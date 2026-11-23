# syntax=docker/dockerfile:1

# Le jar est construit sur l'hôte (./mvnw clean package -DskipTests) avant le build de l'image :
# ça évite de refaire tourner Maven (et de retélécharger les dépendances) dans Docker à chaque fois.
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY target/account-*.jar app.jar
ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
