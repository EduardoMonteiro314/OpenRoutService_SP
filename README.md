# Configuração do OpenRouteService (ORS) para Mapa Local

Olá! Criei este repositório principalmente como um "diário de bordo" e um guia de referência para os meus projetos futuros.

O processo de configurar o OpenRouteService (ORS) com um mapa personalizado teve várias etapas e alguns desafios inesperados. Decidi documentar tudo aqui para ajudar qualquer outra pessoa que possa passar pelo mesmo, e também para facilitar os meus próprios projetos futuros.

Espero que seja útil!

### Contexto do Projeto

**Nota Importante:** Os ficheiros de configuração e o mapa de exemplo (`seu_mapa.osm.pbf`) neste repositório foram utilizados para um teste com uma **área específica e limitada da cidade de São Paulo**. Se você utilizar um mapa maior (como um estado ou país), lembre-se de ajustar a alocação de memória (`XMX`) no ficheiro `docker-compose.yml` para um valor consideravelmente maior para evitar erros.

---

## Pré-requisitos

-   [Docker](https://docs.docker.com/get-docker/) instalado.
-   [Docker Compose](https://docs.docker.com/compose/install/) instalado.
-   Permissão para executar comandos do Docker (ver secção de Troubleshooting).

## Estrutura de Ficheiros

Para que o projeto funcione, a estrutura de pastas e ficheiros deve ser a seguinte:

```
/seu_projeto/
├── docker-compose.yml
├── ors-config.yml
└── ors-data/
    └── seu_mapa.osm.pbf
```

## Ficheiros de Configuração

Estes são os conteúdos exatos e funcionais para a versão `v8.0.0` do ORS.

### `docker-compose.yml`

Este ficheiro orquestra o serviço Docker. Ele define os volumes (pastas partilhadas), as portas de rede e as variáveis de ambiente, como a alocação de memória.

```yaml
services:
  ors-app:
    container_name: ors-app
    image: openrouteservice/openrouteservice:v8.0.0
    ports:
      - "8080:8082"
    volumes:
      # Mapeia a pasta do mapa para o novo caminho de "files"
      - ./ors-data:/ors-core/data/pbf:z
      # Mapeia o ficheiro de configuração para o novo caminho de "config"
      - ./ors-config.yml:/home/ors/config/ors-config.yml:z
      # Mapeia os dados persistentes (grafos, cache, logs)
      - ./graphs:/home/ors/graphs:z
      - ./elevation_cache:/home/ors/elevation_cache:z
      - ./logs/ors:/var/log/ors:z
    environment:
      # MUITO IMPORTANTE: "True" na 1ª vez, "False" nas seguintes
      REBUILD_GRAPHS: "True"
      # Alocação de memória. Ajustar conforme o tamanho do mapa.
      XMS: "1g"
      XMX: "1g"
```

### `ors-config.yml`

Este ficheiro diz ao ORS qual mapa usar e quais perfis de rota ativar.

```yaml
ors:
  engine:
    # Aponta para o ficheiro de mapa DENTRO do contentor
    source_file: /ors-core/data/pbf/sudeste-latest.osm.pbf 
    profiles:
      car:
        enabled: true
        profile: driving-car
        # Desativado para evitar problemas de download de dados de elevação
        elevation: false
```
**Nota:** Lembre-se de substituir `sudeste-latest.osm.pbf` pelo nome exato do seu ficheiro de mapa.

## Guia de Execução

Siga estes passos para iniciar o serviço.

### 1. Primeira Execução (Construção do Mapa)

Na primeira vez, o ORS precisa de processar o seu ficheiro `.pbf` e construir os grafos de rota.

1.  Garanta que o `docker-compose.yml` tem `REBUILD_GRAPHS: "True"`.
2.  No terminal, na pasta do projeto, execute:
    ```bash
    sudo docker compose up
    ```
3.  **Aguarde.** Este processo pode ser demorado (de minutos a horas), dependendo do tamanho do seu mapa e da potência do seu computador.
4.  Quando os logs mostrarem uma mensagem como `Server startup in ... milliseconds`, o serviço está pronto.

### 2. Execuções Seguintes (Uso Normal)

Após a primeira execução bem-sucedida, faça o seguinte para que o serviço inicie em segundos:

1.  Pare o serviço com `sudo docker compose down`.
2.  Edite o `docker-compose.yml` e altere para `REBUILD_GRAPHS: "False"`.
3.  Salve o ficheiro.
4.  A partir de agora, para iniciar o serviço, basta usar `sudo docker compose up`.

## Como Testar

Após o serviço iniciar, pode usar as seguintes URLs no seu navegador para testar:

* **Verificação de Saúde (Health Check):**
    ```
    http://localhost:8080/ors/v2/health
    ```
    *(Resultado esperado: `{"status":"ready"}`)*

* **Teste de Rota (Exemplo para São Paulo):**
    ```
    http://localhost:8080/ors/v2/directions/driving-car?start=LONGITUDE_INICIAL,LATITUDE_INICIAL&end=LONGITUDE_FINAL,LATITUDE_FINAL
    ```
    *(Substitua pelas coordenadas dentro da área do seu mapa. Ex: `start=-46.6388,-23.5451&end=-46.6680,-23.5510`)*

## Guia de Troubleshooting (Resumo da Jornada)

Durante esta configuração, encontrei vários problemas. Aqui estão eles e as suas soluções, para referência futura:

1.  **Erro: `permission denied ... docker.sock`**
    * **Causa:** O utilizador não tem permissão para comunicar com o serviço Docker.
    * **Solução:** Adicionar o utilizador ao grupo `docker` com `sudo usermod -aG docker ${USER}` e reiniciar a sessão, ou usar `sudo` antes de todos os comandos `docker`.

2.  **Problema: Serviço carrega o mapa de Heidelberg em vez do mapa personalizado.**
    * **Causa:** A versão 8.0.0 do ORS mudou a forma de ler as configurações. Ela espera um ficheiro `.yml` em caminhos específicos (`/home/ors/...`) e ignora configurações `.json` ou ficheiros em caminhos antigos (`/ors-core/...`).
    * **Solução:** Usar os ficheiros `docker-compose.yml` e `ors-config.yml` descritos acima, que alinham todos os volumes e caminhos com a nova estrutura esperada pela imagem Docker.

3.  **Erro: `Unable to get an appropriate route profile ... driving-car` (Código 2099)**
    * **Causa:** O perfil de rota não foi explicitamente definido e ativado no ficheiro `ors-config.yml`.
    * **Solução:** Garantir que a secção `profiles` no `ors-config.yml` contém as chaves `enabled: true` e `profile: driving-car`.

4.  **Erro: `Can't decode srtm_...tif`**
    * **Causa:** Falha no download ou ficheiro corrompido dos dados de elevação (relevo).
    * **Solução:** Desativar o uso de dados de elevação no `ors-config.yml` adicionando a linha `elevation: false` no perfil desejado.
