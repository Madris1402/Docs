---
tags:
  - Docker
---
```powershell
docker network create --driver bridge mule-net-test
```

```powershell
docker network connect <network-name> <container-name-or-id>
```

```powershell
docker build -t mule-standalone:v1 .
```

```powershell
docker run -d --name mulesoft -p 8081:8081 -v "${PWD}/apps-batch:/opt/mule/apps" --network mule-net-test --add-host=host.docker.internal:host-gateway mule-standalone:v1
```
