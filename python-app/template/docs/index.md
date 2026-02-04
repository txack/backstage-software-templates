# ${{ values.app_name }} (Flask)

Pequeña API en Flask que expone dos endpoints para ver información del pod y un chequeo de salud. Este documento cubre cómo ejecutarla localmente, en contenedor, y desplegarla en Kubernetes (manifiestos y Helm), además de describir los endpoints disponibles.

## Resumen

- Framework: Flask
- Puerto por defecto: 5000
- Endpoints:
  - `GET /api/v1/healthz` — estado de salud
  - `GET /api/v1/info` — hora, hostname y mensaje
- Código principal: ver [${{ values.app_name }}/src/app.py](../src/app.py)

## Endpoints

### `GET /api/v1/healthz`

Chequeo de salud simple.

Ejemplo:

```bash
curl -s http://localhost:5000/api/v1/healthz
```

Respuesta:

```json
{"status":"up"}
```

### `GET /api/v1/info`

Devuelve información del sistema y un mensaje.

Ejemplo:

```bash
curl -s http://localhost:5000/api/v1/info | jq
```

Respuesta típica:

```json
{
  "time": "10:11:12AM on January 30, 2026",
  "hostname": "${{ values.app_name }}-7f5c9c9b5b-2x7m8",
  "message": "You are doing well!! :)",
  "deployed": "on k8s"
}
```

## Ejecución local

Requisitos: Python 3.12+ y `pip`.

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python src/app.py
```

Prueba rápida:

```bash
curl http://localhost:5000/api/v1/healthz
curl http://localhost:5000/api/v1/info
```

## Contenedor (Docker/Podman)

La imagen se construye a partir de [${{ values.app_name }}/Dockerfile](../Dockerfile).

Construir y ejecutar:

```bash
# Docker
docker build -t ${{ values.app_name }}:local .
docker run --rm -p 5000:5000 ${{ values.app_name }}:local

# Podman
podman build -t ${{ values.app_name }}:local .
podman run --rm -p 5000:5000 ${{ values.app_name }}:local
```

## Despliegue en Kubernetes (manifiestos)

Manifiestos en [${{ values.app_name }}/k8s/](../k8s):

- Deployment: usa la imagen `clloris/${{ values.app_name }}:v2-1` y expone `containerPort: 5000`.
- Service: `ClusterIP` en el puerto `8080` apuntando a `targetPort: 5000`.
- Ingress: clase `nginx`, host `${{ values.app_name }}.test.com`.

Aplicar:

```bash
kubectl apply -f k8s/deploy.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/ingress.yaml
```

Probar (asumiendo Ingress con DNS/resolución adecuada):

```bash
curl http://${{ values.app_name }}.test.com/api/v1/healthz
curl http://${{ values.app_name }}.test.com/api/v1/info
```

## Despliegue con Helm

Chart en [${{ values.app_name }}/charts/${{ values.app_name }}](../charts/${{ values.app_name }}).

Valores relevantes (por defecto):

- `service.port`: 5000 (coincide con el puerto del contenedor)
- `ingress.enabled`: true, `ingress.className`: `nginx`, host `${{ values.app_name }}.test.com`
- Probes `liveness/readiness`: `GET /api/v1/healthz`

Instalación básica:

```bash
helm install ${{ values.app_name }} ./charts/${{ values.app_name }} \
  --set image.repository=clloris/${{ values.app_name }} \
  --set image.tag=ee70802
```

Actualizar valores (ejemplo para cambiar host):

```bash
helm upgrade ${{ values.app_name }} ./charts/${{ values.app_name }} \
  --set ingress.hosts[0].host=myapp.local
```

## Troubleshooting

- El Service publica `8080` pero el contenedor escucha en `5000`; verifica que tu Ingress apunte al Service en el puerto `8080`.
- Comprueba salud con `curl <host>/api/v1/healthz` y revisa logs del pod si falla.
- Si usas otro IngressClass, ajusta `ingress.className` en `values.yaml`.

## Estructura

- Código: [${{ values.app_name }}/src/app.py](../src/app.py)
- Requisitos: [${{ values.app_name }}/requirements.txt](../requirements.txt)
- Dockerfile: [${{ values.app_name }}/Dockerfile](../Dockerfile)
- Manifiestos: [${{ values.app_name }}/k8s/](../k8s)
- Helm Chart: [${{ values.app_name }}/charts/${{ values.app_name }}](../charts/${{ values.app_name }})


