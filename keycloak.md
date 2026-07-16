### Reference
- https://deepwiki.com/backstage/community-plugins/3.2-keycloak-integration
- https://janus-idp.io/blog/2023/01/17/enabling-keycloak-authentication-in-backstage/
- https://howtodoinjava.com/devops/keycloak-on-docker/



### MISC
docker run -d \
  --name backstage-postgres \
  --network backstage-network \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres123 \
  -e POSTGRES_DB=keycloak \
  -v backstage-postgres-data:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:16
  
  
docker exec backstage-postgres pg_isready -U postgres

docker rm -f keycloak
docker run -d \
  --name keycloak \
  --network backstage-network \
  -e KC_DB=postgres \
  -e KC_DB_URL=jdbc:postgresql://backstage-postgres:5432/keycloak \
  -e KC_DB_USERNAME=postgres \
  -e KC_DB_PASSWORD=postgres123 \
  -e KC_BOOTSTRAP_ADMIN_USERNAME=admin \
  -e KC_BOOTSTRAP_ADMIN_PASSWORD=admin123 \
  -e KC_HOSTNAME=localhost \
  -e KC_HOSTNAME_PATH=/keycloak \
  -e KC_HTTP_ENABLED=true \
  -e KC_HTTP_RELATIVE_PATH=/keycloak \
  -e KC_PROXY_HEADERS=xforwarded \
  -p 8080:8080 \
  quay.io/keycloak/keycloak:latest \
  start-dev
