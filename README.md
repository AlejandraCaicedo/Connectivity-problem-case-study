# Connectivity-problem-case-study

## Caso de Estudio: Problemas de Conectividad entre Servicios en Kubernetes

## Escenario

Tienes un microservicio de autenticación que debe comunicarse con una base de datos PostgreSQL. Ambos están en el mismo cluster de Kubernetes. **El problema: los labels del Pod y el selector del Service no coinciden**, causando que no haya conectividad.

---

## PARTE 1: SETUP INICIAL

### Paso 1.1: Crear el namespace

```bash
kubectl create namespace microservices-demo
kubectl config set-context --current --namespace=microservices-demo
```

**Resultado esperado:**

```
namespace/microservices-demo created
```

---

## PARTE 2: CREAR EL PROBLEMA

### Paso 2.1: Desplegar los recursos para reproducir el problema

```bash
kubectl apply -f database-pod.yaml
kubectl apply -f database-service.yaml
kubectl apply -f auth-service.yaml
```

**Resultado esperado:**

```
pod/postgres-db created
service/postgres-service created
pod/auth-service created
```

### Paso 2.2: Esperar a que los pods se levanten

```bash
kubectl wait --for=condition=Ready pod/postgres-db -n microservices-demo --timeout=60s
kubectl wait --for=condition=Ready pod/auth-service -n microservices-demo --timeout=60s
```

**Resultado esperado:**

```
pod/postgres-db condition met
pod/auth-service condition met
```

---

## PARTE 3: MEDICIONES DEL PROBLEMA

### Medición 3.1: Estado de los Pods

```bash
kubectl get pods -n microservices-demo -o wide
```

**Resultado esperado:** Ambos pods en estado `Running`

```
NAME           READY   STATUS    RESTARTS   AGE   IP
postgres-db    1/1     Running   0          2m    10.1.x.x
auth-service   1/1     Running   0          1m    10.1.x.x
```

### Medición 3.2: Verificar los labels del Pod

```bash
kubectl get pod postgres-db -n microservices-demo --show-labels
```

**Resultado esperado:** El label es `app=database-broken`

```
NAME          READY   STATUS    RESTARTS   AGE   LABELS
postgres-db   1/1     Running   0          2m    app=database-broken
```

### Medición 3.3: Verificar el selector del Service

```bash
kubectl describe svc postgres-service -n microservices-demo
```

**Resultado esperado:** El selector busca `app=database` (diferente del label del pod)

Busca esta línea:

```
Selector:          app=database
```

### Medición 3.4: **CRÍTICA - Verificar Endpoints (DEBE ESTAR VACIO)**

```bash
kubectl get endpoints postgres-service -n microservices-demo
```

**Resultado esperado:** Endpoints vacío o sin IPs (ESTE ES EL PROBLEMA)

```
NAME               ENDPOINTS   AGE
postgres-service   <none>      2m
```

### Medición 3.5: Ver logs del auth-service (mostrando el error)

```bash
kubectl logs auth-service -n microservices-demo
```

**Resultado esperado:** Verás intentos fallidos de conexión

```
============================================================
SERVICIO DE AUTENTICACION - Iniciando...
============================================================

[Intento 1/5] Conectando a postgres-service:5432...
✗ Error: OperationalError: could not translate host name "postgres-service" to address: Unknown host
  Reintentando en 3 segundos...
```

### Medición 3.6: Métricas de los Pods (ANTES de solucionar)

```bash
kubectl top pods -n microservices-demo
```

**Resultado esperado:** Verás el consumo de CPU y memoria

```
NAME           CPU(m)   MEMORY(Mi)
postgres-db    50m      80Mi
auth-service   200m     100Mi
```

## PARTE 4: DIAGNOSTICO (Resumen del Problema)

**PROBLEMA IDENTIFICADO:**

| Componente            | Valor                  |
| --------------------- | ---------------------- |
| Label del Pod         | `app: database-broken` |
| Selector del Service  | `app: database`        |
| Endpoints del Service | ∅ (VACÍO)              |
| Estado de conexión    | ✗ FALLO                |
| Causa raíz            | Labels no coinciden    |

---

## PARTE 5: SOLUCIÓN

### Paso 5.1: Crear archivo `solucion.yaml`

Crea un archivo llamado `solucion.yaml` con este contenido:

```yaml
# Base de datos con label CORRECTO
apiVersion: v1
kind: Pod
metadata:
  name: postgres-db
  namespace: microservices-demo
  labels:
    app: database
spec:
  containers:
    - name: postgres
      image: postgres:15-alpine
      env:
        - name: POSTGRES_DB
          value: authdb
        - name: POSTGRES_USER
          value: admin
        - name: POSTGRES_PASSWORD
          value: password123
      ports:
        - containerPort: 5432
      resources:
        requests:
          memory: "256Mi"
        limits:
          memory: "512Mi"
      livenessProbe:
        exec:
          command:
            - /bin/sh
            - -c
            - pg_isready -U admin -d authdb
        initialDelaySeconds: 10
        periodSeconds: 10
```

### Paso 5.2: Eliminar los recursos con problema

```bash
kubectl delete pod postgres-db auth-service -n microservices-demo
```

**Resultado esperado:**

```
pod "postgres-db" deleted
pod "auth-service" deleted
```

### Paso 5.3: Esperar a que se eliminen

```bash
Start-Sleep -Seconds 10
```

O simplemente espera 10 segundos.

### Paso 5.4: Aplicar la solución y desplegar auth-service de nuevo

```bash
kubectl apply -f solucion.yaml
kubectl apply -f auth-service-pod.yaml
```

**Resultado esperado:**

```
pod/postgres-db created
pod/auth-service created
```

### Paso 5.5: Esperar a que se levanten

```bash
kubectl wait --for=condition=Ready pod/postgres-db -n microservices-demo --timeout=60s
kubectl wait --for=condition=Ready pod/auth-service -n microservices-demo --timeout=60s
```

---

## PARTE 6: MEDICIONES DESPUÉS DE LA SOLUCIÓN

### Medición 6.1: Estado de los Pods

```bash
kubectl get pods -n microservices-demo -o wide
```

**Resultado esperado:** Ambos en `Running`

### Medición 6.2: Verificar el label correcto

```bash
kubectl get pod postgres-db -n microservices-demo --show-labels
```

**Resultado esperado:** El label es ahora `app=database`

```
NAME          READY   STATUS    RESTARTS   AGE   LABELS
postgres-db   1/1     Running   0          1m    app=database
```

### Medición 6.3: Verificar el selector del Service

```bash
kubectl describe svc postgres-service -n microservices-demo
```

**Resultado esperado:** Ahora hay coincidencia

```
Selector:          app=database
```

### Medición 6.4: **CRÍTICA - Verificar Endpoints (DEBE TENER UNA IP)**

```bash
kubectl get endpoints postgres-service -n microservices-demo
```

**Resultado esperado:** Debe mostrar una IP (el pod de la BD)

```
NAME               ENDPOINTS       AGE
postgres-service   10.1.x.x:5432   1m
```

### Medición 6.5: Ver logs del auth-service (mostrando éxito)

```bash
kubectl logs auth-service -n microservices-demo
```

**Resultado esperado:** Conexión exitosa

```
============================================================
SERVICIO DE AUTENTICACION (FIXED) - Iniciando...
============================================================

[Intento 1/5] Conectando a postgres-service:5432...
✓ ¡Conexión EXITOSA!
✓ Query ejecutada: (datetime.datetime(2025, 10, 16, 14, 32, 45, 123456, tzinfo=psycopg2.tz.FixedOffsetTimezone(offset=0, name=None)),)
✓ PostgreSQL: PostgreSQL 15.5 on x86_64-pc-linux-gnu, compiled by gcc...

============================================================
STATUS: OPERATIVO ✓✓✓
============================================================
```

### Medición 6.6: Métricas de los Pods (DESPUÉS de solucionar)

```bash
kubectl top pods -n microservices-demo
```

**Resultado esperado:** El auth-service usa menos CPU (no está en intentos de reconexión)

```
NAME           CPU(m)   MEMORY(Mi)
postgres-db    50m      85Mi
auth-service   10m      50Mi
```

Nota: El CPU del auth-service disminuyó porque ya no está intentando conectar constantemente.

### Medición 6.7: Ver eventos actuales

```bash
kubectl get events -n microservices-demo --sort-by='.lastTimestamp'
```

---

## PARTE 7: COMPARATIVA ANTES vs DESPUÉS

| Métrica               | ANTES                | DESPUÉS            |
| --------------------- | -------------------- | ------------------ |
| **Labels coinciden**  | ✗ No                 | ✓ Sí               |
| **Service Endpoints** | 0 (vacío)            | 1 (10.1.x.x:5432)  |
| **Logs auth-service** | ✗ Connection refused | ✓ Exitoso          |
| **CPU auth-service**  | Alto (intentos)      | Bajo (funcionando) |
| **Conectividad**      | ✗ FALLO              | ✓ ÉXITO            |

---

## PARTE 7.5: CAUSAS COMUNES DE PROBLEMAS DE CONECTIVIDAD

| Causa                        | Síntoma                         | Diagnóstico                                 | Solución                                         |
| ---------------------------- | ------------------------------- | ------------------------------------------- | ------------------------------------------------ |
| **Labels no coinciden**      | Service con 0 Endpoints         | `kubectl get endpoints` vacío               | Alinear labels pod ↔ Service selector            |
| **Pod no está Running**      | Pod en CrashLoopBackOff         | `kubectl logs` con errores                  | Revisar logs del pod, revisar imagen             |
| **Puerto incorrecto**        | Connection refused              | `kubectl describe pod` revisa containerPort | Verificar containerPort vs port en Service       |
| **Network Policy bloqueada** | DNS ok, conexión rechazada      | `kubectl get networkpolicies`               | Crear/revisar NetworkPolicy                      |
| **DNS no resuelve**          | "could not translate host name" | `kubectl run busybox -- nslookup`           | Verificar CoreDNS, revisar DNS del cliente       |
| **Namespace diferente**      | Service no encontrado           | `kubectl get svc -A`                        | Usar FQDN: `service.namespace.svc.cluster.local` |
| **Service Type incorrecto**  | No se puede acceder desde fuera | `kubectl get svc` revisar Type              | Cambiar de ClusterIP a NodePort/LoadBalancer     |
| **Pod IP ephemeral**         | Conexiones se pierden           | `kubectl get pod -o wide`                   | Usar Service en lugar de IP directa              |

---

## PARTE 8: LIMPIEZA

### Paso 8.1: Eliminar todos los recursos

```bash
kubectl delete namespace microservices-demo
```

**Resultado esperado:**

```
namespace "microservices-demo" deleted
```

### Paso 8.2: Verificar que está eliminado

```bash
kubectl get namespace | grep microservices-demo
```

**Resultado esperado:** No debe mostrar nada

---

## Resumen del Caso de Estudio

**Problema:** Labels desalineados entre Pod y Service selector

- Pod: `app: database-broken`
- Service selector: `app: database`
- Resultado: 0 Endpoints, sin conectividad

**Solución:** Alinear los labels

- Pod: `app: database` ✓
- Service selector: `app: database` ✓
- Resultado: 1 Endpoint, conectividad exitosa

**Lecciones Aprendidas:**

1. El Service usa el selector para encontrar Pods
2. Los labels del Pod DEBEN coincidir con el selector del Service
3. Si no hay Endpoints, hay desalineación de labels
4. Verificar `kubectl get endpoints` es clave para diagnosticar
