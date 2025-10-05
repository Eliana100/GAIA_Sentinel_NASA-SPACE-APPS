# CUMARU - Sistema de Monitoramento de Riscos Urbanos

O CUMARU é um protótipo full-stack que demonstra um sistema completo de monitoramento e alerta de riscos urbanos (como enchentes e deslizamentos) utilizando dados de observação da Terra **com uma demonstração de integração com APIs reais da NASA**. Ele oferece um frontend interativo com mapa, um backend robusto com API e um motor de alertas, tudo orquestrado via Docker.

## Visão Geral da Arquitetura

O sistema é dividido em camadas:

1.  **Data Sources (NASA Integration / Simulada)**: O backend inclui uma camada `nasa_integrator` que demonstra como buscar dados de APIs da NASA (GPM, MODIS, SMAP, SRTM). Para o protótipo, o processamento desses dados brutos para `tile_features` é simulado via `data_simulator.py`.
2.  **Ingest Layer (Demonstrativa)**: A função `fetch_nasa_data_real` no backend simula a chamada e o consumo de dados das APIs da NASA.
3.  **Processing Layer (Alert Engine)**: O motor de alertas no backend processa os `tile_features` (sejam eles simulados ou derivados de dados NASA), aplica regras heurísticas e gera alertas.
4.  **Feature Store / DB**: SQLite (`data/cumaru.db`) para persistir tiles, feições e alertas.
5.  **API Layer**: Backend FastAPI que expõe endpoints para o frontend e outros serviços.
6.  **Dashboard (Frontend)**: Aplicação React com Mapbox GL JS para visualização de mapa e alertas.
7.  **Notification Layer (Simulada)**: Endpoints de webhook para municípios.

## Tecnologias Utilizadas

*   **Frontend**: React (Vite), Mapbox GL JS, Tailwind CSS
*   **Backend**: Python (FastAPI), SQLite, Pydantic, SQLAlchemy, `httpx` (para chamadas externas)
*   **Orquestração**: Docker, Docker Compose
*   **Design**: Figma (mockups e JSON de estrutura)

## Requisitos

*   Docker e Docker Compose
*   Node.js e npm (para desenvolvimento frontend fora do Docker, opcional)
*   Python e pip (para desenvolvimento backend fora do Docker, opcional)

## Execução Local (com Docker)

1.  **Clone o Repositório:**
    ```bash
    git clone https://github.com/seu-usuario/cumaru.git
    cd cumaru
    ```

2.  **Configurar Variáveis de Ambiente:**
    Crie os arquivos `.env` em `frontend/` e `backend/` com base nos exemplos fornecidos:

    `frontend/.env` (ou `frontend/.env.local` para Vite):
    ```
    VITE_MAPBOX_TOKEN=SEU_MAPBOX_ACCESS_TOKEN  # Gere um token em mapbox.com
    VITE_BACKEND_URL=http://localhost:8000
    ```
    `backend/.env`:
    ```
    DATABASE_URL=sqlite:///./data/cumaru.db
    NASA_EARTHDATA_USERNAME=SEU_USUARIO_EARTHDATA # Crie um em urs.earthdata.nasa.gov
    NASA_EARTHDATA_PASSWORD=SUA_SENHA_EARTHDATA # Necessário para acessar algumas APIs da NASA
    TWILIO_ACCOUNT_SID=ACxxxxxxxxxxxx # Placeholder
    TWILIO_AUTH_TOKEN=your_auth_token # Placeholder
    TWILIO_PHONE_NUMBER=+15017122661 # Placeholder
    ```

    **Importante**:
    *   Substitua `SEU_MAPBOX_ACCESS_TOKEN` pelo seu token real do Mapbox.
    *   Para a integração real da NASA funcionar, você precisará de uma conta Earthdata Login (`NASA_EARTHDATA_USERNAME` e `NASA_EARTHDATA_PASSWORD`). O backend já está configurado para incluir esses headers de autenticação na chamada simulada.

3.  **Construir e Iniciar os Contêineres Docker:**
    ```bash
    docker-compose up --build -d
    ```
    Isso irá construir as imagens do frontend e backend, criar o banco de dados SQLite e iniciar os serviços.

4.  **Inicializar Dados (Opcional, mas Recomendado):**
    Para popular o banco de dados com dados simulados e gerar alertas iniciais, execute os scripts dentro do contêiner do backend:
    ```bash
    docker-compose exec backend python -c "from backend.database import init_db; init_db()"
    docker-compose exec backend python backend/data_simulator.py
    docker-compose exec backend python backend/alert_engine.py --run-once
    ```
    O `alert_engine.py` pode ser configurado como um cron job no ambiente de produção, mas para o protótipo, executamos manualmente para popular.

5.  **Acessar a Aplicação:**
    Abra seu navegador e vá para `http://localhost:5173` (ou a porta que o Vite usar, verificável nos logs do Docker).

## Endpoints da API (Backend FastAPI)

A API do backend está disponível em `http://localhost:8000`.

*   **GET /api/tiles?bbox=[min_lon,min_lat,max_lon,max_lat]&zoom=[zoom_level]**
    *   Retorna uma lista de tiles com `risk_score` e metadados dentro da bounding box e nível de zoom especificados.
    *   Exemplo de payload de retorno:
        ```json
        [
          {
            "tile_id": "h3_8a12b...",
            "lat": -23.55,
            "lon": -46.63,
            "risk_score": 75,
            "type": "flood",
            "timestamp": "2023-10-27T10:00:00Z",
            "popup_info": "Risco Alto: Enchente na região central. Chuva 24h: 60mm."
          }
        ]
        ```

*   **GET /api/alerts/active**
    *   Retorna uma lista de alertas ativos no sistema.
    *   Exemplo de payload de retorno (detalhes no `alert_payload_examples.json`):
        ```json
        [
          {
            "alert_id": "A20251005-0001",
            "tile_id": "h3_8a12b...",
            "type": "flood",
            "score": 82,
            "confidence": 0.72,
            "status": "active",
            "created_at": "2025-10-05T14:30:00Z",
            "location_info": {
              "city": "São Paulo",
              "tile_description": "Zona Norte"
            },
            "recommendations": ["Evacuar áreas ribeirinhas", "Evitar subsolos"],
            "evidence": {
              "rain_24h": 62,
              "soil_moisture": 86,
              "ndvi": 0.12,
              "chart_data": {
                "rain_48h": [
                  { "hour": "-48h", "value": 10 },
                  { "hour": "-24h", "value": 20 },
                  { "hour": "agora", "value": 62 }
                ]
              }
            }
          }
        ]
        ```

*   **POST /api/ingest/nasa-data?start=...&end=...&product=gpm**
    *   **Demonstração de Integração Real com NASA**: Este endpoint simula a chamada a uma API real da NASA para buscar dados de observação da Terra.
    *   **Autenticação**: Usa variáveis de ambiente `NASA_EARTHDATA_USERNAME` e `NASA_EARTHDATA_PASSWORD` para simular a autenticação.
    *   **Parâmetros**: `start` (ISO format date-time), `end` (ISO format date-time), `product` (gpm, modis, smap, srtm - para simular diferentes endpoints).
    *   Para o protótipo, a resposta ainda é um dado simulado, mas a estrutura da requisição para a NASA é realística.
    *   Exemplo de curl:
        ```bash
        curl -X POST "http://localhost:8000/api/ingest/nasa-data?start=2023-10-26T00:00:00Z&end=2023-10-27T00:00:00Z&product=gpm" -H "Authorization: Basic $(echo -n 'SEU_USUARIO_EARTHDATA:SUA_SENHA_EARTHDATA' | base64)"
        ```
    *   Retorna: `{"message": "Dados da NASA (simulados para o protótipo) processados para: gpm", "data": {...}}`

*   **POST /api/ingest/crowd-report**
    *   Aceita relatórios de usuários (simulados).
    *   Método: `POST`
    *   Payload JSON:
        ```json
        {
          "user_id": "user123",
          "lat": -23.5505,
          "lon": -46.6333,
          "type": "flood",
          "photo_url": "http://example.com/photo.jpg",
          "description": "Rua alagada perto da estação."
        }
        ```
    *   Retorna: `{"message": "Relatório de campo recebido.", "report_id": "..."}`

*   **POST /api/webhook/municipality**
    *   Endpoint para receber webhooks de resposta/confirmação de municípios.
    *   Método: `POST`
    *   Payload JSON (exemplo):
        ```json
        {
          "alert_id": "A20251005-0001",
          "status": "acknowledged",
          "message": "Defesa Civil de São Paulo ciente e em ação."
        }
        ```
    *   Retorna: `{"message": "Webhook municipal recebido e processado."}`

## Simulação de Dados

O script `scripts/generate_synthetic_data.py` gera séries temporais sintéticas de chuva, umidade do solo e NDVI para tiles em cidades exemplo (São Paulo, Dhaka, Medellín). Estes dados são usados pelo `data_simulator.py` no backend para popular o banco de dados com `tile_features`, que o `alert_engine` processará. Esta etapa é crucial para o funcionamento do protótipo, **mesmo com a integração demonstrativa da NASA**, pois a transformação de dados brutos da NASA para o formato `tile_features` é complexa e está fora do escopo deste protótipo inicial.

Para gerar os dados sintéticos:
```bash
python scripts/generate_synthetic_data.py
