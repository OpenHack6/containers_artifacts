docker build -f .\dockerfiles\Dockerfile_0 -t user-java:v1 ./src/user-java
docker build -f .\dockerfiles\Dockerfile_1 -t tripviewer:v1 ./src/tripviewer
docker build -f .\dockerfiles\Dockerfile_2 -t userprofile:v1 ./src/userprofile
docker build -f .\dockerfiles\Dockerfile_3 -t poi:v1 ./src/poi
docker build -f .\dockerfiles\Dockerfile_4 -t trips:v1 ./src/trips

docker network create oh
docker pull mcr.microsoft.com/mssql/server:2017-latest
docker run --network oh -e "ACCEPT_EULA=Y" -e 'SA_PASSWORD=stronkpass123!@#' -p 1433:1433 --name sql1 --hostname sql1 -d mcr.microsoft.com/mssql/server:2017-latest
docker exec -t sql1 /opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P 'stronkpass123!@#' -Q "CREATE DATABASE mydrivingDB"
docker exec -t sql1 /opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P 'stronkpass123!@#' -Q "SELECT Name from sys.Databases"

az acr list
az acr repository list --name registryvhk8886
az acr repository show-tags --name registryvhk8886 --repository dataload

az login
az acr login --name registryvhk8886
docker run --network oh -e SQLFQDN=sql1 -e SQLUSER=SA -e SQLPASS='stronkpass123!@#' -e SQLDB=mydrivingDB registryvhk8886.azurecr.io/dataload:1.0

# filling the table
docker run --network oh -e SQLFQDN=sql1 -e SQLUSER=SA -e SQLPASS='stronkpass123!@#' -e SQLDB=mydrivingDB registryvhk8886.azurecr.io/dataload:1.0
docker exec -t sql1 /opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P 'stronkpass123!@#' -Q "SELECT * from mydrivingDB.INFORMATION_SCHEMA.TABLES"
docker exec -t sql1 /opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P 'stronkpass123!@#' -Q "SELECT * from mydrivingDB.dbo.POIs"

# poi
# docker start --network oh -e user-java:v1
docker run -d -p 8080:80 --name poi --network oh --env SQL_USER=SA --env SQL_PASSWORD='stronkpass123!@#' --env SQL_SERVER=sql1 --env SQL_DBNAME="mydrivingDB" poi:v1
# docker exec -it poi /bin/bash
curl -i -X GET 'http://localhost:8080/api/poi/healthcheck'

# tripviewer
# docker run -d -p 8081:80 --name tripviewer -e "USERPROFILE_API_ENDPOINT=http://$ENDPOINT" -e "TRIPS_API_ENDPOINT=http://$ENDPOINT" tripinsights/tripviewer:1.0
docker run --network oh -d -p 8081:80 --name tripviewer tripviewer:v1
curl -i -X GET 'http://localhost:8081'

# user-javadocker run --network oh -d -p 8083:80 --name userprofile -e SQL_PASSWORD='stronkpass123!@#' -e "SQL_SERVER=sql1" -e "SQL_USER=SA" userprofile:v1
# docker run -d -p 8082:80 --name user-java -e "SQL_PASSWORD=$SQL_PASSWORD" -e "SQL_SERVER=$SQL_SERVER" tripinsights/user-java:1.0
docker run --network oh -d -p 8082:80 --name user-java -e SQL_PASSWORD='stronkpass123!@#' -e "SQL_SERVER=sql1" user-java:v1
curl -i -X GET 'http://localhost:8082/api/user-java/healthcheck'

# userprofile
# docker run -d -p 8083:80 --name userprofile -e "SQL_PASSWORD=$SQL_PASSWORD" -e "SQL_SERVER=$SQL_SERVER" tripinsights/userprofile:1.0
docker run --network oh -d -p 8083:80 --name userprofile -e SQL_PASSWORD='stronkpass123!@#' -e "SQL_SERVER=sql1" -e "SQL_USER=SA" userprofile:v1
curl -i -X GET 'http://localhost:8083/api/user'

# trips
# docker run -d -p 8084:80 --name trips -e "SQL_PASSWORD=$SQL_PASSWORD" -e "SQL_SERVER=$SQL_SERVER" -e "OPENAPI_DOCS_URI=http://$EXTERNAL_IP" tripinsights/trips:1.0
docker run --network oh -d -p 8084:80 --name trips -e SQL_PASSWORD='stronkpass123!@#' -e "SQL_SERVER=sql1" -e "SQL_USER=SA" trips:v1
curl -i -X GET 'http://localhost:8084/api/trips'

