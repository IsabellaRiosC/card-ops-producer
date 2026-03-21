# Resumen del chat tecnico - card-ops-producer

Fecha: 2026-03-21

## Objetivo general
Documentar y resolver errores de build/runtime en `card-ops-producer` (tests, Spring Boot en local y Docker Compose), incluyendo OTEL, healthchecks y configuracion.

## Problemas reportados y causa raiz

### 1) Contenedor `otel-collector` no encontraba config
- Error visto: `unable to read the file ... /etc/otel-collector-config.yml: is a directory`.
- Hallazgo posterior: en el estado actual local, `D:\bootcamp-bella\card-ops-producer\otel-collector-config.yml` es archivo (no directorio).
- Estado actual: el collector inicia y muestra `Everything is ready. Begin running and processing data.`

### 2) Error de tipos `ObjectMapper`
- Error: `incompatible types: com.fasterxml.jackson.databind.ObjectMapper cannot be converted to tools.jackson.databind.ObjectMapper`.
- Causa: mezcla de imports entre paquetes `com.fasterxml.jackson...` y `tools.jackson...`.
- Solucion aplicada: alinear `AttemptPolicyOrchestrator` y su test a `tools.jackson.databind.ObjectMapper` para compatibilidad con Boot 4 del proyecto.

### 3) Fallos de tests por `assertThat`
- Error: `cannot find symbol assertThat(...)` en `EventMapperTest`.
- Solucion aplicada: agregar `import static org.assertj.core.api.Assertions.assertThat;`.
- Verificacion: `EventMapperTest` pasa.

### 4) Fallo Mockito por checked exception invalida
- Error: `Checked exception is invalid for this method!` en `AttemptPolicyOrchestratorTest`.
- Causa: `thenThrow` con checked exception incompatible con firma del metodo.
- Solucion aplicada: cambiar a `new RuntimeException("boom")` y limpiar import no usado.
- Verificacion: `AttemptPolicyOrchestratorTest` pasa (5 tests OK).

### 5) En Docker: `No qualifying bean of type 'com.fasterxml.jackson.databind.ObjectMapper'`
- Causa: imagen/JAR desactualizado (el contenedor seguia con artefacto viejo).
- Solucion aplicada: rebuild sin cache + recreate del servicio.
- Verificacion: arranque correcto del servicio sin ese error.

### 6) `503` en `/actuator/health`
- Causa raiz: `redis` aparecia `DOWN` porque `spring.data.redis` estaba mal indentado dentro de `spring.application` en `application.yaml`.
- Solucion aplicada:
  - mover `spring.data.redis` al nivel correcto,
  - corregir `server.port` a bloque `server.port`.
- Verificacion: `/actuator/health` quedo en `UP` y `redis` en `UP`.

### 7) `zookeeper` en estado `unhealthy`
- Causa: healthcheck previo basado en `ruok` no alineado con comandos habilitados en esa imagen.
- Solucion aplicada: healthcheck actualizado a endpoint AdminServer `/commands/srvr`.
- Verificacion: `zookeeper` en `healthy`.

### 8) Warning de Compose por `version` obsoleto
- Solucion aplicada: remover `version: "3.9"` de `docker-compose.yml`.
- Verificacion: `docker compose config` valido sin ese warning.

### 9) Propiedad mal escrita de Resilience4j
- Hallazgo: `resilieindicamence4j` (typo).
- Solucion aplicada: corregido a `resilience4j` para que aplique la configuracion de `@CircuitBreaker`/`@Retry` de `ResilientPublisher`.

## Archivos tocados durante la sesion
- `src/main/java/com/bank/card_ops_producer/domain/policy/AttemptPolicyOrchestrator.java`
- `src/test/java/com/bank/card_ops_producer/domain/policy/AttemptPolicyOrchestratorTest.java`
- `src/test/java/com/bank/card_ops_producer/domain/mapper/EventMapperTest.java`
- `src/main/java/com/bank/card_ops_producer/api/dto/CardReplacementRequestDto.java`
- `src/main/resources/application.yaml`
- `docker-compose.yml`

## Validaciones destacadas ejecutadas
- Test focalizado:
  - `mvnw.cmd -Dtest=EventMapperTest test` -> OK
  - `mvnw.cmd -Dtest=AttemptPolicyOrchestratorTest test` -> OK
  - `mvnw.cmd -Dtest=CardReplacementControllerTest test` -> OK
- Runtime local:
  - `mvnw.cmd spring-boot:run "-Dspring-boot.run.arguments=--spring.main.web-application-type=none"` -> arranque OK
- Docker:
  - rebuild y recreate de `card-ops-producer`
  - `docker inspect` reportando `health=healthy` en `zookeeper` y `card-ops-producer`
  - `/actuator/health` devolviendo estado `UP`

## Estado final observado
- `card-ops-producer`: arranca correctamente y saludable.
- `zookeeper`: saludable.
- `/actuator/health`: `UP`.
- `otel-collector`: inicia y queda operando; persisten warnings intermitentes de resolucion hacia `tempo` segun orden/estado de red, mitigado parcialmente con dependencia declarada.

## Comandos utiles (referencia rapida)
```powershell
docker compose build --no-cache card-ops-producer
docker compose up -d --no-deps --force-recreate card-ops-producer
docker compose up -d --force-recreate zookeeper

docker compose logs --tail 120 card-ops-producer
docker compose logs --tail 120 zookeeper
docker compose logs --tail 120 otel-collector

docker inspect --format "{{.Name}} health={{.State.Health.Status}}" card-ops-producer-zookeeper-1
docker inspect --format "{{.Name}} health={{.State.Health.Status}}" card-ops-producer-card-ops-producer-1

Invoke-RestMethod http://localhost:8085/actuator/health | ConvertTo-Json -Depth 10
```

## Nota
Este archivo resume la sesion tecnica y decisiones aplicadas; si quieres, se puede generar una version ejecutiva (1 pagina) o una version tipo changelog por commit.

